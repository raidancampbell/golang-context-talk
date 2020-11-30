# A BRIEF INTRODUCTION TO CONTEXT
```
“Package context defines the Context type, which carries deadlines, cancellation signals,
and other request-scoped values across API boundaries and between processes.”
 - golang.org
```

---

## A First Encounter: no-ops

"This function needs something called a `context`.
Stack Overflow gives some confusing results, but `context.Background()` is a one-liner I can paste in."

```golang
context.Background()
```
or
```golang
context.TODO()
```

<!--
 - it's the default datatype and therefore pretty safe
 - it's an acceptable solution for nascent codebases, but context.TODO() is preferred
-->

---
## A Second Glance: deadlines

"I need to call another system, but my own API budget is 30 seconds and I have 2 external calls and a database query to execute"

```golang
ctx, cancel := context.WithTimeout(context.Background(), 10 * time.Second)
defer cancel()
```
Examples are seen in `net/http`'s `NewRequestWithContext` or `database/sql`'s `QueryContext`.
Both have respectice sister functions `NewRequest` and `Query`, which look like this: 

```golang
// NewRequest wraps NewRequestWithContext using the background context.
func NewRequest(method, url string, body io.Reader) (*Request, error) {
	return NewRequestWithContext(context.Background(), method, url, body)
}
```
The function implementation always requires context, there's a wrapper for when the caller doesn't care to pass one.

<!--
 - creating a context with a 1 minute deadline from a parent of a 5 minute deadline yields a 1 minute deadline
 - creating a context with a 5 minute deadline from a parent of a 1 minute deadline yields a 1 minute deadline
 	- this is done during creation by a branch in the library
 - WithTimeout and WithDeadline both resolve to a deadline, which is a defined point in time, not a duration of time.
 	- TODO: why is that important to know?
 - also returns a cancel function
 	- always defer the cancel(), executing it twice has no harmful effects
 	- prevents leaking contexts
 		- similar to creating goroutines in a loop, creating contexts in a loop has the potential for leaks
 		- leaks are limited by parent contexts: if the parent context ends, then any leaked contexts are cleaned up
-->

---
## A Third Chance: values

"I want some nice-to-have tracing and metrics in my code"

```golang
type timingContextKey struct{}
ctx := context.WithValue(context.Background(), timingContextKey, time.Now())
```
<!-- 
 - a parent context's value is never overwritten for a given key, it's shadowed
 	- as frames of context fall off the stack, it will become accessible again
 - keys should be unexported structs: this guarantees no collisions
 -->

---
<!-- fg=black bg=red -->
# WARNING

valueCtx is effectively a `map[interface{}]interface{}`.
Storing any required data in the context amounts to subverting the type system: you lose any guarantee that your data exists.
valueCtx is meant for systems that can tolerate failures, such as metrics or arguably logging.
<!-- 
 - at its extreme, what's the point in having _any_ variable in the signature if you can have _all_ of them in a single parameter?
 -->
---
## Key point: Contexts are composed

```
Contexts are composed: each child context is made by adding additional cancellations or values from its parent.
As soon as those contexts fall out of scope their additions, whether they be tighter deadlines or additional data, are lost.
```

This is similar to scopes in programming languages: further down the callstack has inherited (or had the opportunity to inherit)
data from its callers up the callstack.

```
+------------------------------------------------------------------------+
|deadlineCtx, cancelFunc := context.WithDeadline(valCtx, 1 * time.Second)|
|           +--------------------------------------------------+         |
|           |valCtx := context.WithValue(rootCtx, 'foo', 'bar')|         |
|           |       +-------------------------------+          |         |
|           |       |rootCtx := context.Background()|          |         |
|           |       +-------------------------------+          |         |
|           |            key 'foo', value 'bar'                |         |
|           +--------------------------------------------------+         |
|                             deadline 1 second                          |
+------------------------------------------------------------------------+
```

---
## Context Flows

```
"Think long and hard if you ever write a line that looks like

x = ctx

If you still want to write it, think long and hard again."
```

Storing a reference to a context opens the door for conflicts with flowing contexts.

---
## Implementation Patterns
 - most time expensive operations in golang are IO, such as HTTP requests or database queries
 	- we can trust that these implementations handle context cancellations correctly
 	- for anything else, the below code snippet should work well enough, particularly if it's been a while since the last check
```
select {
default:
case <-ctx.Done():
	return nil, ctx.Err()
}
```
 - in idiomatic go a `context` will always be the first argument, if present
 - to prevent leaks, anytime a context is created with a cancel func, immediately defer the cancel func:
 ```
ctx, c := context.WithCancel(context.Background())
defer c()
 ```
 - when using context to store data, the key for the data should always be an unexported struct
 - context flows and should rarely (if ever) be stored in a variable

---

## Presentation source
```
█████████████████████████████████████████████    █████████████████████████████████████████████
█████████████████████████████████████████████    █████████████████████████████████████████████
████ ▄▄▄▄▄ █▀ ▀▄█ ▀▀  ▄▄▀█▀▄█▄▀▀ █ ▄▄▄▄▄ ████    ████ ▄▄▄▄▄ █ ▀▀▄▀▀▄▀▀▀▀▀▀▄▀█▄█▀▀▀█ ▄▄▄▄▄ ████
████ █   █ █▀ █▀▀█▀▀▄▀█ █▄ ▄▀▀▄█▄█ █   █ ████    ████ █   █ █▄█ ▀█ ▄▀ █▄ ▄▀▄▄▄▀ ▄▄█ █   █ ████
████ █▄▄▄█ █▀██▄▀  ▀ ▄▄▄▄▀▀▄█ ▀▀ █ █▄▄▄█ ████    ████ █▄▄▄█ █▀▀▀▀▄▄▀ █ █  ▄█▀  █▀▀█ █▄▄▄█ ████
████▄▄▄▄▄▄▄█▄▀▄▀▄█ ▀▄█▄█ ▀▄▀▄█ █▄█▄▄▄▄▄▄▄████    ████▄▄▄▄▄▄▄█ █▄█▄▀ █▄▀ █▄▀ █▄█▄█ █▄▄▄▄▄▄▄████
████ ▄ ▄ █▄ ▄ █  ▄▄█▀▄▀ ▀▀▄  ▄ ▄▀ █ █ ▀ █████    █████▀▄ ▄█▄█▄▄▄▀ ▀▄▄  █▄███▀█▀█▀█▄▄ █▀▄▄▄████
████ ▄█▀▀ ▄█▀▀█▀▀▄▀▄▄█▄█▄ ▀█▄▄█▄▀  █▄ ▄▀▄████    ████▄█ █▀ ▄ ▄█ ██▀█▄  ▀ ▀█▄▀█▄  ▀█ ▄▄▄█▄▄████
████▀ █▀▄ ▄▀▄▄▀▀█ ▄▄  ▀▀▀█▄ ▄▄▄██ ▄▀▄▄ ▀▄████    ████▄▄▀▄▄ ▄  ▀▄█▀ ▄▀▀▄█ ▄▄▄▀█ ▄▄▀ ▄▀▄▀ ▄█████
████▄██▄▄▄▄▀██▄▀▄▄   █▄█▀▀▄█▄█▀ ▀▀▄ █▀▄▄ ████    ████▀█▀  ▀▄█ ▀█▀▄▄  █ ▀▄ █▀ █▄   ▄█▄█▄█▄ ████
████▄▀▀▀▀█▄██ ▄ ▄ ▄▄▀ ▀▀ ▀▄ ▄ ▄▄ ▄▄ ▄▄ ▀▄████    ████ ▄▀ █▄▄▄▀▀█▀▀  ▄▄█▀▄ ▄▄ ▄  █▀▄▄█▄▀ ▄▄████
█████▀ ▀  ▄█ ▀▀▀█▄▄ ▀▄▀█ ▀ ███▀▄▀  █▀█▄▄ ████    ████▄▄▄██▀▄█▄ ▀ ▀██▄ ▄ ▀▄▄█ ▄██▄ ▀██▄▄█  ████
██████▄█▀▄▄ ▀ ▄ ▄▄ ▄▀   ███▀▄▄▄█▀ ▄▀▄▀▄ ▄████    ████▄▀█ ▀▄▄▄ ▀▄▄█ ▀▄ ██▀█▄█ █  ███▄ ▄▀▄▀█████
█████▄  ██▄▄█  █▀▄▀▄ ▄██▀  ▀ █▄  █▄▀█▄▄▄ ████    ████ ▀▀ ▄▀▄█▄▀  ▄▄▀ ▄█▄█▄ ▄█▄ ▄▄█  ▀███ ▄████
████▀███ ▄▄▀▄ ▀ ▄ ▄▀ ▄█▀ ▀▄▀▄▄▄▄▀▄▄ ▄ ▄ ▄████    ████ █▄ ▀▄▄ ▀▀▄▀ █▄ █ ▀▄▀▀▄▄▄█▄█ ▄▄ ▄▀▄▄█████
████ ██▀▄█▄▄ ▀▄█▄█▀ ▄█ ██▀▀█ █▀ ▀▀▄▄▀██▄ ████    ████ ██▄█ ▄█ ▀  ▄█▀▄ ▄▀▀█▄█ ▄▄ ▄▀▀▄ █▄▄  ████
████▄██▄█▄▄█▀  ▀█▄▄▄▄▄▀  █▄▀█ ▄▄ ▄▄▄ █▄██████    ████▄█▄██▄▄█ ▄▀     ▄▀▀█▄ ▄  ▄▄▀ ▄▄▄ ▄▄▀█████
████ ▄▄▄▄▄ █▄▀▄█▀█▄▄ █▄█ ▀ ██▄▄█ █▄█ ▀▄▄▄████    ████ ▄▄▄▄▄ ████▄▀▄██▀▀▄  ▄▀█▀▄▄█ █▄█ ▄█  ████
████ █   █ █ █▄▀▄▄▄▀ ▄█▀▀█▄ ▄▄▄    ▄ ▄ ██████    ████ █   █ █ ▄ █▄▄█▄████▄▄▀█▀▄▄▄▄▄ ▄ █▄██████
████ █▄▄▄█ █  ▀▀ █  ▀▄ █▀▀▄█▀▄▀▀ ▀█ ▀▄▄▄ ████    ████ █▄▄▄█ █▄▀▀██▄▀█  ▄██▀ █▀█▄▀ ▄▀ ████ ████
████▄▄▄▄▄▄▄█▄████▄▄▄▄▄██▄█▄█▄▄▄██▄█▄█▄▄▄▄████    ████▄▄▄▄▄▄▄███▄█▄█▄█▄▄█▄▄███▄▄▄██▄██▄▄▄▄▄████
█████████████████████████████████████████████    █████████████████████████████████████████████
█████████████████████████████████████████████    █████████████████████████████████████████████
                     [1]                                              [2]
```
1: https://raw.githubusercontent.com/raidancampbell/golang-context-talk/main/part_1_design_considerations.md
2: https://github.com/vinayak-mehta/present

