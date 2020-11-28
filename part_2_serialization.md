# ABUSING CONTEXT IN GO: SERIALIZATION

---
<!-- fg=black bg=red -->
## Probably don't do this at home
What's about to be discussed has a few key assumptions/limitations that severely limit its practical use

---

## Theory

```
At the extreme end of microservice architecture, network communication becomes the substitution for function calls.
```
This presents the problem of data serialization
<!-- 
This change forces many compromises in Go, 
mostly because everything sent across the network must be able to be serialized and deserialized. 
Luckily, the Go standard library includes an encoding package, which handles most serialization use cases.
 -->

---
## Practice
Remember, `context.Context` is an interface that has 4 implementations in the standard library
- `deadline` contains a parent context, time, and cancel func
	- (`timeout` contexts are a wrapper over deadline)
- `cancel` contains a parent context and cancel func
- `value` contains a parent context, key interface, and value interface
- `empty`

---
## Serialization Woes

- functions can't be serialized
- the value context's key and value are an interface, making round trip serialization nontrivial
	- an interface may be a function, making it impossible to serialize
	- it is very commonly an unexported struct, meaning the receiver wouldn't have the type data to deserialize

---
## Serialization Solutions

- Don't do any of those things. Just ignore them.
- Unexported primitives (e.g. "myContextKeyString") work fine: the type data exists on both sides

---
## Implementation
 1. Receive an incoming context: use reflection to determine the concrete type
 1. If the type is `cancelCtx`, remember that we’ll have to create a cancellation function during deserialization
 1. If the context is `timerCtx`, reach in and grab the deadline time.
 	1. when creating a `timerCtx`, the earliest deadline is always chosen. No need to compare deadlines
 1. Remember the deadline for deserialization
 1. If the type is `valueCtx`:
 	1. reach in and grab the key/value `interface`s. Use `gob` to register them, and add them to `map[interface]interface`
 	1. do not overwrite any existing keys
 1. If the context has a parent context, use it to recurse upwards in the context stack
 1. Create our own struct to house the deadline time, cancellation func, and value map. This is what actually gets serialized.
 1. Serialize this struct across the wire using `gob`, and receive it on the other side
 1. At deserialization, we know we need to create a `timerCtx` and cancellation function, so we create them

<!-- 
 - gob can serialize things it has the concrete implementations for (e.g. unexported keys with a primitive type)
  -->

---
## Notes

 - Time is approximate at best between any two computers
 - Context is flattened during serialization, nothing pre-serialization can be popped off post-serialization

---
```go
func buildMap(ctx context.Context, s contextData) contextData {
    rs := reflect.ValueOf(ctx).Elem()
    if rs.Type() == reflect.ValueOf(context.Background()).Elem().Type() {
        return s  // base case: if the current context is an emptyCtx, we're done.
    }

    rf := rs.FieldByName("key")
    if rf.IsValid() { // if there's a key, it's a valueCtx
        
        rf = reflect.NewAt(rf.Type(), unsafe.Pointer(rf.UnsafeAddr())).Elem()
        if rf.CanInterface() {
            key := rf.Interface()

            rv := rs.FieldByName("val")
            rv = reflect.NewAt(rv.Type(), unsafe.Pointer(rv.UnsafeAddr())).Elem()
            if rv.CanInterface() {
                val := rv.Interface()

                if _, exists := s.Values[key]; !exists {
                    s.Values[key] = val
                    gob.Register(key) // register them for serialization
                    gob.Register(val)
                }
            }
        }
    } else {
         // it's either a cancelCtx or timerCtx, implementation omitted for
    }

    parent := rs.FieldByName("Context")
    if parent.IsValid() && !parent.IsNil() {
        // if there's a parent context, recurse
        return buildMap(parent.Interface().(context.Context), s)
    }
    return s
}
```
<!-- 
 - lots of reflection and unsafe boilerplate
 - the gist is to reach in, grab the key/val, ensure we're not shadowing something, and register/save them
 - recurse upwards -->