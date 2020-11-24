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
<!-- TODO: inline code doesn't work in a list -->
Remember, `context.Context` is an interface that has 4 implementations in the standard library
- deadline (timeout contexts are a wrapper over deadline)
	- contains a parent context, time, and cancel func
- cancel
	- contains a parent context and cancel func
- value
	- contains a parent context, key interface, and value interface
- empty
	- ...

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
<!-- TODO: numbered lists don't work -->
<!-- TODO: 
	- the following statement is misleading and unnecessary:
	If we already have a deadline (remember, context is composed), choose the earliest of the existing and current -->
 1. Receive an incoming context: use reflection to determine the concrete type
 2. If the type is cancelCtx, remember that we’ll have to create a cancellation function during deserialization
 3. If the context is timerCtx, reach in and grab the deadline time.
 4. If we already have a deadline (remember, context is composed), choose the earliest of the existing and current
 5. Remember the deadline for deserialization
 6. If the context has a parent context, use it to recurse upwards in the context stack
 7. Create our own struct to house the deadline time and cancellation func. This is what actually gets serialized.
 8. Serialize this struct across the wire, and receive it on the other side
 9. At deserialization, we know we need to create a timerCtx and cancellation function, so we create them

---
## Notes

 - Time is approximate at best between any two computers
 - Context is flattened during serialization, nothing pre-serialization can be popped off post-serialization

---

