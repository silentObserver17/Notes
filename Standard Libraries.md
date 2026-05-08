## `context`
---
You already covered the API from the previous section. Here we go deeper — **internals, propagation mechanics, and production patterns**.

#### How the cancellation tree actually works

Every derived context holds a reference to its parent. When you call `WithCancel`, Go registers the child with the parent internally. When the parent cancels, it iterates its children and cancels them all — recursively.

```go
// Conceptually what WithCancel does internally:
type cancelCtx struct {
    parent   Context
    mu       sync.Mutex
    done     chan struct{}
    children map[canceller]struct{}  // all derived children
    err      error
}

func (c *cancelCtx) cancel(err error) {
    c.mu.Lock()
    c.err = err
    close(c.done)                        // unblocks all <-ctx.Done() receivers
    for child := range c.children {
        child.cancel(err)                // propagate down the tree
    }
    c.children = nil
    c.mu.Unlock()
}
```

The `Done()` channel is **lazily created** on first access. If you never read `Done()`, the channel is never allocated.

1. **Minimal Resource Usage**

When you create a new context (e.g., `context.WithCancel(parent)` or `context.WithTimeout(...)`), Go doesn't immediately allocate a channel object in memory. A channel is a relatively heavy structure. Instead, the context holds a placeholder or `nil` value for the channel until it is actually needed.

2. **"Lazy" Means "Only When Needed"** 

The channel is only allocated and properly initialized the first time you call the Done() method.

- **If you pass around a context** but never actually check `<-ctx.Done()` (e.g., in a `select` statement), the channel is never created.
- This saves memory and CPU time, particularly in applications that create thousands of contexts for request-scoped operations that finish quickly without needing to be cancelled.

3. **What if you never call Done()?**

If you never call `ctx.Done()`, that channel is never allocated, which avoids unnecessary allocations in the heap. 

#### **Summary Table**

| Scenario           | Does `Done()` trigger? | Who triggers it?       | Why?                                           |
| ------------------ | ---------------------- | ---------------------- | ---------------------------------------------- |
| **Success (Fast)** | No                     | No one                 | Request finished before any signal was needed. |
| **Ctrl+C**         | **Yes**                | Server Shutdown        | To stop all active work immediately.           |
| **Timeout**        | **Yes**                | Internal Timer         | To prevent the request from running forever.   |
| **Manual Cancel**  | **Yes**                | Your code (`cancel()`) | You decided to stop the work early.            |
#### Propagation through goroutines

Context doesn't automatically propagate into goroutines — you have to pass it:
```go
func (s *Service) ProcessOrder(ctx context.Context, orderID int) error {
    // WRONG — goroutine is completely detached from cancellation
    go func() {
        s.sendNotification(orderID)
    }()

    // CORRECT — goroutine respects caller's cancellation
    go func(ctx context.Context) {
        s.sendNotification(ctx, orderID)
    }(ctx)

    return nil
}
```

#### Cascading timeout pattern
Each layer of your stack should narrow the deadline, never widen it:

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Incoming request already has a deadline from the server
    ctx := r.Context()

    // Service layer gets a tighter budget
    svcCtx, cancel := context.WithTimeout(ctx, 4*time.Second)
    defer cancel()

    // DB call gets even tighter
    dbCtx, dbCancel := context.WithTimeout(svcCtx, 2*time.Second)
    defer dbCancel()

    result, err := s.db.QueryContext(dbCtx, query)
    // ...
}
```

If the top-level context (from the HTTP server) cancels first, all children cancel automatically regardless of their own deadlines.

#### The select pattern — the core concurrency idiom with context
```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // clean exit

        case job, ok := <-jobs:
            if !ok {
                return nil  // channel closed, we're done
            }
            if err := process(ctx, job); err != nil {
                return err
            }
        }
    }
}
```

#### Detecting timeout vs explicit cancel
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

doSomething(ctx)

switch ctx.Err() {
case context.DeadlineExceeded:
    // timeout hit — maybe retry with backoff
case context.Canceled:
    // caller cancelled — don't retry
case nil:
    // no error
}
```

#### Context values — the right and wrong way
```go
// WRONG — string key collides across packages
ctx = context.WithValue(ctx, "userID", 123)

// WRONG — exported key type, still collision-prone
type Key string
ctx = context.WithValue(ctx, Key("userID"), 123)

// CORRECT — unexported type, guaranteed no collision
type ctxKey int
const (
    ctxKeyUserID ctxKey = iota
    ctxKeyTraceID
    ctxKeyRequestID
)

func WithUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, ctxKeyUserID, id)
}

func UserIDFrom(ctx context.Context) (int, bool) {
    id, ok := ctx.Value(ctxKeyUserID).(int)
    return id, ok
}
```

#### Gotchas

- **`context.WithValue` lookup is O(n)** — it walks the parent chain comparing keys. Don't store many values; bundle them in a struct if needed.
-  **Never pass a cancelled context to cleanup work** — if your handler cancels but you still need to write a response or log something, use `context.Background()` or a fresh context for that work. Use `context.WithoutCancel`. (Go 1.21+) This creates a new context that keeps all the **values** (like your log-id or auth tokens) but **ignores the cancellation signal** of the parent
- **`http.Request.Context()`** is cancelled when the client disconnects. This means if your DB query is running and the client drops, the query gets killed via context — which is usually what you want.

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    err := doWork(ctx)
    if err != nil {
        // DETACH here for cleanup
        cleanupCtx := context.WithoutCancel(ctx) 
        log.Info(cleanupCtx, "Work failed, cleaning up...") // This will now actually run!
        db.Exec(cleanupCtx, "UPDATE status SET failed=true")
    }
}

```

### CONTEXT IMPORTANT POINTS: 
---
#### 1. **context ownership**
The rule:
> The function that creates a derived context owns its cancellation.

```GO
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    defer cancel()

    service.Process(ctx)
}
```

Here:

- handler created the timeout
- handler owns `cancel`
- service should NEVER call `cancel`

This creates clean lifecycle boundaries.

A useful mental model:

```
creator == owner == canceller
```

#### 2. Important nuance: cancellation is cooperative

Context does NOT forcibly stop goroutines.
It only broadcasts:

```go
<-ctx.Done()
```

Your goroutines must voluntarily stop.

Bad:
```go
go func() {
    for {
        computeForever()
    }
}()
```

Good:
```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            compute()
        }
    }
}()
```

#### 3. leaked goroutines from ignored contexts

This is probably the #1 real-world concurrency bug.
**Example:**
```go
func worker(ctx context.Context, jobs <-chan Job) {
    for job := range jobs {
        process(job)
    }
}
```

**Problem:**
- worker ignores cancellation
- goroutine survives request death
- memory + CPU leak

Proper:

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return

        case job, ok := <-jobs:
            if !ok {
                return
            }

            process(job)
        }
    }
}
```

Worth emphasizing heavily.

#### 4. Clarify: `WithValue` is NOT immutable data mutation
This misconception appears often.
This:
```go
ctx = context.WithValue(ctx, key, value)
```

does NOT mutate the existing context.

It creates:
```
new context -> parent context
```

Like:
```
ctx3 -> ctx2 -> ctx1 -> Background
```

Value lookup walks upward through the chain.

That explains:

- O(n) lookup
- shadowing behavior

**Example:**
```go
ctx1 := context.WithValue(ctx, "x", 1)
ctx2 := context.WithValue(ctx1, "x", 2)
ctx2.Value("x") // 2
```

because nearest parent wins.


#### 5. contexts should be short-lived

A subtle but important design principle.

Contexts are:
- request-scoped
- operation-scoped

NOT application state.

Bad:

```go
type App struct {    
	ctx context.Context
}
```

because:
- contexts expire
- deadlines become stale
- cancellation accidentally propagates forever

Contexts should flow downward temporarily.

#### 6.  Clarify timeout layering behavior

This is slightly subtle.

Child deadlines cannot extend parent deadlines.

**Example:**
```go
parent: 5s timeout
child: 30s timeout
```

Actual child deadline:

```go
5s
```

The earliest deadline always wins.

#### 7. `errgroup`

In modern Go concurrency, context and `errgroup` are deeply connected.
```go
g, ctx := errgroup.WithContext(ctx)

g.Go(func() error {
    return fetchUser(ctx)
})

g.Go(func() error {
    return fetchOrders(ctx)
})

g.Go(func() error {
    return fetchPayments(ctx)
})

if err := g.Wait(); err != nil {
    return err
}
```

Key idea:

- first goroutine error cancels shared context
- all sibling goroutines stop
- structured concurrency

This is a HUGE modern Go pattern.

##  `sync`  Coordination & Safety
---
#### What Problem It Solves

> Prevent **race conditions** when multiple goroutines access shared memory.

#### `sync.Mutex` and `sync.RWMutex`

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v[key]++
}

func (c *SafeCounter) Value(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.v[key]
}
```

`RWMutex` — multiple concurrent readers OR one writer:

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()            // multiple goroutines can hold RLock simultaneously
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, val string) {
    c.mu.Lock()             // exclusive — blocks all readers and writers
    defer c.mu.Unlock()
    c.data[key] = val
}
```

**When to use RWMutex vs Mutex:**
- Read-heavy workloads with infrequent writes → `RWMutex` wins.
- Write-heavy or balanced → `Mutex` is simpler and often faster (RWMutex has more overhead).
- If in doubt, start with `Mutex`.

**Critical rules:**

- Never copy a Mutex after first use — it contains internal state. Always embed by value in a struct, pass struct by pointer.
- Lock/Unlock must always be paired. The `defer mu.Unlock()` pattern is idiomatic and prevents forgetting.
- Don't hold a lock across a channel send/receive or any blocking operation — deadlock risk.

```go
// DEADLOCK — holding lock while waiting for a channel
func bad(mu *sync.Mutex, ch chan int) {
    mu.Lock()
    ch <- 1     // blocks waiting for receiver — nobody can acquire mu
    mu.Unlock()
}
```

#### `sync.WaitGroup`
Waits for a collection of goroutines to finish.

```go
func processAll(items []Item) {
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)                   // increment BEFORE starting goroutine
        go func(item Item) {
            defer wg.Done()         // decrement when done
            process(item)
        }(item)                     // pass item as arg — avoid loop capture
    }

    wg.Wait()                       // blocks until counter reaches 0
}
```

**With error collection:**
```go
func processAll(ctx context.Context, items []Item) error {
    var (
        wg   sync.WaitGroup
        mu   sync.Mutex
        errs []error
    )

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            if err := process(ctx, item); err != nil {
                mu.Lock()
                errs = append(errs, err)
                mu.Unlock()
            }
        }(item)
    }

    wg.Wait()
    return errors.Join(errs...)  // Go 1.20+
}
```

**Gotchas:**

- `Add` must happen **before** the goroutine starts — if you call `Add` inside the goroutine, `Wait` might return before `Add` is even called.
- `Done` is `Add(-1)`. Calling `Done` more times than `Add` panics (counter goes negative).
- `WaitGroup` can be reused after `Wait` returns, but only after the counter reaches zero.

#### `sync.Once`

Guarantees a function runs exactly once, even across concurrent callers. The canonical singleton pattern in Go.

```go
type DB struct {
    once sync.Once
    conn *sql.DB
}

func (d *DB) getConn() *sql.DB {
    d.once.Do(func() {
        var err error
        d.conn, err = sql.Open("postgres", dsn)
        if err != nil {
            panic(err)
        }
    })
    return d.conn
}
```

**Important behaviour:**

- All goroutines calling `Do` concurrently will **block** until the first call completes.
- If the function passed to `Do` panics, `Once` considers it done — subsequent calls will **not** re-run the function.
- `Once` cannot be reset. If you need re-initialization, use a new `Once`.

#### `sync.Cond`
A condition variable — lets goroutines wait for a specific condition to become true, then get notified.

```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []int
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Push(item int) {
    q.mu.Lock()
    q.items = append(q.items, item)
    q.mu.Unlock()
    q.cond.Signal()    // wake one waiting goroutine
}

func (q *Queue) Pop() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    for len(q.items) == 0 {
        q.cond.Wait()  // atomically releases lock + sleeps; reacquires on wake
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

`Broadcast()` wakes all waiters. `Signal()` wakes one.

**In practice:** `sync.Cond` is rarely the right choice in Go. Channels are almost always cleaner. Use `Cond` when:

- You have a condition that depends on shared state that channels can't express naturally.
- You need `Broadcast` semantics (wake all waiters at once) — like a "start signal" for a pool of workers.

#### `sync.Map`

A concurrent map optimized for two specific patterns:

1. Write-once, read-many (stable keys after initial load).
2. Each goroutine only reads/writes its own disjoint set of keys.

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
val, ok := m.Load("key")

// LoadOrStore — atomic check-and-set
actual, loaded := m.LoadOrStore("key", "default")
// loaded=true if key existed, actual=existing value
// loaded=false if stored, actual="default"

// Delete
m.Delete("key")

// Iterate
m.Range(func(k, v interface{}) bool {
    fmt.Println(k, v)
    return true  // return false to stop iteration
})
```

**When NOT to use `sync.Map`:**

- General-purpose concurrent map — a plain `map` with `sync.RWMutex` is faster for mixed read/write workloads.
- When you need `Len()` — `sync.Map` has no length operation.
- When you need type safety — everything is `interface{}` / `any`.

#### `sync.Pool`

Object pool for reusing temporarily allocated objects. Reduces GC pressure. Covered in memory section — the key `sync`-specific point:
```go
var pool = sync.Pool{
    New: func() any { return &bytes.Buffer{} },
}

func handler() {
    buf := pool.Get().(*bytes.Buffer)
    buf.Reset()          // must reset state before use
    defer pool.Put(buf)  // return to pool

    buf.WriteString("hello")
    // use buf...
}
```

**GC behaviour:** the GC can clear the pool at any GC cycle. `Pool` is a cache, not a store. Objects in the pool may disappear between any two GC cycles. Never store objects you can't afford to lose.

## `sync/atomic` - Lock-Free Operations

Atomic operations are **lock-free** — they use CPU-level instructions (CAS, XADD, etc.) that are guaranteed to complete without interruption. Faster than mutexes for simple shared scalars.

```go
import "sync/atomic"

var counter int64

// Add and return new value
atomic.AddInt64(&counter, 1)
atomic.AddInt64(&counter, -1)  // decrement

// Load and Store — guaranteed atomic read/write
val := atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, 0)

// Swap — store new value, return old
old := atomic.SwapInt64(&counter, 100)

// CompareAndSwap (CAS) — the foundation of lock-free algorithms
// If *addr == old, set *addr = new, return true
swapped := atomic.CompareAndSwapInt64(&counter, 100, 200)
```

#### `atomic.Value` — store any value atomically
```go
var config atomic.Value

// Store — value must always be the same concrete type
config.Store(Config{Timeout: 5 * time.Second})

// Load
cfg := config.Load().(Config)

// Hot-reload config without stopping the server
go func() {
    for newCfg := range configUpdates {
        config.Store(newCfg)  // all goroutines reading cfg see this atomically
    }
}()
```

**Rules for `atomic.Value`:**

- Cannot store nil.
- Must always store the same concrete type — storing a different type panics.
- Useful for read-heavy shared config, routing tables, feature flags.

#### Go 1.19+ typed atomics — `atomic.Int64`, `atomic.Bool`, etc.

Much cleaner API, same semantics:
```go
var hits atomic.Int64
var ready atomic.Bool
var ptr atomic.Pointer[Config]

hits.Add(1)
hits.Load()
hits.Store(0)
hits.Swap(100)
hits.CompareAndSwap(100, 200)

ready.Store(true)
if ready.Load() { ... }

ptr.Store(&Config{Timeout: 5 * time.Second})
cfg := ptr.Load()  // returns *Config, typed
```

#### CAS loop — the foundation of lock-free data structures
```go
// Lock-free increment (same as AddInt64, but shows the pattern)
func increment(val *int64) {
    for {
        old := atomic.LoadInt64(val)
        new := old + 1
        if atomic.CompareAndSwapInt64(val, old, new) {
            return  // succeeded — nobody changed val between Load and CAS
        }
        // Failed — another goroutine changed val. Retry.
    }
}
```

This is the ABA problem foundation: load old value → compute new → CAS. If another goroutine mutated and restored the value between Load and CAS, you might not notice. For most counters this doesn't matter. For complex data structures, it's a real concern.

#### When to use atomic vs mutex

|Scenario|Use|
|---|---|
|Single integer counter|`atomic.Int64`|
|Single bool flag|`atomic.Bool`|
|Read-heavy config reload|`atomic.Value` / `atomic.Pointer[T]`|
|Multiple fields updated together|`sync.Mutex` — atomics can't group operations|
|Complex state transitions|`sync.Mutex`|
|Performance-critical hot path|Profile first, then consider atomics|
**Critical distinction:** atomics operate on a **single memory word** atomically. If you need to update two fields atomically (e.g. update both a counter and a timestamp together), you cannot do this with atomics alone — a mutex is required, otherwise a reader can see one field updated but not the other.

```go
// WRONG — reader can see hits updated but lastHit not yet updated
var hits atomic.Int64
var lastHit atomic.Int64

hits.Add(1)
lastHit.Store(time.Now().Unix())  // gap here — not atomic together

// CORRECT — both updated under one lock
type Stats struct {
    mu      sync.Mutex
    hits    int64
    lastHit time.Time
}

func (s *Stats) Record() {
    s.mu.Lock()
    s.hits++
    s.lastHit = time.Now()
    s.mu.Unlock()
}
```

#### Memory ordering — the thing most developers skip
Atomics don't just prevent data races — they also establish **memory ordering guarantees**. In Go, all atomic operations have **sequentially consistent** ordering (the strongest guarantee).

```go
var ready atomic.Bool
var data string

// Goroutine 1
data = "result"
ready.Store(true)   // acts as a memory barrier — data write is visible before this

// Goroutine 2
for !ready.Load() { runtime.Gosched() }
fmt.Println(data)   // safe — guaranteed to see "result"
```

Without the atomic, the CPU or compiler could reorder the writes and goroutine 2 might see `ready=true` but `data=""`. The atomic store/load acts as a **memory barrier** that prevents reordering across it.

This is why `sync/atomic` is the correct way to implement a "ready" flag even if you don't care about the atomic increment — you want the memory ordering.

### How they work together — a real pattern
Here's a worker pool that uses all three packages together, the way you'd actually write it in your backend services:

```go
type WorkerPool struct {
    ctx     context.Context
    cancel  context.CancelFunc
    wg      sync.WaitGroup
    jobs    chan Job
    mu      sync.RWMutex
    results map[string]Result
    done    atomic.Bool
}

func NewWorkerPool(parent context.Context, workers int) *WorkerPool {
    ctx, cancel := context.WithCancel(parent)
    p := &WorkerPool{
        ctx:     ctx,
        cancel:  cancel,
        jobs:    make(chan Job, workers*2),
        results: make(map[string]Result),
    }
    for i := 0; i < workers; i++ {
        p.wg.Add(1)
        go p.worker()
    }
    return p
}

func (p *WorkerPool) worker() {
    defer p.wg.Done()
    for {
        select {
        case <-p.ctx.Done():
            return
        case job, ok := <-p.jobs:
            if !ok {
                return
            }
            result := process(job)
            p.mu.Lock()
            p.results[job.ID] = result
            p.mu.Unlock()
        }
    }
}

func (p *WorkerPool) Submit(job Job) bool {
    if p.done.Load() {
        return false
    }
    select {
    case p.jobs <- job:
        return true
    case <-p.ctx.Done():
        return false
    }
}

func (p *WorkerPool) Shutdown() {
    p.done.Store(true)
    p.cancel()
    p.wg.Wait()
}

func (p *WorkerPool) Result(id string) (Result, bool) {
    p.mu.RLock()
    defer p.mu.RUnlock()
    r, ok := p.results[id]
    return r, ok
}
```

- **context** — cancellation signal flows from `Shutdown()` into every blocked worker.
- **sync.WaitGroup** — `Shutdown()` blocks until every worker exits cleanly.
- **sync.RWMutex** — protects the results map with concurrent reads.
- **atomic.Bool** — fast lock-free check on the hot `Submit` path before touching channels.

### Summary

| Primitive             | Purpose                     | Key Rule                         |
| --------------------- | --------------------------- | -------------------------------- |
| `context.WithCancel`  | Manual cancellation         | Always defer cancel()            |
| `context.WithTimeout` | Time-bound operations       | Each layer narrows deadline      |
| `context.WithValue`   | Request-scoped data         | Unexported key type only         |
| `sync.Mutex`          | Exclusive access            | Never copy after first use       |
| `sync.RWMutex`        | Read-heavy shared state     | RLock for reads, Lock for writes |
| `sync.WaitGroup`      | Wait for goroutine group    | Add before goroutine starts      |
| `sync.Once`           | One-time initialization     | Panic in Do = considered done    |
| `sync.Map`            | Write-once read-many map    | Not general purpose              |
| `sync.Pool`           | Reuse allocations           | GC can clear anytime             |
| `atomic.Int64` etc.   | Single-value lock-free ops  | Can't group multiple fields      |
| `atomic.Value`        | Lock-free config swap       | Same type every Store            |
| CAS loop              | Lock-free state transitions | Retry on failure                 |
