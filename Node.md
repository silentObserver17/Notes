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
A compiled program does not work everywhere.
Can i write a code that runs everywhere? -> INTERPRETED LANGUAGE


