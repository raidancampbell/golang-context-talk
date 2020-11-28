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

print() {
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
    return recoverContext().WithValue(ctx, inst, id)
}

func Get() *log.Logger { // replace with your logging library of choice
    id := recoverContext().Value(inst)
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

<!-- ## Rebuilding -->

```
func main() {
    panicker(context.WithValue(context.Background(), "key", "oops!"))
}
// A rather nasty regex for matching the ".(arg1, arg2" part of a stacktrace.
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
 - regex is A rather nasty regex for matching the ".(arg1, arg2" part of a stacktrace.
 - we walk through each line of the stack. the stack is walked from bottom (current execution) to top (creation of groutine)
 - once a match is encountered, parse the memory addresses into a uintptr
 - recreate the iface layout with those pointers as a uintptr array
 - create a pointer to that array
 - use type assertion to assert the type of that pointer as a context
 - use the context
-->