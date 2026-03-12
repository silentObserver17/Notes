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
