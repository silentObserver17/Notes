# 1. Context (Conceptual Mastery)
---
## What Problem It Solves

> How do you control **lifecycle, cancellation, deadlines, and request-scoped data** across goroutines?

## Core Ideas

- Propagates **cancellation signals**
- Carries **deadlines/timeouts**
- Passes **request-scoped metadata**

**The interface:**
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)  // when will it cancel?
    Done() <-chan struct{}                     // closed when cancelled
    Err() error                               // why was it cancelled?
    Value(key any) any                        // request-scoped values
}
```

**The four constructors:**
```go
// Root contexts — never cancelled on their own
ctx := context.Background()  // top-level, always use this as root
ctx := context.TODO()        // placeholder when you're not sure yet — same as Background internally

// Derived contexts — form a tree
ctx, cancel := context.WithCancel(parent)       // manual cancellation
ctx, cancel := context.WithTimeout(parent, 5*time.Second)  // deadline = now + duration
ctx, cancel := context.WithDeadline(parent, t) // explicit deadline time
ctx        := context.WithValue(parent, key, val)  // attach a value
```

**Always call cancel.** It releases resources even if the context expires on its own:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()  // always defer immediately after creation
```

**How cancellation propagates — the tree:**
```
Background
    └── WithCancel (A)
            ├── WithTimeout (B)   ← cancelled when A cancelled OR timeout
            └── WithValue (C)     ← cancelled when A cancelled
```

Cancelling a parent cancels all children. Children cannot cancel parents.

**Checking cancellation in your code:**
```go
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // context.Canceled or context.DeadlineExceeded
        default:
            // do actual work
            process()
        }
    }
}

// For blocking calls, pass ctx directly
rows, err := db.QueryContext(ctx, "SELECT ...")
resp, err := http.NewRequestWithContext(ctx, "GET", url, nil)
```

**`ctx.Err()` values:**
```go
context.Canceled          // parent called cancel()
context.DeadlineExceeded  // timeout hit
```

**Values in context — the right way:**
```go
// Define unexported key type to avoid collisions across packages
type contextKey string
const requestIDKey contextKey = "requestID"

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func RequestIDFrom(ctx context.Context) (string, bool) {
    id, ok := ctx.Value(requestIDKey).(string)
    return id, ok
}
```

**What context is NOT for:**

- Passing optional function arguments (use functional options for that).
- Passing database connections or loggers (put those in structs).
- Storing business logic data — only request-scoped cross-cutting concerns (request ID, auth token, trace ID).

**Gotchas:**

- **Never store context in a struct.** Pass it as the first argument to every function that needs it.
- `context.WithValue` uses interface key comparison — always use an unexported custom type as the key to prevent collision with other packages using the same string.
- A cancelled context passed to a DB query will cause the query to fail even if the DB has already done the work — the result is discarded. This can leave dangling DB-side operations. Be aware in write paths.

### Real-World Use

- HTTP request lifecycle
- DB query timeout
- Microservice communication
- Worker shutdown

# 2. Reflection
---
Reflection lets you **inspect and manipulate types and values at runtime** without knowing them at compile time. The entry point is the `reflect` package, but the concepts matter regardless.

**Two core types: `reflect.Type` and `reflect.Value`:**
```go
var x float64 = 3.14

t := reflect.TypeOf(x)    // reflect.Type  — describes the type
v := reflect.ValueOf(x)   // reflect.Value — holds the actual value

fmt.Println(t.Kind())     // float64
fmt.Println(v.Float())    // 3.14
```

**Kind vs Type:**
```go
type MyInt int

var m MyInt = 5
t := reflect.TypeOf(m)
fmt.Println(t.Name())   // "MyInt"  — the named type
fmt.Println(t.Kind())   // "int"    — the underlying kind
```

Kind is the underlying primitive category. Type is the declared name.

**Modifying values — requires pointer + Elem():**

```go
x := 42
v := reflect.ValueOf(x)
v.SetInt(99)  // PANICS — v is not addressable

// Correct way: pass pointer, call Elem()
v = reflect.ValueOf(&x).Elem()
v.SetInt(99)  // works — x is now 99
```

**Struct reflection — the most common use case:**

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    age   int    // unexported
}

u := User{Name: "Alice", Email: "a@b.com"}
t := reflect.TypeOf(u)
v := reflect.ValueOf(u)

for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)             // reflect.StructField
    value := v.Field(i)            // reflect.Value

    fmt.Println(field.Name)        // "Name", "Email", "age"
    fmt.Println(field.Tag.Get("json"))  // "name", "email", ""
    fmt.Println(field.IsExported()) // true, true, false
    fmt.Println(value.Interface())  // panics on unexported field!
}
```

**Calling methods dynamically:**
```go
type Greeter struct{}
func (g Greeter) Hello(name string) string { return "Hello, " + name }

g := Greeter{}
v := reflect.ValueOf(g)
method := v.MethodByName("Hello")
result := method.Call([]reflect.Value{reflect.ValueOf("Alice")})
fmt.Println(result[0].String())  // "Hello, Alice"
```

**When reflection is appropriate:**

- Writing generic serializers/deserializers (like `encoding/json`).
- ORMs mapping structs to DB columns via struct tags.
- Dependency injection containers.
- Test helpers that need to compare arbitrary structs.

**When it is NOT appropriate:**

- Anywhere performance matters — reflection is 10–100x slower than direct calls.
- When generics can do the job (Go 1.18+) — prefer generics for type-safe abstractions.
- When it makes code hard to reason about — reflection errors are runtime panics, not compile errors.

**Gotchas:**

- `reflect.Value.Interface()` panics on unexported fields.
- `CanSet()` returns false if the value isn't addressable — always check before `Set*()`.
- `reflect.DeepEqual` is often the only reason people reach for reflection in tests — it works but is slow and has surprising behavior with nil slices vs empty slices.

# 3. Unsafe Package (Conceptual)
---
`unsafe` breaks Go's type safety. The compiler gives you raw memory access. There is no GC tracking of unsafe pointers — you are responsible.

**The three core operations:**

```go
unsafe.Sizeof(x)     // size in bytes of x's type (compile-time constant)
unsafe.Alignof(x)    // alignment requirement of x's type
unsafe.Offsetof(s.f) // byte offset of field f in struct s
```

