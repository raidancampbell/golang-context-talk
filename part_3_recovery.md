# ABUSING CONTEXT IN GO: Recovery

---

## Dynamic scoping
```
var x = 1
func main() {
    x = 2
    foo()
    bar()
    print()
}

func foo() {
    x = 3
    print()
}

func bar() {
    x = 4
    print()
}

func print() {
    fmt.Printf("%d", x)
}
```
- static/lexical scoping (what golang uses) prints `344`
- dynamic scoping would print `342`

<!-- 
 - dynamic scoping works its way up the callstack, and uses the first instance of a variable that it sees
 -->

---

## Why would this be useful for context?

Let's say I have a logging framework which retrieves some transaction-level metadata (IDs, start/stop times, etc...) for logging.
```
package logger

type key struct{}
var inst key

func Put(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, inst, id)
}

func Get(ctx context.Context) *log.Logger { // replace with your logging library of choice
    id := ctx.Value(inst)
    if id == "" {
        id = "undefined"
    }
    return log.New(os.Stdout, fmt.Sprintf("id: '%s', ", id), log.LstdFlags)
}
```
A sample invocation would be `logger.Get(ctx).Info("starting to do something")`
I'm a good author: if any of the metadata is missing, I don't fail and instead use some defaults.
But all my users are calling it with `logger.Get(context.TODO()).Info("starting to do something")`, even when they have a valid context being passed in!

---

## How can we fix this?

It's an admittedly contrived scenario, but we could leverage dynamic scoping to reach up the callstack to find an actual context.
We could also depend on dynamic scoping and eliminate the context parameter altogether:
```
package logger

type key struct{}
var inst key

func Put(id string) context.Context {
    return RecoverContext().WithValue(ctx, inst, id)
}

func Get() *log.Logger { // replace with your logging library of choice
    id := RecoverContext().Value(inst)
    if id == "" {
        id = "undefined"
    }
    return log.New(os.Stdout, fmt.Sprintf("id: '%s', ", id), log.LstdFlags)
}
```

---

## How can we implement this?

let’s note some of the restrictions placed on `contex.Context`.
Each restriction limits the work we have to do: narrower scope means less scenarios to handle.

 - A context SHOULD be the first parameter in the function signature. The builtin linter `lintContextArgs` checks for this
 - Context is an interface which contains four unexported implementations under the hood: deadline, cancel, value, and empty.

So we always know where a context will be in a function signature and also the legal values of its underlying concrete type.
The next tools for solving the problem come from understanding the internals of go. Specifically, the `interface`

---

## Interfaces at compile time

Interfaces, such as the `context.Context` interface, are constructed by two parts: an `itab` and the data. The `runtime2.go` source has it defined simply as:
```
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```
itab, short for "itable", short for "interface table", contains all the details the go linker and runtime require for ducktyping to work.
A more in-depth view is available at the go-internals Interfaces chapter. For the time being, we need to know two things:

1. The itab and data pointers make up an interface
1. Different instances of the same concrete interface implementation will recycle the same itab

---

## Interfaces at runtime

When an `interface` is passed at runtime, it's passed as its component itab and data tuple, instead of as a single value.
Take the below snippet and its resulting stacktrace:
```
import "context"

func main() {
    panicker(context.WithValue(context.Background(), "key", "oops!"))
}

//go:noinline
func panicker(ctx context.Context) {
    panic(ctx.Value("key"))
}
```
```
panic: oops!

goroutine 1 [running]:
main.panicker(0x1099b60, 0xc000068060)
        /Users/aidan/go/src/github.com/raidancampbell.github.io.source/content/scratch/abusing-context-part-ii.go:11 +0x61
main.main()
        /Users/aidan/go/src/github.com/raidancampbell.github.io.source/content/scratch/abusing-context-part-ii.go:6 +0x7a
```
One value (`context.Context`) was passed in the source, but at runtime the two interface components were seen: `main.panicker(0x1099b60, 0xc000068060)`.
<!-- question for the audience:
 - why does go pass the components of an interface like this? -->

---

<!-- ## Reading the stack at runtime -->

Let’s modify the above example to gain access to the stack as a string.  Minus the panic’s message, the results remain the same:
```
package main

import (
    "bufio"
    "bytes"
    "context"
    "fmt"
    "runtime"
)

func main() {
    panicker(context.WithValue(context.Background(), "key", "oops!"))
}

func panicker(_ context.Context) {
    var buf [8192]byte
    n := runtime.Stack(buf[:], false) // get the current callstack as a string
    sc := bufio.NewScanner(bytes.NewReader(buf[:n]))
    for sc.Scan() {
        fmt.Println(sc.Text())
    }
}
```
```
goroutine 1 [running]:
main.panicker(0x10ee820, 0xc000090180)
        /Users/aidan/go/src/github.com/raidancampbell.github.io.source/content/scratch/abusing-context-part-ii.go:17 +0x69
main.main()
        /Users/aidan/go/src/github.com/raidancampbell.github.io.source/content/scratch/abusing-context-part-ii.go:12 +0x7a
```

---

```
func main() {
    panicker(context.WithValue(context.Background(), "key", "oops!"))
}
// A rather nasty regex pattern for matching the ".(arg1, arg2" part of a stacktrace.
var twoParamPatt = regexp.MustCompile \
(`^.+[a-zA-Z][a-zA-Z0-9\-_]*\.[a-zA-Z][a-zA-Z0-9\-_]*\((?P<type_itab>0x[0-9a-f]+), (?P<type_data>0x[0-9a-f]+).+`)

func panicker(_ context.Context) {
    var buf [8192]byte
    n := runtime.Stack(buf[:], false) // get the current callstack as a string
    sc := bufio.NewScanner(bytes.NewReader(buf[:n]))
    for sc.Scan() {
        // match the expected regex.  for this example, we're only expecting the below match (addrs will vary):
        // main.panicker(0x10ee820, 0xc000090180)
        matches := twoParamPatt.FindStringSubmatch(sc.Text())
        if matches == nil {
            continue
        }

        // grab the two memory addresses (itab and data value)
        var p1, p2 uintptr
        _, err1 := fmt.Sscanf(matches[1], "%v", &p1)
        _, err2 := fmt.Sscanf(matches[2], "%v", &p2)
        if err1 != nil || err2 != nil {
            continue
        }

        // put the two pointers into the iface layout
        idata := [2]uintptr{p1, p2}

        // declare that the iface is a context
        ctx := *(*context.Context)(unsafe.Pointer(&idata))
        
        // use the context
        fmt.Println(ctx.Value("key"))
    }
}
```

<!-- 
 - rebuilding
 - regex is A rather nasty regex for matching the ".(arg1, arg2" part of a stacktrace.
 - we walk through each line of the stack. the stack is walked from bottom (current execution) to top (creation of groutine)
 - once a match is encountered, parse the memory addresses into a uintptr
 - recreate the iface layout with those pointers as a uintptr array
 - create a pointer to that array
 - use type assertion to assert the type of that pointer as a context
 - use the context
-->

---

```
func RecoverCtx() (context.Context, error) {
    return emptyItab(context.Background())
}

//go:noinline
func emptyItab(_ context.Context) (context.Context, error) {
    return valueItab(context.WithValue(context.Background(), "", ""))
}

//go:noinline
func valueItab(_ context.Context) (context.Context, error) {
    ctx, c := context.WithCancel(context.Background())
    defer c()
    return cancelItab(ctx)
}

//go:noinline
func cancelItab(_ context.Context) (context.Context, error) {
    ctx, c := context.WithDeadline(context.Background(), time.Now())
    defer c()
    return timerItab(ctx)
}

//go:noinline
func timerItab(_ context.Context) (context.Context, error) {
    return doGetCtx()
}

func doGetCtx() (context.Context, error) {
    // TODO
}
```

<!-- 
 - the itabs are fixed constructs from compile time
 - we created a known stack for the first few layers above our implementation
     - when recursing up the stack into the caller's code, we have known references for each implementation's itab
 - the go:noinline pragma -->

---
## Inlining
```
libraidan/pkg/runsafe.doGetCtx(0x13a1520, 0xc0000a6008, 0xbfae3de322870128, 0xbc60e)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:59 +0x5b
libraidan/pkg/runsafe.timerItab(...)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:52
libraidan/pkg/runsafe.cancelItab(0x13a14e0, 0xc0000e09c0, 0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:47 +0x9e
libraidan/pkg/runsafe.valueItab(0x13a15a0, 0xc00009d320, 0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:40 +0x82
libraidan/pkg/runsafe.emptyItab(0x13a1520, 0xc0000a6008, 0xc000040680, 0x100e808, 0x30, 0x130a680)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:33 +0x7e
libraidan/pkg/runsafe.RecoverCtx(...)
```

<!-- 
 - the compiler "elides" the symbols of an inlined function. 
 - it's a compiler optimization to remove the overhead of a function call for shorter functions 
 - the timerItab function was inlined: its implelentation was just a call to another function.
-->

---
## Forcing no inline
```
libraidan/pkg/runsafe.doGetCtx(0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:59 +0xbb
libraidan/pkg/runsafe.timerItab(0x14b5ce0, 0xc0000a44e0, 0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:52 +0x4c
libraidan/pkg/runsafe.cancelItab(0x14b5c60, 0xc0000e8c40, 0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:47 +0x1a5
libraidan/pkg/runsafe.valueItab(0x14b5d20, 0xc00009d320, 0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:40 +0x150
libraidan/pkg/runsafe.emptyItab(0x14b5ca0, 0xc0000a6008, 0x0, 0x0, 0x0, 0x0)
    /Users/aidan/go/src/libraidan/pkg/runsafe/context.go:33 +0xd7
libraidan/pkg/runsafe.RecoverCtx(0x0, 0x0, 0x0, 0x0)
```

---
## Where are we now?
 - a `context` will always be the first argument, if present
 - we can successfully dump a stack and rebuild a context from it
 - golang natively exposes only 4 implementations of the `Context` interface
 - We have a fixed stack above our (TODO) implementation, with each of the standard context implementations
 - We know `itab`s are fixed memory locations determined at compile time

Given the above, the final implementation of context recovery becomes trivial:
as we move up the callstack, note the `itab` pointer of each implementation.
Once we pass into our caller's code, look for pointer matches on the first parameter.
If one is found, take the first two parameters and turn them into a context.

---
```
    stackMatch++

    // grab the two memory addresses (itab and type value)
    var p1, p2 uintptr
    _, err1 := fmt.Sscanf(matches[1], "%v", &p1)
    _, err2 := fmt.Sscanf(matches[2], "%v", &p2)
    if err1 != nil || err2 != nil {
        continue
    }

    // build up the legal values for each implementation of context
    // the stackMatch must match the known location in the stack.
    // Otherwise we might return a malformed context
    if stackMatch == 1 && strings.Contains(sc.Text(), "timerItab") {
        deadlineType = p1
    } else if stackMatch == 2 && strings.Contains(sc.Text(), "cancelItab") {
        cancelType = p1
    } else if stackMatch == 3 && strings.Contains(sc.Text(), "valueItab") {
        valueType = p1
    } else if stackMatch == 4 && strings.Contains(sc.Text(), "emptyItab") {
        emptyType = p1
    } else if p1 != emptyType && p1 != valueType && p1 != cancelType && p1 != deadlineType {
        // if we're in the caller's code, and the first parameter isn't a 
        // known context implementation, then skip this stack frame
        continue
    }

    if stackMatch <= 4 { // we're still building the legal context implementations
        continue
    }
    // at this point we're done building the legal context implementations, 
    // and this matched one. rebuild a context from the addresses, and return
    idata := [2]uintptr{p1, p2}
    return *(*context.Context)(unsafe.Pointer(&idata)), nil
```

---

## Presentation source
```
█████████████████████████████████████████████	█████████████████████████████████████████████
█████████████████████████████████████████████	█████████████████████████████████████████████
████ ▄▄▄▄▄ █▀█ █▄█▀▀▄ ▄▄▀█▀▄█▄▀▀ █ ▄▄▄▄▄ ████	████ ▄▄▄▄▄ █ ▀▀▄▀▀▄▀▀▀▀▀▀▄▀█▄█▀▀▀█ ▄▄▄▄▄ ████
████ █   █ █▀▀▀█ ▀ ▀ ▀█ █▄ ▄▀▀▄█▄█ █   █ ████	████ █   █ █▄█ ▀█ ▄▀ █▄ ▄▀▄▄▄▀ ▄▄█ █   █ ████
████ █▄▄▄█ █▀ █▀▀▀▄▀ ▄▄▄▄▀▀▄█ ▀▀ █ █▄▄▄█ ████	████ █▄▄▄█ █▀▀▀▀▄▄▀ █ █  ▄█▀  █▀▀█ █▄▄▄█ ████
████▄▄▄▄▄▄▄█▄▀ ▀▄█▄█ █▄█ ▀▄▀▄█ █▄█▄▄▄▄▄▄▄████	████▄▄▄▄▄▄▄█ █▄█▄▀ █▄▀ █▄▀ █▄█▄█ █▄▄▄▄▄▄▄████
████▄▄ ▄▄▀▄▄ ▄▀▄▀  █▀▄▀ ▀▀▄  ▄ ▄▀ █ █ ▀ █████	█████▀▄ ▄█▄█▄▄▄▀ ▀▄▄  █▄███▀█▀█▀█▄▄ █▀▄▄▄████
████ █▄█▄█▄▄ ▄▄█▀█▀▄▄█▄█▄ ▀█▄▄█▄▀  █▄ ▄▀▄████	████▄█ █▀ ▄ ▄█ ██▀█▄  ▀ ▀█▄▀█▄  ▀█ ▄▄▄█▄▄████
█████▀▀ ██▄▄ ▄▄█▄ █▄▀ ▀▀▀█▄ ▄▄▄██ ▄▀▄▄ ▀▄████	████▄▄▀▄▄ ▄  ▀▄█▀ ▄▀▀▄█ ▄▄▄▀█ ▄▄▀ ▄▀▄▀ ▄█████
████▀  ▀▀ ▄▄█▀▄ ▄▄▄  █▄█▀▀▄█▄█▀ ▀▀▄ █▀▄▄ ████	████▀█▀  ▀▄█ ▀█▀▄▄  █ ▀▄ █▀ █▄   ▄█▄█▄█▄ ████
█████▄▄ ▀▀▄▀██▀▄▀ ▄▄▀ ▀▀ ▀▄ ▄ ▄▄ ▄▄ ▄▄ ▀▄████	████ ▄▀ █▄▄▄▀▀█▀▀  ▄▄█▀▄ ▄▄ ▄  █▀▄▄█▄▀ ▄▄████
████▄ ██▄█▄▀▄ ▀█▀█ ▄█▄▀█ ▀ ███▀▄▀  █▀█▄▄ ████	████▄▄▄██▀▄█▄ ▀ ▀██▄ ▄ ▀▄▄█ ▄██▄ ▀██▄▄█  ████
████▄▄▀▀▄█▄ ▀▄▀█▄▄▄▄    ███▀▄▄▄█▀ ▄▀▄▀▄ ▄████	████▄▀█ ▀▄▄▄ ▀▄▄█ ▀▄ ██▀█▄█ █  ███▄ ▄▀▄▀█████
████▄█▀▀█ ▄▀ █▀ ▄█▄▄ ▄██▀  ▀ █▄  █▄▀█▄▄▄ ████	████ ▀▀ ▄▀▄█▄▀  ▄▄▀ ▄█▄█▄ ▄█▄ ▄▄█  ▀███ ▄████
████ █ ▀█▄▄██▀ ▄▀▀█▀ ▄█▀ ▀▄▀▄▄▄▄▀▄▄ ▄ ▄ ▄████	████ █▄ ▀▄▄ ▀▀▄▀ █▄ █ ▀▄▀▀▄▄▄█▄█ ▄▄ ▄▀▄▄█████
████ █ ▀  ▄██▄██▀ ▄▄▄█ ██▀▀█ █▀ ▀▀▄▄▀██▄ ████	████ ██▄█ ▄█ ▀  ▄█▀▄ ▄▀▀█▄█ ▄▄ ▄▀▀▄ █▄▄  ████
████▄█▄█▄▄▄▄ █ █▄█▄▄▄▄▀  █▄▀█ ▄▄ ▄▄▄ █▄██████	████▄█▄██▄▄█ ▄▀     ▄▀▀█▄ ▄  ▄▄▀ ▄▄▄ ▄▄▀█████
████ ▄▄▄▄▄ █▄▄  ▄█▀▄ █▄█ ▀ ██▄▄█ █▄█ ▀▄▄ ████	████ ▄▄▄▄▄ ████▄▀▄██▀▀▄  ▄▀█▀▄▄█ █▄█ ▄█  ████
████ █   █ █ ██▄▀▀▀▀ ▄█▀▀█▄ ▄▄▄    ▄ ▄  ▀████	████ █   █ █ ▄ █▄▄█▄████▄▄▀█▀▄▄▄▄▄ ▄ █▄██████
████ █▄▄▄█ █ ▀ █▀ ▄ ▀▄ █▀▀▄█▀▄▀▀ ▀█ ▀▄▄▄ ████	████ █▄▄▄█ █▄▀▀██▄▀█  ▄██▀ █▀█▄▀ ▄▀ ████ ████
████▄▄▄▄▄▄▄█▄▄██▄█▄▄▄▄██▄█▄█▄▄▄██▄█▄█▄▄▄▄████	████▄▄▄▄▄▄▄███▄█▄█▄█▄▄█▄▄███▄▄▄██▄██▄▄▄▄▄████
█████████████████████████████████████████████	█████████████████████████████████████████████
█████████████████████████████████████████████	█████████████████████████████████████████████
                     [1]                                              [2]
```
1: https://raw.githubusercontent.com/raidancampbell/golang-context-talk/main/part_3_recovery.md
2: https://github.com/vinayak-mehta/present
