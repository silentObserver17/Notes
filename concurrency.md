# CONCURRENCY AND SYSTEM DESIGN 

# Exercise 1 — Broken Counter

```java
class Counter {
    private int count = 0;
    public void increment() {
        count++;
    }
    public int get() {
        return count;
    }
}
```

# Your Task

Implement **3 thread-safe versions**.

Using:

synchronized

***

Using:

AtomicInteger

***

Using:

ReentrantLock



### SOLUTION:

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

class Counter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int get() {
        return count;
    }
}

class CounterSynchronized {
    private int count = 0;
    public synchronized void increment() {
        count++;
    }
    public int get() {
        return count;
    }
}

class CounterAtomic{
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }
    public int get() {
        return count.get();
    }
}

class CounterReentrant{
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        }finally {
            lock.unlock();
        }
    }

    public int get() {
        return count;
    }
}

public class BasicConcurrency {
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        CounterSynchronized cs = new CounterSynchronized();
        CounterAtomic ca = new CounterAtomic();
        CounterReentrant cr = new CounterReentrant();

        int threadCount = 100;
        int incrementsPerThread = 100;

        CountDownLatch countDownLatch = new CountDownLatch(threadCount);

        for(int i = 0; i < threadCount; i++){
            new Thread(()->{
                try{
                    for(int j = 0; j < incrementsPerThread; j++){
                        counter.increment();
                        cs.increment();
                        ca.increment();
                        cr.increment();
                    }
                }
                finally {
                    countDownLatch.countDown();
                }
            }).start();
        }

        countDownLatch.await();
        System.out.println("Normal Final count: " + counter.get());
        System.out.println("Synchronized Final count: " + cs.get());
        System.out.println("Atomic Final count: " + ca.get());
        System.out.println("Reentrant Final count: " + cr.get());
        System.out.println("Expected count: " + (threadCount * incrementsPerThread));
    }
}

/* 
OUTPUT:
Normal Final count: 9823
Synchronized Final count: 10000
Atomic Final count: 10000
Reentrant Final count: 10000
Expected count: 10000
*/

```



> **Why AtomicInteger is usually faster than synchronized?**

**How AtomicInteger Actually Works**

Internally it does something like this:

```
loop:
 oldValue = read current value
 newValue = oldValue + 1
 if CAS(memory, oldValue, newValue)
 success
 else
 retry
```

CAS (Compare-And-Swap) is a **CPU instruction**. 

Example flow:

count = 5

```
Thread A:
read 5
compute 6
CAS(5 -> 6) ✓ success
```



```
Thread B:
read 5
compute 6
CAS(5 -> 6) ✗ fails because value already changed
retry
read 6
compute 7
CAS(6 -> 7) ✓ success
```

No locks needed.

This is called **lock-free programming**.



**Why AtomicInteger is Often Faster**

###### synchronized

Uses **OS-level locking**.

Steps:

```
acquire monitor
block thread
context switch
release monitor
```

Very expensive.



###### AtomicInteger

Uses **CPU atomic instructions**.

```
CAS instruction
retry if conflict
```

No blocking.

No context switching.

Much faster under moderate contention.



###### But AtomicInteger is NOT Always Better

Under **very high contention**, CAS can spin too much.

Example:



> 1000 threads updating same variable



Then we get many CAS retries.

In such cases JVM uses optimizations like:

> LongAdder

which spreads contention across multiple counters.



##### Real Production Tip

In high throughput systems we rarely use:

> AtomicInteger

Instead we use:

> LongAdder

Example:

```java
LongAddercounter=newLongAdder();
counter.increment();
```

Used heavily inside:

> ConcurrentHashMap



## CountDownLatch

Think of a `CountDownLatch` as a **starting gate** or a **finish line barrier**. It is a synchronization utility that allows one or more threads to wait until a specific set of operations being performed in other threads completes

##### How it Works (The Mechanics)

A `CountDownLatch` is initialized with a **count** (in our case, 100).

1. **`await()`**: The main thread calls this. It effectively says, "Stop here and don't move until the count reaches zero."
2. **`countDown()`**: Each worker thread calls this right before it dies. It decrements the count by 1.
3. **The Release**: Once the 100th thread calls `countDown()`, the latch opens, and the main thread is allowed to continue.



A `CountDownLatch` is functionally the Java equivalent of a **`sync.WaitGroup`**.

| Action         | Go (sync.WaitGroup)	    | Java (CountDownLatch)                           |
| -------------- | ----------------------- | ----------------------------------------------- |
| **Initialize** | `var wg sync.WaitGroup` | `CountDownLatch latch = new CountDownLatch(n);` |
| **Increment**  | `wg.Add(1)`             | Done at initialization (usually).               |
| **Decrement**  | `wg.Done()`             | `latch.countDown()`                             |
| **Wait**       | `wg.Wait()`             | `latch.await()`                                 |



#### Important Question for You

Is `get()`**thread-safe**?

Or should it also be:

```java
public synchronized int get()
```

Explain **why**.



###### ANSWER:

> readonly will always be thread safe

❌ **This is NOT always true in Java.**

This is one of the **most subtle and important rules of the Java Memory Model**.

```java
class CounterSynchronized {
    private int count = 0;
    public synchronized void increment() {
        count++;
    }
    public int get() {
        return count;
    }
}
```



###### The Real Problem: Visibility

Even though `get()` only reads, it can still return **stale data**.

Why?

Because **there is no happens-before relationship between the write and the read**.



> Thread A

`increment()`

Inside the synchronized block:

`count++`

When the thread exits the synchronized block, the JVM **flushes changes to main memory**.

> Thread B

get()

But `get()` is **not synchronized**.

So Thread B might read from:

```
CPU cache
register
stale memory
```

It may **not see the updated value immediately**.

So the read can be stale.



###### Correct Rule

A read is only safe if there is a **happens-before relationship**.

In your case that happens if:

**Option 1 — synchronize the getter**

```java
public synchronized int get() {
returncount;
}
```

Now:

`unlock(increment) happens-before lock(get)`

So visibility is guaranteed.



**Option 2 — use&#x20;****`volatile`**

```java
private volatile int count;
```

Now every read sees the latest write.

***

**Option 3 — use&#x20;****`AtomicInteger`**

Which we already did.



###### Why our Program Still Printed Correct Values

Our program waited here:

```java
countDownLatch.await();
```

This introduces a **happens-before relationship**

When a thread calls:

`countDown()`

all its previous actions happen-before the thread that returns from `await()`.

So by the time `main()` reads the value, **all writes are visible**.

That is why your output was correct.

But if another thread called `get()`**while threads were still incrementing**, it might read stale values.



## Very Important Concurrency Principle

A variable shared across threads must be protected by **one of these mechanisms**:

1. `synchronized`
2. `volatile`
3. `Lock`
4. `Atomic`
5. `final` (special case)

Otherwise behavior is **undefined under the Java Memory Model**.



## Quick Challenge (Very Important)

```java
static boolean ready = false;
static int number = 0;
new Thread(() -> {
    number = 42;
    ready = true;
}).start();
new Thread(() -> {
    while(!ready){}
    System.out.println(number);
}).start();
```

There is **no synchronization** and **no volatile**.

So there is **no happens-before relationship**.

That means the JVM and CPU are free to:

* reorder instructions
* cache variables
* delay visibility



##### Possible Behaviors (Under the Java Memory Model)

###### Case 1 — Correct execution

Thread A:

```
number = 42
ready = true
```

Thread B:

```
ready == true
print(number) -> 42
```

Output:

> 42

This is what we usually see.

***

###### Case 2 — Instruction Reordering

The JVM may reorder:

```
ready = true
number = 42
```

Now execution becomes:

Thread A:

```
ready = true
number = 42
```

Thread B sees:

`ready == true`

But `number` is **still 0**.

Output:

> 0

This is legal according to the Java Memory Model.

***

###### Case 3 — Infinite Loop

Thread B may cache `ready`:

`ready = false`

Then it repeatedly checks its cached value:

`while(!ready) {}`

But it never reloads from main memory.

Result:

> infinite loop

This can actually happen on some CPUs.



###### The Correct Fix

Make `ready` volatile.

```java
static volatile boolean ready=false;
```

Now we create a **happens-before relationship**:

```
write ready=true
   happens-before
read ready
```

Because `number` was written **before**`ready`, it must also be visible.

Now:

```
0 cannot happen
infinite loop cannot happen
```

***

###### This Example Teaches a Deep Rule

When threads communicate via a **shared flag**, that flag must be:

```
volatile
or
synchronized
or
atomic
```

Otherwise behavior is undefined.



### What is the difference between Volatile and Atomic Integer in Java?

To understand the difference, you first have to understand the two "demons" of multithreading: **Visibility** and **Atomicity**.

* **Visibility:** Does Thread B see the change Thread A just made?
* **Atomicity:** Can a thread finish a multi-step operation (like `read-modify-write`) without another thread interfering?


##### 1. Volatile: The "Visibility" Tool

The `volatile` keyword is a lightweight flag. It tells the Java Virtual Machine (JVM): "Do not cache this variable in a CPU register; always read it from and write it to the main memory."

* **What it does:** Ensures that if Thread A changes a boolean `stopRequested` to `true`, Thread B will see that change immediately.
* **What it DOES NOT do:** It does **not** make operations atomic.

If you use `volatile int count = 0;` and call `count++`, it is still broken. Two threads can both read `0`, both increment to `1`, and both write `1` back. You lost an update.



##### 2. AtomicInteger: The "Full Package"

`AtomicInteger` provides **both** Visibility and Atomicity. It uses the `volatile` mechanics under the hood for visibility, but it adds the **Compare-And-Swap (CAS)** instruction we discussed earlier to handle the "increment" safely.

* **What it does:** Ensures that `incrementAndGet()` is treated as a single, uninterruptible unit of work at the hardware level.
* **The tradeoff:** It is a class object, so it has slightly more memory overhead than a primitive `volatile int`.



# Challenge

Implement a **blocking queue** with:

`enqueue()`

`dequeue()`

Rules:

1. If queue is **full**, producers must wait.
2. If queue is **empty**, consumers must wait.

Implement this using:

```
wait()
notifyAll()
synchronized
```



### SOLUTION:

```java
class BlockingQueue<T> {
    private Queue<T> queue = new LinkedList<>();
    private int capacity;

    public BlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public synchronized void enqueue(T item) throws InterruptedException {
        while(queue.size() == capacity){
            wait();
        }
        queue.offer(item);
        notifyAll();
    }

    public synchronized T dequeue() throws InterruptedException {
        while(queue.isEmpty()){
            wait();
        }

        T item = queue.poll();
        notifyAll();
        return item;
    }
}

// --------------- MAIN FUNCTION  -------------------------------
BlockingQueue<Integer> bq = new BlockingQueue<>(5);
new Thread(() -> {
    try {
        for (int i = 0; i < 10; i++) {
            bq.enqueue(i);
            System.out.println("Producer " + i + " enqueued");
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}).start();

new Thread(() -> {
    try {
        while(true) {
            int item = bq.dequeue();
            System.out.println("Consumer " + item + " dequeued");
        }
    }catch (InterruptedException e){
        e.printStackTrace();
    }
}).start();


```

##### In the above solution Why did we use *while* loop instead of *if* condition?

Using `if` instead of `while` leads to a dangerous bug known as a **Spurious Wakeup**.

###### 1. The "Spurious Wakeup" Problem

The Java documentation specifically warns that a thread can wake up from `wait()` for no apparent reason, even if nobody called `notify()`.

If you use an **`if`****&#x20;statement**:

1. **Thread A** calls `dequeue()`, sees the queue is empty, and hits `wait()`.
2. A **Spurious Wakeup** occurs. Thread A wakes up.
3. Because it's an `if`, it moves to the next line: `queue.poll()`.
4. **Crash!** The queue is still empty, and you get a `NoSuchElementException`.

By using a **`while`****&#x20;loop**, the thread wakes up, goes back to the top of the loop, checks `isEmpty()` again, realizes it’s still empty, and goes back to sleep safely.



###### 2. The "Early Bird" (Race Condition) Scenario

Even without "spurious" wakeups, `while` is necessary when you have multiple consumers.

Imagine this sequence:

1. **Queue is empty.**
2. **Consumer 1** calls `dequeue()` and starts `wait()`.
3. **Consumer 2** calls `dequeue()` and starts `wait()`.
4. **Producer** adds **one** item and calls `notifyAll()`.
5. **Both Consumers** wake up.
6. **Consumer 1** grabs the lock first, polls the item, and finishes.
7. **Consumer 2** now grabs the lock. If it used an `if`, it would proceed to call `poll()` on an **empty queue** because Consumer 1 already took the item!

The `while` loop forces Consumer 2 to re-verify the state of the queue before acting.



### The Golden Rule

In Java (and most low-level threading libraries), we should **never** call `wait()` outside of a loop. The idiom always looks like this:

```java
synchronized (lock) {
    while (!conditionMet) {
        lock.wait();
    }
    // Perform action
}

```



##### Why `notifyAll()` and not `notify()` ?

While `notify()` is technically "faster" (it only wakes up one thread), it is incredibly dangerous in a **Blocking Queue** because it can lead to a **Deadlock** where everyone is asleep and no one is working.

###### 1. The "Lost Signal" Problem

Imagine you have a `BlockingQueue` with a capacity of 1:

1. **Consumer A** and **Consumer B** both try to `dequeue()`. The queue is empty, so they both go to `wait()`.
2. **Producer P** calls `enqueue()`, adds an item, and calls `notify()`.
3. The JVM picks **Consumer A** to wake up.
4. **Before Consumer A can grab the lock**, **Producer P** tries to `enqueue()` again. The queue is now full, so **Producer P** goes to `wait()`.
5. **Consumer A** finally gets the lock, takes the item, and calls `notify()`.

**Here is the disaster:** If the JVM happens to pick **Consumer B** to wake up, Consumer B checks the queue, sees it is empty, and goes back to `wait()`.

**The result?Producer P** is still sleeping even though there is now space in the queue! Because we only woke up *one* thread, and that thread wasn't the one that could actually do work, the whole system grinds to a halt.



###### 2. The "Wrong Thread" Problem

In your `BlockingQueue`, the `wait()` set contains **two types of threads**:

* Producers waiting for space.
* Consumers waiting for items.

When we call `notify()`, the JVM doesn't care who it wakes up. It picks one at random.

* If a Producer finishes and calls `notify()`, it **wants** to wake up a Consumer.
* But the JVM might wake up another Producer instead. That Producer sees the queue is still full and goes back to sleep.
* The Consumers stay asleep, even though there is an item ready for them.

By using `notifyAll()`, we wake up **everyone**. The "wrong" threads will realize they can't move and go back to sleep, but the "right" thread is guaranteed to get the signal and keep the gears turning.



##### Why `wait()` Must Be Inside `synchronized`

If we try to call `wait()` without being inside a `synchronized` block, Java will immediately throw an **`IllegalMonitorStateException`**.

###### 1. The "Lost Wakeup" Disaster

The main reason is to ensure **Atomicity** between checking the condition (like `queue.isEmpty()`) and going to sleep (`wait()`).

Imagine if Java allowed `wait()` without `synchronized`:

1. **Consumer** checks `if (queue.isEmpty())`. It is true.
2. **Context Switch:** The OS pauses the Consumer right *before* it calls `wait()`.
3. **Producer** runs, adds an item to the queue, and calls `notify()`.
4. **The Tragedy:** Since the Consumer hasn't officially started "waiting" yet, the notification has nowhere to go. **The signal is lost.**
5. **Consumer** resumes and calls `wait()`. It is now sleeping on a queue that actually has an item, waiting for a `notify()` that already happened.

By requiring `synchronized`, Java guarantees that the check and the wait happen as one "atomic" move. The Producer cannot send a notification until the Consumer has officially released the lock and entered the waiting state.



###### 2. What is a "Monitor"?

In Java, every `Object` has a **Monitor** (an internal lock).

* `synchronized` means: "I am now the owner of this object's Monitor."
* `wait()` means: "I am giving up ownership of this Monitor and going to sleep. Wake me up when the Monitor is available again."

You can't "give up" ownership of something you don't own in the first place! That is why the `IllegalMonitorStateException` occurs.

`synchronized` in Java is actually a **monitor construct**, not just a lock.

A monitor includes:

```
lock
condition queue
wait set
notify mechanism
```

###

###### 3. What happens if we remove `synchronized` from the function?

If we removed the `synchronized` keyword from our `enqueue` or `dequeue` methods:

1. **Immediate Crash:** As soon as a thread hits the `wait()` line, the JVM sees the thread doesn't own the "Monitor" for that object and throws an Exception.
2. **Memory Inconsistency:** Without `synchronized`, there is no "Memory Barrier." Even if the code didn't crash, Thread A might update the queue, but Thread B might still see the old version of the queue because the changes are stuck in Thread A's CPU cache.
3. **Data Corruption:** Two threads could call `queue.offer()` at the exact same time, corrupting the internal pointers of the `LinkedList`, leading to a broken data structure.



###### The Workflow of `wait()`

When a thread calls `wait()` inside a synchronized block, it does something very special:

* It **releases the lock** atomically.
* It goes to the "Wait Set."
* When it wakes up (after a `notify`), it **automatically tries to re-acquire the lock** before continuing.



### Build a Thread Pool from Scratch

Architecture:

```
Client submits task
        |
        v
   Task Queue (BlockingQueue)
        |
        v
   Worker Threads
        |
        v
   Execute Runnable
```



##### SOLUTION:

```java
class SimpleThreadPool{
  private final Queue<Runnable> workerQueue = new LinkedList<>();
    private final List<Worker> workers = new ArrayList<>();
    private final int capacity = 100; // internal queue limit

    private boolean isShutdown = false;

    public SimpleThreadPool(int workerCount) {
        for(int i = 0; i < workerCount; i++){
            Worker worker = new Worker("worker-" + i);
            workers.add(worker);
            worker.start();
        }
    }

    public synchronized void submit(Runnable task) throws InterruptedException {
        if (isShutdown){
            throw new IllegalStateException("Cannot submit task: pool is already shut down");
        }

        while(workQueue.size() == capacity){
            wait();
        }
        workQueue.offer(task);
        notifyAll();
    }

    public synchronized void shutdown(){
        isShutdown = true;
        notifyAll();
    }

    public void awaitTermination() throws InterruptedException {
        for (Worker worker : workers) {
            worker.join(); // wait for each worker thread to fully exit
        }
    }

    class Worker extends Thread{
        Worker(String name) {
            super(name);
        }

        @Override
        public void run() {
            while(true) {
                Runnable task = null;

                synchronized (SimpleThreadPool.this) {
                    while (workQueue.isEmpty() && !isShutdown) {
                        try {
                            SimpleThreadPool.this.wait();
                        } catch (InterruptedException e) {
                            return;
                        }
                    }

                    if (isShutdown && workQueue.isEmpty()) {
                        return;
                    }

                    task = workQueue.poll();
                    SimpleThreadPool.this.notifyAll();
                }

                if(task != null) {
                    task.run();
                }
            }
        }
    }
}

  

  public static void main(String[] args) throws InterruptedException {
      SimpleThreadPool simpleThreadPool = new SimpleThreadPool(3);

      // Submit 10 Tasks.
      for(int i = 1; i <= 10; i++){
          final int taskId = i;
          simpleThreadPool.submit(() -> {
              String name =  Thread.currentThread().getName();
              System.out.println("Task " + taskId + " started by " + name);
              // simulate Work
              try{
                  Thread.sleep(500);
              } catch (Exception e) {
                  System.out.println(name + " interrupted.");
              }
          });
      }

      // 3. Initiate shutdown
      System.out.println("--- Initiating Shutdown ---");
      simpleThreadPool.shutdown();
      simpleThreadPool.awaitTermination(); // ← add this!
      System.out.println("All tasks done.");
  }

```

**The&#x20;****`main`****&#x20;thread is the producer, and those 3&#x20;****`Worker`****&#x20;threads are the consumers, constantly watching the queue.**



###### 1. The Submission (The Producer)

When we call `submit(task)`, the `main` thread acts as the Producer.

* It checks if there is room in the `workQueue` (using the `capacity` check).
* If the queue is full, it calls `wait()` and goes to sleep.
* Once it puts a task in the queue, it calls `notifyAll()` to wake up the **Workers**.

###### 2. The Worker Loop (The Consumer)

The `Worker` threads are the Consumers. They sit in a `while(true)` loop.

* **The Wait:** If the queue is empty, they call `wait()`.
* **The Handover:** As soon as a task is submitted, one worker wakes up, grabs the `task` from the queue, and immediately **exits the synchronized block**.
* **The Execution:** The line `task.run()` happens *outside* the lock. This is the "secret sauce" that allows Worker 1, Worker 2, and Worker 3 to all run tasks at the exact same time.



###### 1. Why `notifyAll()` in `shutdown()`?

If a worker thread is currently "idle" (stuck in `wait()` because the queue is empty), it will stay there forever unless someone wakes it up. By calling `notifyAll()` inside `shutdown()`, we kick every worker back into the `while` loop so they can check the `isShutdown` flag and return.



###### 2. Executing `task.run()` outside `synchronized`

This is a critical performance detail. If we ran the task *inside* the `synchronized` block, only one worker could execute a task at a time, effectively turning our "thread pool" into a single-threaded slowpoke. By grabbing the task and then releasing the lock, multiple workers can run their tasks in parallel.



###### 3. Graceful Exit

Notice the condition: `if (isShutdown && taskQueue.isEmpty())`. This ensures that if we call `shutdown()`, the workers will still finish whatever tasks are already sitting in the queue before they die.



###### The Magic of the Lambda `() -> { ... }`

When we write `() -> { ... }`, the Java compiler looks at the `submit(Runnable task)` method signature. It says: *"Okay, the user is giving me a block of code. Does it match the signature of the&#x20;**`Runnable`**&#x20;interface?"*

* `()`: This matches the empty parameters of the `run()` method.
* `->`: This is the bridge.
* `{ ... }`: This becomes the **body** of the `run()` method.

The compiler automatically "wraps" our code into a `Runnable` object behind the scenes. This is called **Target Typing**.



## Executors and Thread Pools

#### Why Executors Exist

Before Java 5, people created threads like this:



```java
Threadt=newThread(() -> doWork());
t.start();
```

Problems with this approach:

1. **Thread creation is expensive**
2. No reuse of threads
3. Hard to limit number of threads
4. Hard to manage lifecycle

So Java introduced the **Executor Framework** in **Java 5 (****`java.util.concurrent`****)**.



#### Executor Interface (The Simplest Layer)

Java started with a very simple abstraction.

```java
public interface Executor {
 void execute(Runnablecommand);
}
```

Example:

```java
Executor executor = new Executor() {
    public void execute(Runnable r) {
        new Thread(r).start();
    }
};
executor.execute(() -> System.out.println("Running task"));
```

But this simple executor **creates a new thread every time**, which defeats the purpose.

So Java provides better implementations.

#####

##### ExecutorService (Real Executor)

Most real-world code uses **ExecutorService**.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
```

This creates a **thread pool with 4 threads**.

```java
executor.submit(() -> {
  System.out.println(Thread.currentThread().getName());
});
```

| Method          | Purpose                    |
| --------------- | -------------------------- |
| `execute()`     | run task                   |
| `submit()`      | run task and return result |
| `shutdown()`    | graceful shutdown          |
| `shutdownNow()` | force shutdown             |
| `invokeAll()`   | run multiple tasks         |

##### Runnable vs Callable

Executors support **two types of tasks**.

###### Runnable (no return)

```java
Runnable task = () -> {
    System.out.println("Running");
};
```

Submit:

```java
executor.execute(task);
```

###### Callable (returns result)

```java
Callable<Integer> task= () -> {
  return 42;
};
```

Submit:

```java
Future result = executor.submit(task);
```

Get result:

```java
System.out.println(result.get());
```

So the flow becomes:

`Callable → ExecutorService → Future → Result`



##### Future (Result Placeholder)

A **Future** represents a result that will be available later.

```java
Future<Integer> future=executor.submit(() -> 10+20);
```

Retrieve:

```java
Integer result = future.get();
```

| Method     | Meaning          |
| ---------- | ---------------- |
| `get()`    | wait for result  |
| `isDone()` | check completion |
| `cancel()` | cancel task      |



##### Executors Utility Class

Java provides factory methods via **Executors** class.



###### Fixed Thread Pool

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
```

```
Threads: Fixed
Queue: Unlimited
```

Use when:

* predictable load
* CPU-bound tasks



###### Cached Thread Pool

```java
ExecutorService executor = Executors.newCachedThreadPool();
```

```

Threads: unlimited
Idle threads destroyed after 60s
```

Use when:

* many short-lived tasks

Danger:

⚠️ Can create **too many threads**.

##### &#xA;Single Thread Executor

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```

Only **one thread executes tasks sequentially**.

Guarantees **ordering**.

##### &#xA;Scheduled Thread Pool

```java
ScheduledExecutorService executor = Executors.newScheduledThreadPool(2);
```

Supports **delayed or periodic tasks**.

Example:

```java
executor.schedule(() -> {
    System.out.println("Delayed task");
}, 3, TimeUnit.SECONDS);
```

##### Internal Architecture (Very Important)

Under the hood most executors use **ThreadPoolExecutor**.

```
Task Submitted
      ↓
Task Queue
      ↓
ThreadPoolExecutor
      ↓
Worker Threads
```

| Component      | Purpose             |
| -------------- | ------------------- |
| Core threads   | minimum threads     |
| Max threads    | upper limit         |
| Queue          | holds waiting tasks |
| Worker threads | execute tasks       |

Executors factory methods are just **shortcuts**.

Example:

```java
Executors.newFixedThreadPool(4)
```

Actually creates:

```java
new ThreadPoolExecutor(
    4,
    4,
    0L,
    TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue()
);
```

##### Example (Full Flow)

```java
import java.util.concurrent.*;
public class ExecutorExample {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        Future future = executor.submit(() -> {
            Thread.sleep(1000);
            return 42;
        });
        System.out.println("Waiting for result...");
        Integer result = future.get();
        System.out.println("Result: " + result);
        executor.shutdown();
    }
}
```



Executors solve **3 major concurrency problems**:

1️⃣ **Thread reuse**

Thread creation is expensive



2️⃣ **Concurrency control**

limit number of threads



3️⃣ **Task abstraction**

Submit tasks instead of managing threads

***

## 1. Why `Executors.newFixedThreadPool()` is Dangerous

When you write:

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
```

Under the hood Java creates:

```java
new ThreadPoolExecutor(
    4, 
    4,
    0L,
    TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>()
);
```

The critical part is:

```java
LinkedBlockingQueue<Runnable>()
```

This queue is **unbounded**.

Meaning:

```
Tasks keep getting queued forever
```

Example failure scenario:

```
incoming requests = 10,000/sec
processing capacity = 500/sec
```

Result:

```
Queue grows infinitely
Memory keeps increasing
Eventually → OutOfMemoryError
```

That’s why **production systems rarely use&#x20;****`Executors`****&#x20;factory methods**.

Instead people create **ThreadPoolExecutor directly**.

***

### 2. ThreadPoolExecutor Core Parameters

Constructor:

```java
ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue
)

```

Let’s break each part.

***

### corePoolSize

Minimum number of threads **always kept alive**.

Example:

```
corePoolSize = 4
```

Even if idle:

```
4 worker threads exist
```

***

### maximumPoolSize

Maximum allowed threads.

```
maxPoolSize = 10
```

Executor can grow **up to 10 threads** under heavy load.

***

### keepAliveTime

Idle threads above core size are destroyed after this time.

Example:

```
core = 4
max = 10
```

Threads 5–10 will die after `keepAliveTime`.

***

### workQueue

Queue holding waiting tasks.

Types:

#### LinkedBlockingQueue

```
Unbounded queue
```

Risk: memory growth.

***

#### ArrayBlockingQueue

```
Bounded queue
```

Example:

```
capacity = 1000
```

Safer for production.

***

##### SynchronousQueue

```
No queue
Task must immediately find a thread
```

Used in **CachedThreadPool**.

***

## 3. How ThreadPoolExecutor Actually Works

When a task arrives:

```
submit(task)
```

Executor follows this **4-step algorithm**.

***

#### Step 1 — Create core threads

If:

```
running threads < corePoolSize
```

Create new thread.

```
Task → new worker thread
```

***

#### Step 2 — Put in queue

If:

```
running threads ≥ corePoolSize
```

Task goes to **queue**.

```
Task → workQueue
```

***

#### Step 3 — Create extra threads

If queue is **full**:

```
running threads < maxPoolSize
```

Create extra thread.

***

#### Step 4 — Reject task

If:

```
queue full
AND
threads == maxPoolSize
```

Executor **rejects task**.

***

## 4. Rejection Policies

Executor uses **RejectedExecutionHandler**.

Common policies:

#### AbortPolicy (default)

Throws exception.

```
RejectedExecutionException
```

***

#### CallerRunsPolicy

Caller thread runs task.

```
main thread executes task
```

Good for **backpressure**.

***

#### DiscardPolicy

Task silently dropped.

***

#### DiscardOldestPolicy

Removes oldest queued task.

***

## 5. Visual Execution Flow

```
           Task Submitted
                 │
                 ▼
        runningThreads < corePoolSize ?
               │
         YES ──┘──► create thread
               │
               NO
               ▼
            queue full ?
               │
           NO ─┘──► enqueue task
               │
               YES
               ▼
      runningThreads < maxPoolSize ?
               │
           YES ─┘──► create extra thread
               │
               NO
               ▼
           reject task

```

Understanding this flow = **you understand executors**.

***

## 6. Real Production Thread Pool

Example configuration:

```java
ExecutorService executor =
    new ThreadPoolExecutor(
        4,
        10,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(100),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );

```

Behavior:

```
4 core threads
queue size = 100
max threads = 10
extra threads die after 60s
```

This prevents **memory explosion**.

***

## Question 1

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

for(int i=0;i<100;i++){
    executor.submit(() -> {
        while(true){}
    });
}

```

Your answer:

> infinite loop of two threads

✅ **Mostly correct**, but let's analyze deeper.

#### Step-by-step execution

Thread pool configuration:

```
corePoolSize = 2
maxPoolSize = 2
queue = LinkedBlockingQueue (unbounded)
```

Execution flow:

1️⃣ First task submitted

→ **Thread-1 created**

→ runs `while(true){}` forever

2️⃣ Second task submitted

→ **Thread-2 created**

→ runs `while(true){}` forever

3️⃣ Remaining **98 tasks**

```
runningThreads == corePoolSize
```

So executor **queues them**.

```
queue size = 98

```

But:

```
Thread-1 → infinite loop
Thread-2 → infinite loop
```

They **never finish**, so queued tasks are **never executed**.

***

#### Final result

```
CPU usage = 2 cores at 100%
Queue = grows to 98 tasks
Thread starvation = YES
Program never finishes
```

This is called:

```
Thread starvation
```

because tasks are **waiting forever**.

So the **full correct answer** is:

> Two threads run infinite loops consuming CPU, remaining tasks stay queued forever.

***

## Question 2

```java
ExecutorService executor =
    Executors.newSingleThreadExecutor();

for(int i=0;i<5;i++){
    executor.submit(() -> {
        System.out.println(Thread.currentThread().getName());
    });
}

```

Your answer:

> single thread prints 5 times

✅ Correct.

But here’s the **key insight** interviewers want:

```
SingleThreadExecutor guarantees task ordering
```

Execution:

```
Task1 → Task2 → Task3 → Task4 → Task5
```

All executed by **same worker thread**.

Example output:

```
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
pool-1-thread-1
```

Important guarantee:

```
FIFO execution
```

***

## Question 3

What happens if we forget:

```
executor.shutdown();
```

Your answer:

> infinite loop and program won't end

🟡 **Almost correct but needs precision.**

What actually happens:

```
Executor threads are NON-DAEMON threads
```

Meaning:

```
JVM does not exit while they are alive
```

Even if all tasks finish:

```
worker threads remain alive
```

So:

```
Program appears "stuck"
```

Example:

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

executor.submit(() -> System.out.println("Hello"));
```

Output:

```
Hello
```

But program **does not terminate**.

Because pool threads are waiting for more tasks.

Correct explanation:

> JVM does not exit because executor threads remain alive.

***

### Question 4

Difference between:

```java
execute()
submit()
```

Your answer:

> execute runs code but submit returns result

✅ Correct idea. Let's make it **precise**.

| Feature            | execute() | submit()         |
| ------------------ | --------- | ---------------- |
| Return value       | void      | Future           |
| Runnable support   | yes       | yes              |
| Callable support   | no        | yes              |
| Exception handling | uncaught  | stored in Future |

***

### execute()

```java
executor.execute(() -> {
    throw new RuntimeException();
});

```

Result:

```
Exception printed to console
```

***

### submit()

```java
Future<?> f = executor.submit(() -> {
    throw new RuntimeException();
});
```

No exception printed.

Instead:

```java
f.get()
```

throws:

```
ExecutionException
```

This is a **very common interview trap**.

***

# Question 5

```java
Executors.newCachedThreadPool()
```

Your answer:

> unlimited threads → system crash

✅ Correct.

But let's explain **why**.

Internally:

```java
corePoolSize = 0
maxPoolSize = Integer.MAX_VALUE
queue = SynchronousQueue
```

Important detail:

```
SynchronousQueue has NO capacity
```

Meaning:

```
task must find a thread immediately
```

If no idle thread exists:

```
new thread created
```

Example disaster scenario:

```
incoming requests = 50,000/sec
task duration = 1 sec
```

Executor will create:

```
50,000 threads
```

Result:

```
context switching explosion
memory exhaustion
CPU thrashing
system crash
```

***

### Next Exercise (Much Harder)

Let’s test **deep executor understanding**.

What happens here?

```java
ThreadPoolExecutor executor =
    new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(2)
    );

for(int i=0;i<10;i++){
    int task = i;

    executor.submit(() -> {
        Thread.sleep(10000);
        System.out.println(task);
        return null;
    });
}
```

First, here is the pool configuration again:

```java
ThreadPoolExecutor executor =
    new ThreadPoolExecutor(
        2,                // corePoolSize
        4,                // maximumPoolSize
        60,
        TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(2) // queue capacity = 2
    );
```

Key parameters:

```
corePoolSize = 2
maxPoolSize = 4
queue capacity = 2
```

Tasks submitted:

```
10 tasks
each sleeps for 10 seconds
```

***

#### The Actual Executor Algorithm

When a task arrives, Java follows this **strict order**:

1️⃣ If `runningThreads < corePoolSize` → **create new thread**

2️⃣ Else → **try to enqueue task**

3️⃣ If queue is **full** → **create new thread (if < maxPoolSize)**

4️⃣ If threads == maxPoolSize → **reject task**

This order is extremely important.

***

# Step-by-Step Execution

### Task 0

```
threads = 0 < core(2)
```

Create **Thread-1**

```
Thread-1 running task0
```

***

### Task 1

```
threads = 1 < core(2)
```

Create **Thread-2**

```
Thread-2 running task1
```

***

### Task 2

Now:

```
threads = 2 (core reached)
```

Executor tries **queue**.

```
Queue: [task2]
```

***

### Task 3

Queue still has space.

```
Queue: [task2, task3]
```

Queue is now **FULL**.

***

### Task 4

Queue full → check threads.

```
threads = 2
max = 4
```

So create **Thread-3**

```
Thread-3 running task4
```

***

### Task 5

Queue still full → thread creation allowed.

Create **Thread-4**

```
Thread-4 running task5
```

Now:

```
threads = 4 (max reached)
queue = full
```

***

### Task 6

Executor tries:

```
queue full
threads == max
```

➡ **REJECTED**

Throws:

```
RejectedExecutionException
```

The same happens for:

```
task6
task7
task8
task9
```

***

# Final State

Threads created:

```
4 threads
```

Running tasks:

```
task0
task1
task4
task5
```

Queued tasks:

```
task2
task3
```

Rejected tasks:

```
task6
task7
task8
task9
```

***

# Important Correction in Your Answer

You said:

> thread 2 and 3 will go into queue

Correct.

But then:

> it will create thread 4 and 5

❌ Slight mistake.

ThreadPoolExecutor will create **only up to maxPoolSize threads total**, not additional ones.

So:

```
Thread-1
Thread-2
Thread-3
Thread-4
```

Total = **4 threads**.

***

# Timeline Visualization

```
Submit task0 → Thread1
Submit task1 → Thread2

Submit task2 → Queue
Submit task3 → Queue

Submit task4 → Thread3
Submit task5 → Thread4

Submit task6 → REJECT
Submit task7 → REJECT
Submit task8 → REJECT
Submit task9 → REJECT
```

***

# Interview Insight

Most developers assume the order is:

```
threads → max → queue
```

But the real order is:

```
core threads → queue → max threads
```

That detail is why this question shows up in **senior Java interviews**.

***

# One Small Brain-Twister (very common)

What changes if we use:

```
new LinkedBlockingQueue<>()
```

instead of:

```
new ArrayBlockingQueue<>(2)
```

in the same thread pool?

Would the pool ever create **Thread-3 and Thread-4**?



We change only this:

```java
new ArrayBlockingQueue<>(2)
```

to

```java
new LinkedBlockingQueue<>()
```

Everything else stays the same:

```
corePoolSize = 2
maximumPoolSize = 4
queue = LinkedBlockingQueue (unbounded)
```

***

# Step-by-Step Execution

### Task 0

```
threads = 0 < core(2)
```

Create **Thread-1**

```
Thread-1 → task0 (sleep 10s)
```

***

### Task 1

```
threads = 1 < core(2)
```

Create **Thread-2**

```
Thread-2 → task1 (sleep 10s)
```

***

### Task 2

Now:

```
threads = 2 == corePoolSize
```

Executor tries to **enqueue** the task.

Queue has **unlimited capacity**, so it succeeds.

```
Queue: [task2]
```

***

### Task 3

```
Queue: [task2, task3]
```

***

### Task 4

```
Queue: [task2, task3, task4]
```

***

### Tasks 5–9

All of them go to the queue.

Final queue:

```
[task2, task3, task4, task5, task6, task7, task8, task9]
```

***

# Final State

Threads created:

```
Thread-1
Thread-2
```

Queue:

```
8 tasks waiting
```

Execution flow:

```
Thread-1 finishes task0 → takes task2
Thread-2 finishes task1 → takes task3
```

And so on.

***

# Key Insight (VERY Important)

**ThreadPoolExecutor will NOT create threads beyond&#x20;****`corePoolSize`****&#x20;if the queue never fills.**

Because the algorithm is:

```
1. Fill core threads
2. Queue tasks
3. If queue full → create extra threads
```

But with an **unbounded queue**:

```
queue never becomes full
```

Therefore:

```
threads never grow beyond corePoolSize
```

So in this example:

```
maxPoolSize = 4
```

is **completely useless**.

The pool will **never create Thread-3 or Thread-4**.

***

# Why This Is Dangerous

If tasks are slow:

```
incoming tasks = 1000/sec
processing capacity = 2/sec
```

Queue becomes:

```
1M
5M
20M
...

```

Eventually:

```
OutOfMemoryError
```

This is **exactly why&#x20;****`Executors.newFixedThreadPool()`****&#x20;is considered dangerous**.

Because internally it uses:

```
LinkedBlockingQueue (unbounded)
```

***

# The Production Fix

Instead of:

```java
Executors.newFixedThreadPool(10);
```

Senior engineers usually do:

```java
new ThreadPoolExecutor(
    10,
    20,
    60,
    TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000)
);

```

This gives **bounded memory**.

***

# One Final Interview Trap (Very Famous)

What happens here?

```java
ThreadPoolExecutor executor =
    new ThreadPoolExecutor(
        0,
        4,
        60,
        TimeUnit.SECONDS,
        new SynchronousQueue<>()
    );

```

Tasks:

```java
for(int i=0;i<5;i++){
    executor.submit(() -> {
        Thread.sleep(10000);
        return null;
    });
}

```

Questions:

1️⃣ How many threads will be created?

2️⃣ Will any task go to queue?

3️⃣ Will rejection happen?

This scenario actually reveals **how&#x20;****`CachedThreadPool`****&#x20;works internally**, and understanding it unlocks one of the most interesting executor behaviors.

The two key observations you already made are exactly the ones that matter:

* `corePoolSize = 0`
* `SynchronousQueue` has **no capacity**

Let’s finish the reasoning step-by-step.

***

# Pool Configuration

```java
ThreadPoolExecutor executor =
    new ThreadPoolExecutor(
        0,                 // corePoolSize
        4,                 // maxPoolSize
        60,
        TimeUnit.SECONDS,
        new SynchronousQueue<>()
    );
```

Key properties:

```
corePoolSize = 0
maxPoolSize = 4
queue = SynchronousQueue
```

Important rule about **SynchronousQueue**:

```
It cannot store tasks.
A task must be handed directly to a worker thread.
```

So there is **no buffering**.

***

# Executor Algorithm Reminder

When a task arrives:

1. If `threads < corePoolSize` → create thread
2. Else try to **enqueue task**
3. If queue full → create thread (if `< maxPoolSize`)
4. Else → reject

***

# Step-by-Step Execution

### Task 0

```
threads = 0
corePoolSize = 0
```

Condition:

```
threads < corePoolSize → false
```

So executor tries to **enqueue**.

But:

```
SynchronousQueue cannot hold tasks
```

So enqueue fails immediately.

Next step:

```
threads < maxPoolSize → true
```

Create **Thread-1**

Thread-1 runs task0.

***

### Task 1

Same process:

* cannot enqueue
* threads < max

Create **Thread-2**

***

### Task 2

Create **Thread-3**

***

### Task 3

Create **Thread-4**

***

### Task 4

Now:

```
threads = 4
maxPoolSize = 4
```

Executor tries:

1. enqueue → fails (SynchronousQueue)
2. create thread → not allowed (max reached)

Therefore:

```
RejectedExecutionException
```

***

# Final Result

Threads created:

```
Thread-1
Thread-2
Thread-3
Thread-4
```

Running tasks:

```
task0
task1
task2
task3
```

Rejected:

```
task4
```

***

# Answers to the Questions

### 1️⃣ How many threads are created?

```
4 threads
```

***

### 2️⃣ Will any task go to queue?

```
No
```

Because:

```
SynchronousQueue cannot store tasks
```

***

### 3️⃣ Will rejection happen?

Yes.

```
task4 → rejected
```

***

# Why This Design Exists

This configuration is almost the same idea behind:

```
Executors.newCachedThreadPool()
```

Internally it uses:

```
corePoolSize = 0
maxPoolSize = Integer.MAX_VALUE
queue = SynchronousQueue
```

Meaning:

```
If a thread is free → reuse it
Otherwise → create new thread
```

That’s why cached pools **scale threads aggressively**.

***

# One Very Interesting Insight

With a `SynchronousQueue`, the pool behaves like this:

```
No queue
Direct handoff
Thread-per-task (until max)
```

So the execution model becomes:

```
Task → Thread
Task → Thread
Task → Thread
```

instead of

```
Task → Queue → Thread
```

***

# Quick Challenge (Very Good Mental Exercise)

Predict what happens here:

```java
ThreadPoolExecutor executor =
    new ThreadPoolExecutor(
        2,
        4,
        60,
        TimeUnit.SECONDS,
        new SynchronousQueue<>()
    );

for(int i = 0; i < 6; i++) {
    executor.submit(() -> {
        Thread.sleep(10000);
        return null;
    });
}

```

Try to determine:

1️⃣ How many threads are created

2️⃣ Which task gets rejected

3️⃣ Does the queue ever get used

This one forces you to combine **core threads + synchronous queue**, which is a **classic executor interview puzzle**.

You said:

> 2 threads created → queue can't hold → create thread3 & thread4 → rest rejected

✅ **100% correct reasoning.**

This shows you understand the **three key executor behaviors**:

* **core thread creation**
* **queue behavior**
* **max thread expansion**



# Common Atomic Classes

Java provides many atomic types.

### Numbers

```java
AtomicInteger
AtomicLong
AtomicDouble (vialibraries)
```

### References

```java
AtomicReference<T>
```

### Arrays

```java
AtomicIntegerArray
```

### Field Updaters

```java
AtomicIntegerFieldUpdater
```

### Advanced

```java
LongAdder
LongAccumulator
```

# Important Limitation: ABA Problem

CAS compares **value only**, not history.

Example:

```
Thread A reads value = A
Thread B changes A → B → A
Thread A performs CAS
```

CAS succeeds even though the value changed in between.

This is the **ABA problem**.

Java solves this using:

```java
AtomicStampedReference
AtomicMarkableReference
```

***

`AtomicInteger` guarantees **atomicity of the increment operation**, but it **does NOT guarantee when threads run**.

So the issue here is **not a race condition**, but **thread scheduling**.

We have:

```java
AtomicInteger x = new AtomicInteger(0);

Thread t1 = new Thread(() -> {
    x.incrementAndGet();
});

Thread t2 = new Thread(() -> {
    x.incrementAndGet();
});

t1.start();
t2.start();

System.out.println(x.get());

```

***

### The Key Idea

`AtomicInteger` guarantees **atomicity of the increment operation**, but it **does NOT guarantee when threads run**.

So the issue here is **not a race condition**, but **thread scheduling**.

***

### Possible Execution Orders

##### Case 1 — Main thread prints immediately

```
main: print x (0)
t1: increment
t2: increment
```

Output:

```
0
```

Because both worker threads haven't run yet.

***

### Case 2 — One thread runs first

```
t1: increment → x = 1
main: print
t2: increment
```

Output:

```
1
```

***

### Case 3 — Both threads finish first

```
t1: increment
t2: increment
main: print
```

Output:

```
2
```

***

# Possible Outputs

You correctly said:

```
0
1
2
```

✅ All are possible.

***

# Important Insight

Even though `AtomicInteger` ensures:

```
incrementAndGet() is thread-safe
```

It **does not ensure completion order**.

Atomicity guarantees:

```
No lost updates
```

But it **does not guarantee timing**.

***

# How to Guarantee Output = 2

We must **wait for both threads**.

### Option 1 — `join()` (simplest)

```java
t1.start();
t2.start();

t1.join();
t2.join();

System.out.println(x.get());

```

Now output is always:

```
2
```

***

### Option 2 — `CountDownLatch`

```java
CountDownLatch latch = new CountDownLatch(2);

Thread t1 = new Thread(() -> {
    x.incrementAndGet();
    latch.countDown();
});

Thread t2 = new Thread(() -> {
    x.incrementAndGet();
    latch.countDown();
});

t1.start();
t2.start();

latch.await();

System.out.println(x.get());

```

***

### Option 3 — ExecutorService

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

executor.submit(() -> x.incrementAndGet());
executor.submit(() -> x.incrementAndGet());

executor.shutdown();
executor.awaitTermination(1, TimeUnit.SECONDS);

System.out.println(x.get());

```

***

# Important Concurrency Lesson

This example teaches the difference between:

| Concept    | Meaning                                   |
| ---------- | ----------------------------------------- |
| Atomicity  | Operation happens without race conditions |
| Ordering   | Which thread runs first                   |
| Completion | When threads finish                       |

`AtomicInteger` solves:

```
atomicity
```

But not:

```
thread completion
```

***

# One More Interesting Twist

What happens if we replace `AtomicInteger` with:

```
int x = 0;
```

and inside threads do:

```
x++;
```

Then even **after waiting for both threads**, the result could be:

```
1
```

Because:

```
x++ is not atomic
```

That contrast is exactly **why atomic variables exist**.

***

# LOCKS AND REENTRANT LOCKS 

We’ll build the picture in this order:

1. Why Java introduced the **Lock API**
2. The **`Lock`****&#x20;interface**
3. **`ReentrantLock`** (core implementation)
4. **Fair vs Non-fair locks**
5. Important methods (`tryLock`, `lockInterruptibly`, etc.)
6. **Condition variables**
7. When to use **locks vs synchronized**

***

### 1. Why Java Introduced Explicit Locks

Before Java 5, synchronization was done with:

```java
synchronized(lockObject) {
    // critical section
}

```

or

```java
public synchronized void increment() {
    count++;
}

```

This works well but has **limitations**:

| Problem                         | Explanation           |
| ------------------------------- | --------------------- |
| Cannot try acquiring lock       | Thread must wait      |
| Cannot interrupt lock wait      | `synchronized` blocks |
| Cannot implement fairness       | No queue control      |
| One condition queue per monitor | Limited coordination  |

So Java introduced the **Lock framework** in `java.util.concurrent.locks`.

***

### 2. The `Lock` Interface

The `Lock` interface gives **manual control** over locking.

Example:

```java
Lock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}

```

Important: **always unlock in&#x20;****`finally`**.

Why?

```java
lock.lock();
criticalWork();
lock.unlock();
```

If `criticalWork()` throws an exception:

```
lock never released
deadlock
```

The safe pattern:

```java
lock.lock();
try {
    criticalWork();
} finally {
    lock.unlock();
}
```

***

### 3. First Example

Let’s recreate the counter example with `ReentrantLock`.

```java
import java.util.concurrent.locks.*;

class Counter {

    private int count = 0;
    private final Lock lock = new ReentrantLock();

    public void increment() {
        lock.lock();

        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    public int get() {
        return count;
    }
}

```

Multiple threads can call `increment()` safely.

***

### 4. What Does "Reentrant" Mean?

Reentrant = **same thread can acquire the same lock multiple times**.

Example:

```java
Lock lock = new ReentrantLock();

lock.lock();
lock.lock();
lock.lock();

lock.unlock();
lock.unlock();
lock.unlock();

```

This works.

The lock keeps an **internal hold count**.

```
Thread A acquires lock → count = 1
Thread A acquires again → count = 2
Thread A acquires again → count = 3
```

Only when count becomes **0** does the lock release.

This behavior is the same as Java’s **intrinsic monitor locks** (`synchronized`).

***

### 5. Fair vs Non-Fair Locks

You can create a fair lock:

```java
Lock lock = new ReentrantLock(true);
```

Default:

```java
new ReentrantLock()
```

is **non-fair**.

##### Non-Fair Lock (default)

Threads can **jump the queue**.

```
Thread1 waiting
Thread2 waiting
Thread3 arrives → gets lock first
```

This improves **throughput**.

***

##### Fair Lock

First-come-first-serve:

```
Thread1 → lock
Thread2 → next
Thread3 → next
```

Fairness reduces starvation but can reduce performance.

***

### 6. `tryLock(timeout)` (Very Important)

wait but not forever

```java
if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try {
        // Got lock within 2 seconds
    } finally {
        lock.unlock();
    }
} else {
    // 2 seconds passed, still couldn't get it
    System.out.println("Timed out");
}

```

This is extremely useful for **deadlock avoidance**.

***

## 7. `tryLock()` in ReentrantLock

Instead of **blocking** like `lock()`, `tryLock()` attempts to acquire the lock and **immediately returns a boolean** — `true` if it got the lock, `false` if not.

###### Basic `tryLock()`

```java
ReentrantLock lock = new ReentrantLock();

if (lock.tryLock()) {
    try {
        // Got the lock, do work
    } finally {
        lock.unlock();
    }
} else {
    // Couldn't get lock, do something else
    System.out.println("Lock busy, skipping...");
}
```

No waiting at all — if the lock is taken, it moves on instantly.


vs `lock()` comparison

|                 | `lock()`           | `tryLock()`                 | `tryLock(timeout)`                |
| --------------- | ------------------ | --------------------------- | --------------------------------- |
| If lock is busy | Blocks forever     | Returns `false` immediately | Waits up to timeout               |
| Return type     | `void`             | `boolean`                   | `boolean`                         |
| Use case        | Must have the lock | Skip if busy                | Retry logic / deadlock prevention |

##### Key use case — **deadlock prevention**

```java
// Two threads needing two locks — classic deadlock scenario
boolean gotBoth = false;
if (lock1.tryLock()) {
    try {
        if (lock2.tryLock()) {
            try {
                // Got both locks safely
                gotBoth = true;
            } finally {
                lock2.unlock();
            }
        }
    } finally {
        lock1.unlock();
    }
}
if (!gotBoth) {
    // Back off and retry later instead of deadlocking
}
```

With `lock()`, if Thread A holds `lock1` and waits for `lock2`, while Thread B holds `lock2` and waits for `lock1` — **deadlock**. `tryLock()` lets you back off instead of waiting forever.

***

## 8. `lockInterruptibly()`

Normally:

```
lock.lock();

```

If a thread waits for the lock, it **cannot be interrupted**.

But with:

```java
lock.lockInterruptibly();
```

the thread **can be interrupted** while waiting.

Example:

```java
try {
    lock.lockInterruptibly();
} catch (InterruptedException e) {
    System.out.println("Interrupted while waiting");
}

```

This is **very useful in responsive systems**.

***

## 9. Condition Variables

This is where `ReentrantLock` becomes **much more powerful than synchronized**.

With `synchronized`, we have:

```java
wait()
notify()
notifyAll()

```

But **only one wait queue exists**.

With locks we can create **multiple condition queues**.

Example:

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

Threads can:

```java
condition.await();
condition.signal();
```



***

## Example: Producer Consumer

```java
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();
Condition notEmpty = lock.newCondition();

```

Producer waits:

```java
notFull.await();
```

Consumer signals:

```java
notFull.signal();
```

Multiple conditions allow **fine-grained coordination**.

***

### 10. Lock vs Synchronized

| Feature             | synchronized | Lock         |
| ------------------- | ------------ | ------------ |
| Simplicity          | Easy         | More verbose |
| try acquire         | ❌            | ✅            |
| interruptible wait  | ❌            | ✅            |
| fairness control    | ❌            | ✅            |
| multiple conditions | ❌            | ✅            |

So rule of thumb:

```
Use synchronized for simple cases
Use Lock for advanced concurrency control
```

***

### 11. Important Internal Detail

`ReentrantLock` internally uses **AQS (AbstractQueuedSynchronizer)**.

This is the **core framework behind many Java concurrency utilities**:

```java
ReentrantLock
Semaphore
CountDownLatch
FutureTask
```

All built on AQS.

Understanding this will eventually explain **how Java manages lock queues and CAS loops**.

***

# 12. Small Exercise

What happens here?

```java
Lock lock = new ReentrantLock();

lock.lock();

try {
    lock.lock();
    System.out.println("Inside");
} finally {
    lock.unlock();
}

lock.unlock();

```

Will it:

A) Deadlock

B) Throw exception

C) Print "Inside"

Think carefully—this checks if you truly understand **reentrancy**.


### Exercise Answer

> main thread acquires lock → acquires again → prints → unlocks twice

Exactly right.

Execution:

```
Thread(main) acquires lock → holdCount = 1
Thread(main) acquires lock again → holdCount = 2
print "Inside"

unlock → holdCount = 1
unlock → holdCount = 0
lock released

```

Output:

```
Inside
```

This works because **ReentrantLock is reentrant**.

***

# 2. Now the Important Part: AQS

AQS stands for:

```
AbstractQueuedSynchronizer
```

It is a **framework used to build locks and synchronizers**.

Most of Java's concurrency tools are built on top of AQS.

Examples:

```
ReentrantLock
Semaphore
CountDownLatch
ReentrantReadWriteLock
FutureTask
```

Instead of every class implementing complex synchronization logic, Java provides **AQS as a reusable foundation**.

Think of it as:

```
Low-level concurrency engine
```

***

### 3. What Problem AQS Solves

When building a lock, you must handle:

```
thread acquisition
waiting threads
thread queues
fairness
blocking
unblocking
```

This is extremely complex.

AQS provides:

```
state management
FIFO wait queue
blocking/unblocking threads
CAS-based synchronization
```

So lock implementations only define **how the state changes**.

***

# 4. Core Concept of AQS

AQS manages **one integer state**:

```
int state
```

This state represents the **synchronization status**.

Examples:

### ReentrantLock

```
state = number of lock holds
```

Example:

```
state = 0 → unlocked
state = 1 → locked once
state = 2 → same thread reentered
```

***

### Semaphore

```
state = permits available
```

Example:

```
state = 5 → 5 permits left
```

***

### CountDownLatch

```
state = remaining count
```

Example:

```
state = 3 → waiting for 3 events
```

***

# 5. AQS Uses CAS

Remember our earlier CAS discussion?

AQS heavily uses:

```java
compareAndSetState()
```

Example:

```java
if state == expected
    update state
```

This ensures **atomic updates without locks**.

***

# 6. AQS Wait Queue

If a thread **fails to acquire the lock**, AQS places it in a **queue**.

Structure:

```
Head → Node → Node → Node
```

Each node represents a **waiting thread**.

Example:

```
Thread1 holds lock

Thread2 waiting
Thread3 waiting
Thread4 waiting
```

Queue:

```
HEAD → T2 → T3 → T4
```

***

# 7. Simplified ReentrantLock Logic

When `lock()` is called:

### Step 1 — Try CAS

```java
if state == 0
    CAS(state,0,1)
```

If successful:

```
thread becomes owner
```

Fast path (no blocking).

***

### Step 2 — Reentrancy Check

If the **same thread already owns the lock**:

```java
state++
```

This enables **reentrant behavior**.

***

### Step 3 — Failure

If another thread holds the lock:

```
thread added to wait queue
thread parked (blocked)
```

***

# 8. Unlock Logic

When `unlock()` is called:

```
state--
```

If:

```
state == 0
```

then:

```
lock fully released
next thread in queue is unparked
```

***

# 9. Visual Flow

```
Thread tries lock
        │
        ▼
   CAS state?
    │      │
   YES     NO
    │      │
 lock acquired
           │
           ▼
     add to queue
           │
           ▼
       park thread

```

***

# 10. Internal Structure of AQS

Simplified internal fields:

```
volatile int state;
Node head;
Node tail;
```

Node structure:

```
Thread reference
next node
prev node
wait status
```

Queue type:

```
CLH queue (variant)
```

***

# 11. Why AQS Is Brilliant

It combines:

```
CAS
queueing
blocking
```

into one framework.

Benefits:

```
scalable
high-performance
reusable
```

Instead of writing lock algorithms repeatedly.

***

# 12. Example: ReentrantLock Built on AQS

Internally:

```
ReentrantLock
      ↓
Sync class
      ↓
extends AbstractQueuedSynchronizer
```

The `Sync` class implements:

```java
tryAcquire()
tryRelease()
```

AQS handles the rest.

***

# 13. Real Code Example (Simplified)

Inside `ReentrantLock`:

```java
protected final boolean tryAcquire(int acquires) {

    int c = getState();

    if (c == 0) {

        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }

    }

    else if (Thread.currentThread() == getOwner()) {

        setState(c + acquires);
        return true;

    }

    return false;
}

```

That logic implements:

```
first acquire
reentrant acquire
```

***

# 14. Why AQS Is Important for Interviews

Many concurrency primitives rely on it.

Understanding AQS helps you understand:

```
locks
semaphores
latches
futures
```

All of them use the same internal engine.

***

# 15. Where You’ll See AQS Again

Next topics that rely on it:

```
Semaphore
CountDownLatch
ReadWriteLock
StampedLock
FutureTask
```

***

# Before moving forward, one quick thinking check

Imagine this code:

```java
ReentrantLock lock = new ReentrantLock();

Thread t1 = new Thread(() -> {
    lock.lock();
    try {
        Thread.sleep(5000);
    } finally {
        lock.unlock();
    }
});

Thread t2 = new Thread(() -> {
    lock.lock();
    try {
        System.out.println("Acquired");
    } finally {
        lock.unlock();
    }
});

```

Question:

```
What happens to Thread t2 while t1 holds the lock?
```

Does it:

A) Spin continuously

B) Sleep / block

C) Crash

D) Retry CAS forever

This checks if you understand **how AQS parks threads**.

***

# Step-by-Step Execution

### Thread t1 starts

```
lock.lock();
```

Inside `ReentrantLock.lock()`:

1. `state == 0`
2. CAS succeeds

```
CAS(state, 0 → 1)
```

Now:

```
state = 1
owner = t1
```

`t1` enters the critical section.

***

### Thread t2 starts

```
lock.lock();
```

Now internally:

```
state = 1
owner = t1
```

So `t2` cannot acquire the lock.

ReentrantLock now calls AQS:

```
acquire(1)
```

Inside AQS:

1️⃣ Try `tryAcquire()`

Fails.

2️⃣ Thread is **added to wait queue**

Queue becomes:

```
HEAD → t2
```

3️⃣ Thread is **parked**

Java internally calls:

```
LockSupport.park()
```

This means:

```
t2 is BLOCKED (not spinning)
```

Very important point:

```
CPU usage = 0
```

This is why AQS scales well.

***

# After 5 Seconds

`t1` executes:

```
lock.unlock();
```

Inside unlock:

```
state--
```

Now:

```
state = 0
```

Lock fully released.

AQS now wakes the **next thread in queue**.

```
LockSupport.unpark(t2)
```

`t2` wakes up and retries `tryAcquire()`.

Now:

```
state = 0
CAS(state,0,1) succeeds
```

`t2` acquires the lock.

***

# Final Execution

`t2` runs:

```
System.out.println("Acquired");
```

Output:

```
Acquired
```

Exactly what you predicted. ✅

***

# Important Insight (Very Interview Relevant)

`t2` does **NOT spin**.

Instead:

```
CAS → fail
enqueue
park thread
wait for unpark
retry
```

This avoids **CPU waste**.

Comparison:

| Lock Type     | Behavior                        |
| ------------- | ------------------------------- |
| Spin Lock     | loops continuously              |
| ReentrantLock | blocks thread using park/unpark |

***

# Internal Queue Visualization

While `t1` holds lock:

```
state = 1

Owner → t1

Queue:
HEAD → t2
```

When `t1` unlocks:

```
unpark(t2)
```

Queue:

```
HEAD → (removed)
```

***

# Subtle Detail (Advanced)

When `t2` wakes up, it **does not immediately own the lock**.

Instead it **retries CAS again**.

Why?

Because another thread **might have acquired the lock first**.

So flow is:

```
unpark
retry CAS
if success → acquire
else → park again
```

This ensures correctness under **high contention**.

***

# Now One Conceptual Question (This is where people usually get stuck)

Consider:

```java
ReentrantLock lock = new ReentrantLock();

Thread t1 = new Thread(() -> {
    lock.lock();
    try {
        Thread.sleep(5000);
    } finally {
        lock.unlock();
    }
});

Thread t2 = new Thread(() -> {
    lock.lock();
    lock.unlock();
});

Thread t3 = new Thread(() -> {
    lock.lock();
    lock.unlock();
});

```

Assume execution order:

```
t1 → t2 → t3
```

Queue becomes:

```
HEAD → t2 → t3
```

Now when `t1` unlocks, **which thread gets the lock first?**

A) Always `t2`

B) Could be `t2` or `t3`

C) Always `t3`

D) Random thread

This question leads directly into **Fair vs Non-Fair locks**, which is the **next major concept in Java locking**. 🚀

### Timeline

```
t1 holds lock
t2 is parked in queue, sleeping

t1 unlocks
    ↓
AQS signals t2 to wake up ──→ t2 needs time to:
                               - get scheduled by OS
                               - get CPU time
                               - actually resume execution
    ↓
Meanwhile... t3 arrives fresh
    ↓
t3 is already running on CPU
t3 tries CAS(state, 0, 1)  ← happens instantly
    ↓
t3 wins, acquires lock
    ↓
t2 finally wakes up, tries CAS, fails
t2 goes back to sleep in queue
```

`t2` loses despite waiting longer.

This is called **barging**. 

###### So Correct Answer

The correct answer to the question was:

B) Could be `t2` or `t3`

Because non-fair locks allow barging.



##### Fair vs Non-Fair Summary

| Property   | Non-Fair Lock  | Fair Lock   |
| ---------- | -------------- | ----------- |
| Default    | Yes            | No          |
| Throughput | Higher         | Lower       |
| FIFO order | Not guaranteed | Guaranteed  |
| Barging    | Allowed        | Not allowed |

#### One Very Interesting Fact

Even **fair locks are not perfectly fair**.

Because **OS thread scheduling is unpredictable**.

But AQS tries its best to maintain FIFO.



#### One More Question (This reveals deep understanding)

Consider this code:

```java
ReentrantLock lock = new ReentrantLock(true); // fair lock
Thread t1 = new Thread(() -> {
    lock.lock();
    try {
        Thread.sleep(5000);
    } finally {
        lock.unlock();
    }
});
Thread t2 = new Thread(() -> {
    lock.lock();
    lock.unlock();
});
Thread t3 = new Thread(() -> {
    lock.lock();
    lock.unlock();
});
```

Queue:

```
HEAD → t2 → t3
```

Now imagine a completely new thread `t4` appears right when `t1` unlocks.

Question:

Can `t4` ever acquire the lock before `t2`?

Yes or No?

**Your answer is correct:&#x20;****`t4`****&#x20;cannot acquire the lock before&#x20;****`t2`****.**

But let’s understand **why**, because the mechanism is interesting and tied directly to **AQS internals**.

***

## What Happens When `t1` Unlocks

Queue:

```
HEAD → t2 → t3
```

`t1` releases the lock:

```
state = 0
```

AQS then:

```
unpark(t2)
```

So `t2` becomes runnable.

***

### Now `t4` Appears

`t4` calls:

```
lock.lock()
```

Since this is a **fair lock**, `ReentrantLock` does something **extra** before trying CAS.

Internally it checks:

```java
hasQueuedPredecessors()
```

***

### What `hasQueuedPredecessors()` Does

It checks if there are threads already waiting.

Conceptually:

```java
head.next != currentThread
```

If **someone is already waiting**, the new thread **must not barge**.

So for `t4`:

```
queue = HEAD → t2 → t3
```

Result:

```java
hasQueuedPredecessors() = true
```

Therefore:

```
t4 cannot acquire lock
```

Instead it joins the queue.

Queue becomes:

```
HEAD → t2 → t3 → t4
```

***

### Then `t2` Wakes Up

`t2` was already unparked earlier.

Now it retries acquisition:

```
CAS(state,0,1)
```

Success.

`t2` becomes owner.

***

### Final Order

```
t1 → t2 → t3 → t4
```

FIFO preserved.

***

### Why This Is Important

Fair locks enforce:

```
no barging
```

Non-fair locks allow:

```
new threads skipping queue
```

That small check:

```
hasQueuedPredecessors()
```

is the **entire fairness mechanism**.

***

# Real Internal Difference

### Non-fair lock

```java
if (compareAndSetState(0, 1))
    acquire
```

### Fair lock

```java
if (!hasQueuedPredecessors() &&
    compareAndSetState(0, 1))
    acquire
```

That single condition changes everything.

***

### Performance Tradeoff

Fair locks are slower because **every acquisition must check the queue**.

Non-fair locks skip that check.

Which is why **default Java locks are non-fair**.

***

### Quick Mental Model

Non-fair lock:

```
queue exists
new thread arrives
new thread may grab lock first
```

Fair lock:

```
queue exists
new thread arrives
must go to queue
```

***

### One More Advanced Insight (Important for Interviews)

Even with **fair locks**, this can happen:

```
t2 is unparked
but OS delays t2
```

Meanwhile:

```
t3 runs first
```

But `t3`**still cannot acquire the lock**, because:

```java
hasQueuedPredecessors() = true
```

So fairness is preserved **at the lock level**, even if scheduling is unpredictable.

***
# Java Thread Lifecycle

A thread in Java goes through **several well-defined states** during its lifetime.

The states are defined in

> java.lang.Thread.State.

Java defines **6 states**.

```
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

Lifecycle overview:

```
        start()
NEW  -------------> RUNNABLE
                      |
                      v
                   RUNNING
                      |
       --------------------------------
       |              |              |
    BLOCKED        WAITING     TIMED_WAITING
       |              |              |
       ---------------|--------------
                      v
                   RUNNABLE
                      |
                      v
                 TERMINATED
```

Important: **RUNNING is not a Java state** — it is part of `RUNNABLE`.

## 1️⃣ NEW State

A thread is in **NEW** state when it is **created but not started yet**.

Example:

```java
Thread t = new Thread(() -> {
    System.out.println("Hello");
});
```

At this moment:

```java
Thread.State = NEW
```

The thread **exists as an object** but the OS scheduler doesn't know about it yet.

## 2️⃣ RUNNABLE State

When we call:

```java
t.start();
```

The thread moves to **RUNNABLE**.

```
NEW → RUNNABLE
```

Now the thread is:

* registered with the **OS scheduler**
* eligible to run on CPU

Important nuance:

In Java:

> RUNNABLE = ready to run OR currently running

Java does **not distinguish** between:

```
READY
RUNNING
```

Both are represented as:

> RUNNABLE

## 3️⃣ BLOCKED State

A thread enters **BLOCKED** when it is waiting to acquire a **monitor lock** (`synchronized`).

Example:

```java
synchronized(lock) {
    // critical section
}
```

If another thread holds the lock:

```
Thread → BLOCKED
```

Example:

```
Thread A holds lock
Thread B tries synchronized(lock)
Thread B → BLOCKED
```

The thread will remain blocked **until the lock becomes available**.

Important:

> **BLOCKED** only happens with synchronized

Not with `wait()`.

## 4️⃣ WAITING State

A thread enters **WAITING** when it waits **indefinitely for another thread to signal it**.

Typical ways:

```java
Object.wait()
Thread.join()
LockSupport.park()
```

Example:

```java
synchronized(lock) {
    lock.wait();
}
```

Thread state:

> WAITING

The thread will remain here **until another thread wakes it up**.

Example wake-up:

```java
lock.notify();
lock.notifyAll();
```

## 5️⃣ TIMED\_WAITING State

This is similar to WAITING but **with a timeout**.

Examples:

```
Thread.sleep(1000)
wait(1000)
join(1000)
parkNanos()
```

State:

> TIMED\_WAITING

After timeout:

> TIMED\_WAITING → RUNNABLE

## 6️⃣ TERMINATED State

A thread enters **TERMINATED** when its `run()` method completes.

Example:

```java
Thread t=newThread(() -> {
System.out.println("Done");
});
```

After execution finishes:

> Thread.State = TERMINATED

Important rule:

```
A terminated thread cannot be restarted.
```

This will throw exception:

```java
t.start();
t.start(); // IllegalThreadStateException
```

### Full Example

```java
public class ThreadStateExample {
    public static void main(String[] args) throws Exception {
        Thread t = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {}
        });
        System.out.println(t.getState()); // NEW
        t.start();
        System.out.println(t.getState()); // RUNNABLE
        Thread.sleep(100);
        System.out.println(t.getState()); // TIMED_WAITING
        t.join();
        System.out.println(t.getState()); // TERMINATED
    }
}
```



### Important Distinction (Interview Critical)

| State          | Why thread stops             |
| -------------- | ---------------------------- |
| BLOCKED        | waiting for **monitor lock** |
| WAITING        | waiting for **signal**       |
| TIMED\_WAITING | waiting **with timeout**     |

Example mapping:

```
synchronized → BLOCKED
wait() → WAITING
sleep() → TIMED_WAITING
```

### NEW

Think of thread lifecycle like **airport travel**.

```
NEW
(ticket booked)

RUNNABLE
(waiting for boarding)

RUNNING
(on plane)

BLOCKED
(waiting for runway)

WAITING
(waiting for passenger)

TIMED_WAITING
(waiting for fixed time)

TERMINATED
(reached destination)
```

### One Important Detail (Almost Everyone Misses)

Calling **`run()`****&#x20;does NOT start a new thread**.

Wrong:

```java
t.run();
```

This executes **in the current thread**.

Correct:

```java
t.start();
```

`start()`:

```
creates new OS thread
calls run()
```

***

## 1️⃣ `Thread.sleep()`

Method from

> java.lang.Thread

Purpose:

```
Pause the current thread for a fixed time.
```

Example:

```java
Thread.sleep(2000);
```

Meaning:

```
Current thread pauses for 2 seconds
```

### State Transition

```
RUNNABLE → TIMED_WAITING → RUNNABLE
```

### Important Properties

1️⃣ **Does NOT release locks**

Example:

```java
synchronized(lock) {
    Thread.sleep(5000);
}
```

Even during sleep:

```
lock is still held
```

Other threads **cannot enter the synchronized block**.

2️⃣ Sleep is **static**

```java
Thread.sleep(...)
```

Not:

```java
t.sleep(...)
```

Because it always affects **current thread**.

***

## 2️⃣ `Object.wait()`

Method from

> java.lang.Object

Purpose:

```
Wait until another thread signals.
```

Example:

```java
synchronized(lock) {
    lock.wait();
}
```

### State Transition

```
RUNNABLE → WAITING → RUNNABLE
```

If timeout used:

```
RUNNABLE → TIMED_WAITING → RUNNABLE
```

### Important Rule

`wait()`**must be called inside synchronized**.

Example:

```java
synchronized(lock) {
    lock.wait();
}
```

Otherwise:

```
IllegalMonitorStateException
```

### Critical Behavior

When a thread calls `wait()`:

```
1. Releases the lock
2. Enters WAITING state
3. Sleeps until notified
```

This is **very different from sleep()**.

***

### 3️⃣ `notify()` and `notifyAll()`

Used with `wait()`.

Example:

```java
synchronized(lock) {
    lock.notify();
}
```

or

```java
synchronized(lock) {
    lock.notifyAll();
}
```

### Behavior

```
notify()     → wakes one waiting thread
notifyAll()  → wakes all waiting threads
```

Important:

Woken threads **do not run immediately**.

They move to:

```
WAITING → BLOCKED → RUNNABLE
```

Because they must **re-acquire the monitor lock**.

***

## 4️⃣ `Thread.join()`

Also from

> java.lang.Thread.

Purpose:

```
Wait for another thread to finish.
```

Example:

```java
Thread worker = new Thread(() -> {
    doWork();
});

worker.start();
worker.join();
```

Meaning:

```
Current thread waits until worker finishes.
```

### State Transition

```
RUNNABLE → WAITING → RUNNABLE
```

If timeout used:

```java
worker.join(1000);
```

State:

```
RUNNABLE → TIMED_WAITING → RUNNABLE
```

***

## Example of `join()`

```java
public class JoinExample {
    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(2000);
                System.out.println("Worker done");
            } catch (InterruptedException e) {}
        });

        worker.start();

        System.out.println("Waiting for worker...");

        worker.join();

        System.out.println("Main resumes");
    }
}

```

Output:

```
Waiting for worker...
Worker done
Main resumes
```

***

### `sleep()` vs `wait()` (Very Important)

| Feature                | sleep()        | wait()              |
| ---------------------- | -------------- | ------------------- |
| Class                  | Thread         | Object              |
| Lock released?         | ❌ No           | ✅ Yes               |
| Requires synchronized? | ❌ No           | ✅ Yes               |
| Purpose                | pause thread   | thread coordination |
| State                  | TIMED\_WAITING | WAITING             |

Example difference:

#### sleep()

```
Thread holds lock but pauses
```

#### wait()

```
Thread releases lock and waits
```

***

### `wait()` vs `join()`

| Feature       | `wait()`            | `join()`          |
| ------------- | ------------------- | ----------------- |
| Used for      | thread coordination | thread completion |
| Requires lock | yes                 | no                |
| Object        | any object          | thread object     |

Example:

```
Producer / Consumer → wait()
Thread completion → join()
```

***

### Real Bug Example (Classic Interview Problem)

Wrong code:

```java
synchronized(lock) {
    while(queue.isEmpty()) {
        Thread.sleep(100);
    }
}

```

Problem:

```
Thread holds lock while sleeping
Producer cannot acquire lock
Deadlock-like stall
```

Correct version:

```java
synchronized(lock) {
    while(queue.isEmpty()) {
        lock.wait();
    }
}
```

Here:

```
consumer releases lock
producer can insert item
producer calls notify()
```

This is the **correct coordination pattern**.

***

## Real Debugging Tip (Senior Engineer Trick)

You can inspect thread states using:

```
jstack <pid>
```

Example output:

```
Thread-1   WAITING (on object monitor)
Thread-2   BLOCKED (on object monitor)
Thread-3   TIMED_WAITING (sleeping)
```

You instantly know:

```
who holds locks
who is waiting
who is sleeping
```

This is **extremely useful when debugging deadlocks**.

***

### Important Rule Most Developers Miss

Always use **`wait()`****&#x20;inside a loop**.

Wrong:

```java
if(queue.isEmpty()) {
    wait();
}
```

Correct:

```java
while(queue.isEmpty()) {
    wait();
}
```

Reason:

```
Spurious wakeups
```

The JVM **can wake a thread even without notify()**.

***

# Intrinsic Locks and `synchronized`

Every object in Java has an **intrinsic lock** (also called a **monitor lock**).

The keyword

> synchronized

is used to **acquire and release that lock automatically**.

This mechanism is built directly into the JVM.

***

## What Is an Intrinsic Lock?

Every object implicitly contains:

```
Object
 ├── Data
 └── Monitor (Intrinsic Lock)
```

That monitor ensures **mutual exclusion**.

Meaning:

```
Only ONE thread can execute synchronized code
protected by the same lock at a time.
```

***

## Example Without Synchronization

```java
class Counter {
    int count = 0;

    void increment() {
        count++;
    }
}

```

Multiple threads running:

```java
count++
```

can cause **race conditions**.

Because this operation actually becomes:

```
load count
add 1
store count
```

Two threads may overwrite each other.

***

### Fix Using `synchronized`

```java
class Counter {
    int count = 0;

    synchronized void increment() {
        count++;
    }
}
```

Now:

```
only one thread enters increment() at a time
```

This ensures **thread safety**.

***

### What Happens Internally

When a thread enters a synchronized block:

```
monitorenter
```

When it exits:

```
monitorexit
```

These are **JVM bytecode instructions**.

The lock is:

```
acquired → critical section → released
```

If another thread tries to enter:

```
Thread → BLOCKED
```

***

## Two Ways to Use `synchronized`

### 1️⃣ Synchronized Method

```java
synchronized void increment() {
    count++;
}
```

Equivalent to:

```java
void increment() {
    synchronized(this) {
        count++;
    }
}
```

The lock used is:

```
this (current object)
```

***

### 2️⃣ Synchronized Block

More flexible:

```java
synchronized(lockObject) {
    // critical section
}
```

Example:

```java
class Counter {

    private final Object lock = new Object();
    int count = 0;

    void increment() {
        synchronized(lock) {
            count++;
        }
    }
}

```

Now the monitor belongs to:

```
lock object
```

not the whole class instance.

***

### Why Blocks Are Often Better

Blocks allow **fine-grained locking**.

Example:

```java
synchronized(lockA) {
    // modify A
}

synchronized(lockB) {
    // modify B
}

```

Now operations on A and B **can run concurrently**.

This improves **scalability**.

***

## Object Lock vs Class Lock

This is an important distinction.

#### Object Lock

```java
synchronized void method()
```

or

```java
synchronized(this)
```

Lock used:

```
this instance
```

Each object has its **own lock**.

Example:

```
Counter c1
Counter c2
```

Two threads can execute simultaneously.

***

### Class Lock

Used with static methods.

```java
static synchronized void method() {
}
```

Equivalent to:

```java
synchronized(Counter.class) {
}
```

Lock used:

```
Class object
```

This means:

```
one thread per class
```

even across instances.

***

### Example Demonstrating Both

```java
class Example {
    synchronized void objectLock() {
        System.out.println("Object lock");
    }

    static synchronized void classLock() {
        System.out.println("Class lock");
    }
}

```

Locks used:

```
objectLock() → instance monitor
classLock() → Example.class monitor
```

***

# How Threads Behave

Example:

```
Thread A enters synchronized block
Thread B tries same block
```

Result:

```
Thread A → RUNNABLE
Thread B → BLOCKED

```

Thread B waits until:

```
Thread A exits
```

Then:

```
Thread B → RUNNABLE
```

***

### Important Property: Automatic Unlock

Locks are **automatically released**.

Example:

```java
synchronized(lock) {
    doWork();
}
```

Even if exception occurs:

```
lock is released
```

This prevents many bugs.

***

### Critical Property: Visibility

`synchronized` also provides **memory visibility guarantees**.

Meaning:

```
changes by Thread A
become visible to Thread B
after lock release/acquire
```

This relates to the **Java Memory Model**, which we'll cover later.

***

## Common Mistake

Locking on the wrong object.

Bad:

```java
synchronized(new Object()) {
    count++;
}
```

This creates **new lock each time**.

So:

```
no synchronization actually occurs
```

Correct:

```java
private final Object lock = new Object();
```

***

### Classic Example

Thread-safe counter:

```java
class Counter {

    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int get() {
        return count;
    }
}

```

Now:

```
increment() and get() share same lock
```

ensuring consistency.

***

# Summary

Intrinsic locks provide:

```
Mutual exclusion
Visibility guarantees
Automatic lock management
```

Key concepts:

```
Every object has a monitor
synchronized acquires monitor
one thread at a time
other threads BLOCKED
```

***

## 1️⃣ Reentrancy in `synchronized`

Intrinsic locks used by

> **synchronized**

are **reentrant**.

Meaning:

```
A thread that already holds a monitor lock
can acquire it again without blocking.
```

Example:

```java
class Example {
    synchronized void methodA() {
        System.out.println("methodA start");
        methodB();
        System.out.println("methodA end");
    }

    synchronized void methodB() {
        System.out.println("methodB");
    }
}

```

Execution:

```
Thread T1 enters methodA
T1 acquires monitor lock

methodA calls methodB

methodB also requires the same monitor
```

Since **T1 already owns the lock**, it is allowed to enter.

Execution continues normally.

Output:

```
methodA start
methodB
methodA end
```

So the lock is **not acquired once per method**, but **per thread**.

***

## 2️⃣ Intrinsic Monitor Hold Count

Every monitor internally tracks two things:

```
Monitor
 ├ Owner Thread
 └ Hold Count
```

Conceptually:

```
Owner = thread currently holding the lock
HoldCount = number of times it acquired the lock
```

Example execution:

```
Thread T1 enters methodA
HoldCount = 1
```

Inside `methodA`:

```
methodB() is called
```

Now:

```
Thread T1 acquires same lock again
HoldCount = 2
```

When `methodB` exits:

```
HoldCount = 1
```

When `methodA` exits:

```
HoldCount = 0
Lock released
```

So the lock is only released when:

```
holdCount == 0
```

***

## 3️⃣ Why Reentrancy Exists

Without reentrancy, **nested synchronized calls would deadlock**.

Example:

```java
class BankAccount {
    synchronized void deposit() {
        updateBalance();
    }

    synchronized void updateBalance() {
        // update logic
    }
}

```

Execution:

```
Thread T1 calls deposit()
deposit() acquires lock
```

Then:

```
deposit() calls updateBalance()
```

Now `updateBalance()` tries to acquire the same lock.

If locks were **not reentrant**:

```
updateBalance() would BLOCK
```

But the lock is owned by:

```
Thread T1 itself
```

So T1 would be waiting for **itself**.

That would create:

```
self-deadlock
```

Because Java locks are **reentrant**, the JVM detects:

```java
currentThread == lockOwner
```

and simply:

```
holdCount++
```

***

### 4️⃣ Self-Deadlock (What Reentrancy Prevents)

Self-deadlock means:

```
A thread waits for a lock it already holds.
```

Example if locks were not reentrant:

```
Thread T1
  acquires Lock A
  tries to acquire Lock A again
  waits forever
```

This would freeze the thread.

Reentrancy prevents this by allowing:

```
same thread
same lock
multiple acquisitions
```

Important distinction:

Reentrancy **does NOT prevent deadlocks between threads**.

Example still deadlocks:

```
Thread1 → lockA → lockB
Thread2 → lockB → lockA
```

This is **classic circular deadlock**.

Reentrancy only prevents **self-deadlock**.

***

### 5️⃣ Recursive Lock Terminology

In operating system literature, you may see two terms:

```
Reentrant lock
Recursive lock
```

They mean **the same thing**.

Example:

| Term            | Used By               |
| --------------- | --------------------- |
| Reentrant Lock  | Java                  |
| Recursive Mutex | POSIX / OS literature |

Example in POSIX:

```
PTHREAD_MUTEX_RECURSIVE
```

Example in Java:

```
ReentrantLock
```

Both allow:

```
same thread
multiple lock acquisitions
```

***

### 6️⃣ Visual Example of Reentrancy

Example execution timeline:

```
Thread T1 calls methodA()
```

```
Acquire monitor
HoldCount = 1
```

```
methodA calls methodB()
```

```
Acquire monitor again
HoldCount = 2
```

```
methodB exits
HoldCount = 1
```

```
methodA exits
HoldCount = 0
Lock released
```

Now another thread can acquire it.

***

### 7️⃣ Example Showing Hold Count Behavior

```java
class ReentrantExample {

    synchronized void outer() {
        System.out.println("Outer start");
        inner();
        System.out.println("Outer end");
    }

    synchronized void inner() {
        System.out.println("Inner method");
    }
}

```

Execution:

```
T1 enters outer()
lock acquired
holdCount = 1

inner() called
lock acquired again
holdCount = 2

inner() exits
holdCount = 1

outer() exits
holdCount = 0
```

Lock finally released.

***

### 8️⃣ Important Insight

Because intrinsic locks are reentrant:

```
nested synchronized calls are safe
```

This allows developers to structure code naturally:

```
public API methods
→ internal helper methods
→ all synchronized
```

Without worrying about **self-deadlocks**.

***

### Summary

Intrinsic lock reentrancy works like this:

```
Each monitor tracks:

owner thread
hold count
```

Rules:

```
same thread → allowed to reenter
holdCount increases
unlock only when holdCount == 0
```

Reentrancy prevents:

```
self-deadlock
```

But not:

```
multi-thread deadlock
```

***
# Java Memory Model (JMM)

The Java Memory Model defines **how threads interact through memory** — it specifies the rules that govern when changes made by one thread become visible to other threads, and what values a thread can read from shared variables.

***

## Why JMM Exists

Modern hardware is complex:

* CPUs have **multiple levels of cache** (L1, L2, L3)
* Compilers and CPUs **reorder instructions** for performance
* Each thread may work with a **local copy** of a variable rather than main memory

Without a memory model, concurrent programs would behave unpredictably across different hardware and JVM implementations.

***

## The Core Abstraction

JMM models memory as:

* **Main Memory** — shared storage where all variables live
* **Working Memory (per thread)** — each thread has its own local cache/registers

A thread **reads** a variable into its working memory, **operates** on it, and **writes** it back. The problem: another thread may not see the updated value immediately.

***

## Key Concepts

### 1. Visibility

When one thread writes a value, another thread may **not see it** due to caching. JMM defines when a write is **guaranteed to be visible**.

```java
// Without synchronization — thread B may never see x = 1
int x = 0;

Thread A: x = 1;
Thread B: while (x == 0) {} // may loop forever!
```

### 2. Atomicity

Some operations are **not atomic** by default. For example, `long` and `double` reads/writes are 64-bit and may be split into two 32-bit operations on some platforms.

```java
long counter = 0;
counter++;  // NOT atomic! It's read → modify → write (3 steps)
```

### 3. Ordering / Reordering

The compiler and CPU may **reorder instructions** for optimization, as long as it doesn't affect single-threaded behavior. But this can break multi-threaded logic.

```java
// The compiler might reorder these two lines!
flag = true;
data = 42;

// Another thread checking flag=true might still see data=0
```

***

## The Happens-Before Relationship

This is the **heart of JMM**. If action **A happens-before** action **B**, then:

* All effects of A are **visible** to B
* A appears to execute **before** B

### Built-in Happens-Before Rules:

| Rule               | Description                                                                            |
| ------------------ | -------------------------------------------------------------------------------------- |
| **Program Order**  | Each action in a thread happens-before every subsequent action in the same thread      |
| **Monitor Lock**   | An `unlock` happens-before every subsequent `lock` on the same monitor                 |
| **Volatile Write** | A write to a `volatile` variable happens-before every subsequent read of that variable |
| **Thread Start**   | `Thread.start()` happens-before any action in the started thread                       |
| **Thread Join**    | All actions in a thread happen-before `Thread.join()` returns                          |
| **Transitivity**   | If A happens-before B, and B happens-before C, then A happens-before C                 |

***

## JMM Tools to Enforce Correct Behavior

### `synchronized`

* Guarantees **mutual exclusion** (atomicity)
* On **exit**: flushes all writes to main memory
* On **entry**: invalidates local cache, reads fresh from main memory

```java
synchronized (lock) {
    counter++; // safe — atomic + visible
}

```

### `volatile`

* Guarantees **visibility** (not atomicity)
* Every write goes **directly to main memory**
* Every read comes **directly from main memory**
* Prevents reordering around volatile access

```java
volatile boolean flag = false;

// Thread A
data = 42;
flag = true;  // volatile write — flushes data too

// Thread B
if (flag) {   // volatile read
    use(data); // guaranteed to see data = 42
}

```

> **Note:**`volatile` does NOT make compound operations like `i++` atomic.
> So this is still unsafe:

```java
volatile int count;
count++; // ❌ not atomic
```

### `final`

* Fields written in the constructor and marked `final` are **safely published**
* Other threads are guaranteed to see the final field's value after construction completes

```java
class Immutable {
    final int value;
    Immutable(int v) { this.value = v; }
}

```

### `java.util.concurrent` (Higher-level tools)

Built on top of JMM guarantees:

| Tool                          | Use Case                   |
| ----------------------------- | -------------------------- |
| `AtomicInteger`, `AtomicLong` | Atomic compound operations |
| `ReentrantLock`               | Flexible locking           |
| `ConcurrentHashMap`           | Thread-safe map            |
| `CountDownLatch`, `Semaphore` | Thread coordination        |

***

## Classic JMM Problem: Double-Checked Locking

```java
// BROKEN without volatile (pre-Java 5)
class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {             // check 1
            synchronized (Singleton.class) {
                if (instance == null) {     // check 2
                    instance = new Singleton(); // reordering hazard!
                }
            }
        }
        return instance;
    }
}

```

The problem: `instance = new Singleton()` is 3 steps:

1. Allocate memory
2. Initialize object
3. Assign reference to `instance`

Steps 2 and 3 can be **reordered**, so another thread might see a non-null but **uninitialized** object.

**Fix:** declare `instance` as `volatile`.

```java
private static volatile Singleton instance; // ✅ correct
```

***

## Summary

| Concept          | Tool                       | Guarantees                   |
| ---------------- | -------------------------- | ---------------------------- |
| Visibility       | `volatile`, `synchronized` | Writes seen by other threads |
| Atomicity        | `synchronized`, `Atomic*`  | Operations not interleaved   |
| Ordering         | `volatile`, `synchronized` | No harmful reordering        |
| Safe Publication | `final`, `synchronized`    | Objects safely shared        |

***

The JMM is subtle but essential. The golden rule: **whenever shared mutable state is accessed by multiple threads, always use proper synchronization** — either via `synchronized`, `volatile`, or higher-level concurrency utilities.



***

# Happens-Before (Deep Dive)

From

```
Java Memory Model
```

***

## 🧠 The Core Rule

```
If A happens-before B,
then B is guaranteed to see ALL effects of A.
```

Two guarantees:

```
1. Visibility → B sees A’s writes
2. Ordering → A is not reordered after B
```

***

## ⚠️ If There Is NO Happens-Before

Then:

```
Anything can happen 😈
```

* stale values
* reordering
* inconsistent reads

This is called a **data race**.

***

## 🔑 The 6 Happens-Before Rules (Must Know)

These are the **only tools** you have to reason about visibility.

***

### 1️⃣ Program Order Rule

Within the same thread:

```java
x = 10;
y = 20;
```

```
x happens-before y
```

Always true.

***

### 2️⃣ Monitor Lock Rule (`synchronized`)

```java
Thread A:
synchronized(lock) {
    x = 10;
}
```

```java
Thread B:
synchronized(lock) {
    print(x);
}
```

Guarantee:

```
unlock(A) happens-before lock(B)
```

So:

```
B will see x = 10 ✅
```

***

### 3️⃣ Volatile Rule

```java
volatile boolean ready;
```

```java
Thread A:
x = 42;
ready = true;
```

```java
Thread B:
if (ready) {
    print(x);
}

```

Guarantee:

```
write(ready) happens-before read(ready)
```

So:

```
B will see x = 42 ✅
```

***

### 4️⃣ Thread Start Rule

```java
Thread t = new Thread(() -> {
    print(x);
});

x = 10;
t.start();
```

Guarantee:

```
x = 10 happens-before t starts
```

So:

```
thread sees x = 10 ✅
```

***

### 5️⃣ Thread Join Rule

```java
Thread t = new Thread(() -> {
    x = 42;
});

t.start();
t.join();

print(x);

```

Guarantee:

```
thread completion happens-before join returns
```

So:

```
print(x) → 42 ✅
```

***

### 6️⃣ Transitivity Rule (Most Powerful)

```
If A happens-before B
and B happens-before C
then A happens-before C
```

This lets you **chain guarantees**.

***

### 🔥 Real Example Using Transitivity

```java
Thread A:
x = 42;
ready = true; // volatile
```

```java
Thread B:
if (ready) {
    print(x);
}
```

Reasoning:

```java
x = 42 happens-before ready = true  (program order)
ready = true happens-before read ready (volatile rule)
```

So:

```
x = 42 happens-before print(x)
```

✅ Safe

***

### ❌ Example Without Happens-Before

```java
int x = 0;
boolean ready = false;

Thread A:
x = 42;
ready = true;

Thread B:
if (ready) {
    print(x);
}

```

No volatile, no lock.

So:

```
NO happens-before relationship
```

Possible outputs:

```
0 ❌
42 ✅
nothing 😈
```

***

### 🧠 How to Think Like a Pro

When you see multithreaded code:

👉 Ask this:

```
"What is the happens-before relationship?"
```

If you can’t find one:

```
The code is broken.
```

***

### 🧠 Instant  Cheat Sheet

| Scenario                                               | Safe? | Why                                    |
| ------------------------------------------------------ | ----- | -------------------------------------- |
| Two threads, no sync, shared variable                  | ❌     | No HB chain                            |
| Same `synchronized` block, same lock                   | ✅     | unlock HB lock                         |
| Different locks on shared data                         | ❌     | No HB between them                     |
| Write before `start()`, read inside thread             | ✅     | start() rule                           |
| Read after `join()`                                    | ✅     | join() rule                            |
| `volatile` write then read                             | ✅     | volatile rule                          |
| `volatile` on one field, non-volatile on related field | ⚠️    | Only safe if write order is controlled |

## Part 2: Real-World Production Bugs from JMM Misunderstanding

***

### Bug 1 — The Invisible Update (Missing Visibility)

```java
// Seen in real server shutdown hooks
class Worker implements Runnable {
    private boolean running = true;  // ← NOT volatile

    public void stop() {
        running = false;  // Thread A writes
    }

    public void run() {
        while (running) {  // Thread B reads — may NEVER see false!
            doWork();
        }
    }
}

```

**What happens in production:** The worker thread **never stops**. CPU caches `running=true` in its register. The JVM has no obligation to re-read it from main memory.

**Fix:**`private volatile boolean running = true;`

🔍 **Why it's sneaky:** Works fine in debug mode (different thread scheduling), fails only under load or on multi-core machines.

***

### Bug 2 — Locking on Different Objects

```java
class Counter {
    private int count = 0;

    public void increment() {
        synchronized (this) { count++; }  // locks on instance A
    }
}

// Meanwhile, someone does:
Counter c = new Counter();
synchronized (new Object()) {  // ← completely different lock!
    c.increment();
}

```

**Real version of this bug:**

```java
// Two teams wrote these independently
class OrderService {
    private List<Order> orders = new ArrayList<>();

    public void add(Order o) {
        synchronized (orders) { orders.add(o); }  // lock on list
    }
}

class ReportService {
    public void generate(List<Order> orders) {
        synchronized (OrderService.class) {  // ← DIFFERENT lock!
            for (Order o : orders) { ... }
        }
    }
}

```

**What happens:** Concurrent modification, `ConcurrentModificationException`, or corrupted reads — all under production load.

**Fix:** Always lock on the **same shared object**, or use `CopyOnWriteArrayList` / `Collections.synchronizedList`.

***

### Bug 3 — Non-Atomic Check-Then-Act

```java
// Classic race condition in a service layer
class TicketService {
    private int availableSeats = 1;

    public void book() {
        if (availableSeats > 0) {       // Thread A checks: 1 > 0 ✅
            // ← Thread B also checks: 1 > 0 ✅ (context switch here)
            availableSeats--;            // Both decrement → seats = -1 😱
            confirmBooking();
        }
    }
}

```

**Production impact:** Double-bookings, overselling inventory, negative bank balances — this pattern causes real financial bugs.

**Fix:**

```java
public synchronized void book() {
    if (availableSeats > 0) {
        availableSeats--;
        confirmBooking();
    }
}
// OR use AtomicInteger + compareAndSet for lock-free version

```

***

### Bug 4 — Partially Constructed Object (Unsafe Publication)

```java
class Config {
    public int timeout;
    public String url;

    public Config() {
        timeout = 5000;
        url = "https://api.example.com";
    }
}

// Shared across threads WITHOUT synchronization
static Config config;

// Thread A
config = new Config();  // reference assigned BEFORE constructor finishes?!

// Thread B
if (config != null) {
    connect(config.url);     // may see url = null!
    setTimeout(config.timeout); // may see timeout = 0!
}

```

**What happens:** The JVM can reorder the assignment of `config` reference **before** the constructor body finishes. Thread B sees a non-null but **half-initialized** object.

**Fix options:**

```java
// Option 1: volatile
static volatile Config config;

// Option 2: make fields final (immutable object = always safe)
class Config {
    public final int timeout;
    public final String url;
    ...
}

// Option 3: synchronize both read and write

```

***

### Bug 5 — volatile on a Compound Action (False Sense of Safety)

```java
class PageViewCounter {
    private volatile long views = 0;

    public void recordView() {
        views++;  // Looks safe because volatile... BUT IT'S NOT!
    }
}

```

**What happens:**`views++` is **three operations**:

1. Read `views` from main memory
2. Increment locally
3. Write back to main memory

Two threads can both read `100`, both increment to `101`, both write `101` — you lost a count.

**Fix:**

```java
// Option 1
private AtomicLong views = new AtomicLong(0);
views.incrementAndGet();  // truly atomic

// Option 2
synchronized (this) { views++; }

```

***

## Part 3: Why volatile Is Not Atomic But Guarantees Visibility & Ordering

This is the most misunderstood part of JMM. Let's break it down precisely.

***

### What volatile Actually Does at the Hardware Level

When you mark a field `volatile`, the JVM inserts **memory barriers** (also called memory fences) around reads and writes.

```
Normal write:          Volatile write:
─────────────          ───────────────
store to register  →   [StoreStore barrier]  ← prevents reordering of prior stores
write to cache     →   store directly to MAIN MEMORY
                       [StoreLoad barrier]   ← ensures all threads see the write

```

These barriers tell the CPU: **"don't reorder across this point, and flush/invalidate caches."**

***

### Visibility — Why It's Guaranteed

```java
volatile boolean flag = false;
int data = 0;

// Thread A
data = 42;
flag = true;   // ← StoreStore barrier BEFORE this write
               //   guarantees data=42 is flushed first
               // ← StoreLoad barrier AFTER this write
               //   forces write to main memory immediately

// Thread B
if (flag) {    // ← LoadLoad barrier ensures fresh read from main memory
    print(data); // sees 42 — guaranteed by HB chain
}

```

Every volatile **write** goes to main memory. Every volatile **read** comes from main memory. No thread-local caching. That's visibility.

***

### Ordering — Why It's Guaranteed

The memory barriers prevent 4 types of reordering:

| Barrier    | Prevents                                                        |
| ---------- | --------------------------------------------------------------- |
| LoadLoad   | Load before barrier can't move after it                         |
| StoreStore | Store before barrier can't move after it                        |
| LoadStore  | Load before barrier can't move after store                      |
| StoreLoad  | Store before barrier can't move after load — **most expensive** |

```java
// Without volatile — compiler/CPU free to reorder:
x = 1;
y = 2;
// Could execute as y=2 then x=1 — fine for single thread

// With volatile y:
x = 1;         // StoreStore barrier ensures this stays BEFORE
y = 2;         // volatile write — ordering locked in

```

***

### Atomicity — Why It's NOT Guaranteed

Visibility and ordering are about **memory flushing and instruction ordering**. Atomicity is about **indivisibility of operations**.

```java
volatile long views = 0;
views++;
```

Even with all the memory barriers in place, `views++` still compiles to:

```
LOAD  views    → register    (read from main memory ✅ visible)
ADD   register, 1
STORE register → views       (write to main memory ✅ visible)
```

Between `LOAD` and `STORE`, another thread can execute its own `LOAD`. Both see the same value. Both store the incremented value. **One increment is lost.**

Volatile ensured both threads read from and write to **main memory** — but it can't prevent the **interleaving** of those three steps. Only a lock or CAS (Compare-And-Swap) can do that.

***

### The Precise Rule to Remember

> **volatile = visibility + ordering, but NOT atomicity**

> Use `volatile` when: one thread writes, others only read. Use `AtomicXxx` or `synchronized` when: multiple threads write.

```java
// ✅ Safe — only one writer
volatile boolean shutdownFlag;

// ✅ Safe — only one writer (config updated atomically by publishing new reference)
volatile Config currentConfig;

// ❌ Unsafe — multiple writers
volatile int counter;
counter++;  // race condition

// ✅ Fix
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();

```

***

### Visual Summary

```
                    volatile          synchronized       AtomicInteger
                    ────────          ────────────       ─────────────
Visibility          ✅ YES            ✅ YES             ✅ YES
Ordering            ✅ YES            ✅ YES             ✅ YES
Atomicity           ❌ NO             ✅ YES             ✅ YES
Performance         🟢 Fast           🔴 Slower          🟡 Medium
Use case            Single writer,    Critical sections, Counters,
                    status flags      compound ops       accumulators

```

***
