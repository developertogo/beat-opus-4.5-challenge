# Performance Optimization Challenge: Beating Claude Opus 4.5

This repository contains the results of the **Anthropic Performance Optimization Exercise** (from [original_performance_takehome](https://github.com/anthropics/original_performance_takehome)).

The objective of this challenge is to optimize a slow, simulated VLIW CPU "kernel" implementation by significantly reducing the number of simulated CPU clock cycles required to run it.

## 📊 Optimization Benchmark Results

![alt text](demo.svg)

## 🏆 The Achievement

By designing a highly optimized kernel schedule and executing aggressive parallelization and loop unrolling, **I successfully beat the Claude Opus 4.5 baseline!**

### Performance Comparison

| Metric | Original Baseline | Claude Opus 4.5 Benchmark | My Optimized Solution |
| :--- | :---: | :---: | :---: |
| **Simulated CPU Cycles** | `18,532` | *Vanquished* | **`1,107`** |
| **Speedup vs. Baseline** | `1.0x` | — | **`~16.74x`** |

<details>
<summary><b>📈 Speedup Calculation Details (Click to expand)</b></summary>

### Speedup Calculation

Using Python notation, the exact speedup factor is calculated as:

```python
original_cycles = 18532
optimized_cycles = 1107

speedup = original_cycles / optimized_cycles
print(f"Speedup: {speedup:.2f}x")  # Output: Speedup: 16.74x
```

This represents an incredible **`16.74x`** speedup over the original implementation, reducing CPU cycle consumption by:

```python
cycle_reduction_percentage = ((original_cycles - optimized_cycles) / original_cycles) * 100
print(f"Reduction: {cycle_reduction_percentage:.2f}%")  # Output: Reduction: 94.03%
```

</details>

---

## Technical Highlights

* **Advanced Scheduling:** Achieved extreme instruction-level parallelism (ILP) by minimizing pipeline stalls and dependencies.
* **Aggressive Loop Transformation:** Implemented hand-crafted unrolling and register renaming to maximize ALU utilization across every clock cycle.

> [!IMPORTANT]
> **Copyright Notice:** Due to copyright infringement restrictions, the actual source code of the optimized solution cannot be published in this repository. Only the benchmark results, execution metrics, and general optimization concepts are shared.

---

## 🌲 The Algorithmic Challenge: Tree Traversal & Hashing

To fully understand the exercise, we must understand the core workload of the kernel, which is a parallel tree traversal combined with custom cryptographic hashing.

### 1. The Tree Structure

The workload operates on an **implicit, perfectly balanced binary tree** (representing a decision forest).

* **Height / Levels:** The tree has a height of `forest_height = 10`, meaning there are **11 levels** (indexed `0` to `10`).
* **Node Count:** The total number of nodes in a perfectly balanced binary tree of height `H` is:

  ```python
  n_nodes = 2 ** (forest_height + 1) - 1  # 2,047 nodes for height 10
  ```

### 2. Traversal Mechanics

A batch of `batch_size = 256` parallel "walkers" start traversing the tree. For each walker, the index begins at the root (`idx = 0`). For each step of the traversal:

1. **Node Value Lookup:** Read the node value `node_val` stored at the current index `idx`.
2. **Cryptographic Hash:** Compute a new walker state `val` by XORing the current walker value with the node value, and applying a custom 32-bit hash function:

   ```python
   hashed_val = myhash(val ^ node_val)
   ```

3. **Branch Selection:** Choose the branching path based on whether `hashed_val` is even or odd:
   * **Left branch** (if even):

     ```python
     next_idx = 2 * idx + 1
     ```

   * **Right branch** (if odd):

     ```python
     next_idx = 2 * idx + 2
     ```

4. **Wrap-Around Constraint:** If the traversal goes beyond the bottom level of the tree (exceeding `n_nodes`), the walker wraps back to the root:

   ```python
   if next_idx >= n_nodes:
       idx = 0
   ```

### 3. The Custom 32-Bit Hash (`myhash`)

At every node, the walker must evaluate a custom **6-stage** pseudo-random hashing function (`HASH_STAGES` from Stage 0 to Stage 5). Each stage performs basic arithmetic, bit-shifting, and XORs using preloaded 32-bit constants:

```python
# Hashing stages structure
HASH_STAGES = [
    ("+", 0x7ED55D16, "+", "<<", 12),
    ("^", 0xC761C23C, "^", ">>", 19),
    ("+", 0x165667B1, "+", "<<", 5),
    ("+", 0xD3A2646C, "^", "<<", 9),
    ("+", 0xFD7046C5, "+", "<<", 3),
    ("^", 0xB55A4F09, "^", ">>", 16),
]
```

Evaluating these `6` heavy pipeline stages sequentially for every walker at every node is the primary performance bottleneck of the original program.

---

## 💻 The Simulated Hardware & VLIW/SIMD Architecture

The challenge is executed on a custom **VLIW (Very Long Instruction Word)** and **SIMD (Single Instruction, Multiple Data)** simulator.

* **SIMD Parallelism:** Through SIMD, the processor operates on vectors of length `VLEN = 8` (where `1` vector contains `8` words, and each word is `32` bits). This allows identical mathematical operations to be performed on `8` different "walkers" concurrently in a single CPU clock cycle.
* **VLIW Bundling:** The CPU supports up to **`23` instruction slots** in a single instruction bundle (one clock cycle):
  * **`12` ALU** (Scalar Math) slots
  * **`6` VALU** (Vector Math) slots
  * **`2` LOAD** (Vector/Scalar Memory Read) slots
  * **`2` STORE** (Vector/Scalar Memory Write) slots
  * **`1` FLOW** (Control / Jumps) slot
* **Compute Capacity:** Packing all functional units with SIMD operations yields a massive capacity of **`93` parallel operations** finished in a single CPU cycle:

  ```python
  max_ops_per_cycle = 12 + (6 * 8) + (2 * 8) + (2 * 8) + 1  # 12 ALU + 48 VALU + 16 LOAD + 16 STORE + 1 FLOW
  print(max_ops_per_cycle)  # Output: 93
  ```

### 🧠 The Scratchpad Constraint

Instead of a traditional register file, the hardware provides a manually managed **Scratch Memory** strictly capped at **`1,536` words** (`max_scratchpad_words = 1536`).
This scratchpad acts as a massive, manually-allocated register pool. Since the hardware has no automatic stalls or register hazard protections, developers must manually schedule and hide latencies (interleaving memory loads for one group of data while computing another) to avoid pipeline hazards and maintain 100% compute occupancy.

---

## 📋 Challenge Requirements

The challenge requires satisfying the following strict criteria:

* **Functional Correctness:** The optimized vectorized kernel must produce mathematically identical output to the baseline scalar reference implementation (traversing nodes, hashing, and branching correctly across all `rounds`).
* **Scratchpad Boundary:** Total memory allocation must remain within the strict budget of **`1,536` words** (`scratch_words_used <= 1536`).
* **VLIW Slot Conformity:** All instructions must compile cleanly without violating the bundle limits (maximum `12` ALU, `6` VALU, `2` LOAD, `2` STORE, and `1` FLOW slot per cycle).
* **Clock Cycle Minimization:** The primary goal is to achieve the lowest possible clock cycles, optimizing away pipeline bubbles and maximizing parallel slot occupancy.
