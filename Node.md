# Interpreted Language and the role of V8
## Machine Code:
- RISC vs CISC => CISC are complex and are not usually done in one cpu cycle. RISC are somewhat simpler and thats why are used in arm which is in mobile device.

## ⚙️ **CISC (Complex Instruction Set Computing)**

* **Philosophy:** Make instructions *powerful* so one instruction can do a lot (e.g., load from memory + add + store back in one step).
* **Examples:** Intel x86, AMD64.
* **Characteristics:**

  * Large, complex instruction set.
  * Some instructions take **multiple CPU cycles**.
  * Easier for **assembly programmers** (back in the day), but more work for hardware to decode/execute.
  * High transistor cost for the decoder and microcode.
* **Pros:** Fewer instructions per program (since one instruction can do more).
* **Cons:** Harder to pipeline, more power-hungry, variable instruction length.

---

## ⚙️ **RISC (Reduced Instruction Set Computing)**

* **Philosophy:** Keep instructions *simple and uniform* so they can execute **in 1 CPU cycle** (most of the time).
* **Examples:** ARM, RISC-V, MIPS, SPARC.
* **Characteristics:**

  * Smaller, simpler instruction set.
  * Each instruction does a small task (load, add, store are separate).
  * Fixed instruction size (e.g., 32-bit), easy to pipeline.
  * Needs more instructions per program, but each runs fast.
* **Pros:** High performance per watt, easier parallelism, simpler hardware.
* **Cons:** Compiler must do more work to generate efficient code.

---

## 📱 Why **ARM (RISC) dominates in mobile devices**

* **Power efficiency** is critical (battery life).
* RISC design = simpler, predictable, lower power per instruction.
* Easier to scale cores and integrate in System-on-Chip (SoC).

Meanwhile, **CISC (x86)** dominates desktops/servers because:

* Legacy software ecosystem (decades of x86 code).
* Heavy workloads that benefit from powerful instructions + aggressive micro-optimizations.

---

## 🔎 Fun fact

Modern CPUs blur the line:

* Intel/AMD **x86 (CISC)** chips **internally translate complex CISC instructions into RISC-like micro-ops** before execution.
* So under the hood, even CISC CPUs are “RISC-ish”!

---

👉 So your summary is on point:

* **CISC** = complex instructions, often multi-cycle, used in PCs/servers.
* **RISC** = simpler, efficient instructions, often single-cycle, used in ARM (mobiles, embedded).

---
High Level languages are more convenient than assembly. There are a lot of abstractions to hide complexity. Need to compiile for CPU. comiles turns code to machine code.
translate whatever you have written in an language into something that cpu understands is called compile. we have to compile against an architecture.
intels instructions won't run on mac. or android arm device. 

## Linking
The linker takes all these object files + libraries and connects them into one executable. The file is called "executable".

## INTERPRETD LANGUAGES: 
- A compiled program does not work everywhere.
- Can i write a code that runs everywhere? -> INTERPRETED LANGUAGE
- An interpreted language is one where the code you write (source code) is not directly translated into machine code beforehand. Instead, another program called an interpreter reads and executes it line by line (or statement by statement) at runtime.

Example:
Python, JavaScript, Ruby, PHP are commonly interpreted.

| Feature             | Compiled Languages (C, C++, Go)               | Interpreted Languages (Python, JS, Ruby)           |
| ------------------- | --------------------------------------------- | -------------------------------------------------- |
| **Execution**       | Translated to machine code **before** running | Executed **line by line / via VM**                 |
| **Speed**           | Faster (no runtime translation)               | Slower (translation happens at runtime)            |
| **Portability**     | Needs recompilation on each platform          | Highly portable (interpreter available everywhere) |
| **Error Detection** | Errors caught at compile-time                 | Errors appear at runtime                           |
| **Flexibility**     | Less dynamic (fixed types, structures)        | More dynamic (can eval code at runtime)            |

Your code is NOT compiled to a native .exe.
Instead, it’s fed into an interpreter program (which is a compiled executable, written in C/C++ usually).

Example:

Python → python.exe (Windows) / python3 (Linux)

Node.js (for JavaScript) → node.exe

Great follow-up 👍 Let’s break down **JIT (Just-In-Time compilation)**.

---

## 🔑 What is JIT?

* **JIT = Just-In-Time Compiler**
* It’s a hybrid approach between **compiled** and **interpreted** execution.
* Instead of compiling everything **ahead of time** (like C/C++) or interpreting **line by line** (like old Python), JIT compiles **parts of the code into machine code at runtime** (just in time to run it).

So:
➡️ You get **speed like compiled languages** + **flexibility like interpreted languages**.

---

## 🔨 How JIT Works (Step by Step)

1. **Source Code → Bytecode**

   * Language first compiles your code into bytecode (portable, platform-independent).
   * Example: Java → `.class` file (bytecode).

2. **Interpreter starts execution**

   * Interpreter/VM starts running bytecode instruction by instruction.

3. **Hotspot Detection**

   * The JIT compiler watches which parts of the code run **frequently (hot paths)**.
   * Example: a loop that runs 1 million times.

4. **On-the-fly Compilation**

   * JIT takes those hot parts of bytecode and **compiles them into native machine code**.
   * Stores that machine code in memory.

5. **Direct Execution**

   * Next time that code runs, the VM skips interpretation and **executes the cached machine code directly**.
   * → Much faster execution.

---

## 🔨 Example

### Without JIT (pure interpretation)

```python
for i in range(1_000_000):
    x = i * 2
```

* Each iteration: interpreter reads the loop, parses, runs instruction by instruction.
* Very slow.

### With JIT

* After a few iterations, JIT detects this loop runs many times.
* It compiles the loop into **native CPU code once**.
* The rest of the million iterations run at near-C speed.

---

## 🔑 Languages that use JIT

* **Java** (HotSpot JVM → compiles bytecode into native code at runtime)
* **.NET (C#, F#)** CLR uses JIT
* **JavaScript** (V8 in Chrome/Node.js, SpiderMonkey in Firefox)
* **PyPy** (a JIT-enabled Python implementation)

---

## ⚡ Comparison

| Method             | Description                                       | Speed                                 |
| ------------------ | ------------------------------------------------- | ------------------------------------- |
| **Compiled (AOT)** | Compile everything to machine code before running | Fast startup + fast runtime           |
| **Interpreted**    | Execute line by line with interpreter             | Slow                                  |
| **JIT**            | Compile *hot* code to machine code at runtime     | Startup slower, but runtime very fast |

---

✅ **In short:**
JIT = **Compiles parts of interpreted code into machine code at runtime**, giving a balance of **performance + flexibility**.

---
