# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->251.727<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->1.179<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->432386.0<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13672.2<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5012<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
"threads": {
    "1": 2300.247,
    "2": 2041.796,
    "4": 2518.667,
    "8": 2250.35
  }
the fastest is with 2 threads
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version forces the CPU to fetch data one piece at a time, wasting massive amounts of time waiting on memory (cache delays) and processing loop instructions. The vectorized version uses specialized hardware instructions (like SIMD) to grab large, contiguous blocks of data from memory all at once. The machine then applies a single math operation to this entire block simultaneously, fully utilizing its processing power and memory bandwidth.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
This behavior occurs because the standard Google Colab CPU runtime provides only 2 physical vCPUs. Spawning more than 2 threads leads to CPU oversubscription, where thread context-switching overhead consumes CPU cycles. Furthermore, PyTorch intra-op threads must synchronize intermediate tensor states across barriers. Beyond 2 cores, this coordination overhead and L1/L2 cache contention completely outweigh any parallel processing gains.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)


<!--answer:gpu_why-->
Transformer training relies on massive, highly parallel matrix multiplications ($QK^T$, $AV$, and FFN projections). GPUs feature thousands of cores executing identical arithmetic operations concurrently across large, contiguous tensor blocks in memory. 

Conversely, purely sequential, control-heavy workloads—such as single-threaded recursive tree traversals or pointer-chasing linked lists—do not see this speed-up, as GPUs cannot parallelize dependent steps and suffer from severe thread divergence.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism plateaus when physical hardware constraints or software coordination overhead outpace compute gains, across four distinct limits:

1. **Work that cannot be divided:** Naive attention processes tokens sequentially in scalar loops, incurring massive interpreter overhead before vectorization batches arithmetic into unified SIMD instructions.
2. **Hardware limits:** In the CPU thread sweep, performance peaked at 2 threads (2040 ms/step) because the Colab instance provided only 2 physical vCPUs.
3. **Coordination overhead:** Increasing threads from 2 to 4 regressed latency to 2520 ms/step. Inter-thread barrier synchronization and thread-chasing context switches on oversubscribed cores cost more time than parallel execution saved.
4. **Data movement constraints:** In GPU execution, scaling becomes bounded by memory movement. Transferring small tensors across the PCIe bus introduces latency that dominates total step time unless compute density is high enough to saturate GPU pipelines.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would benchmark step latency across varying sequence lengths ($T = 32$ to $2048$) while explicitly measuring host-to-device PCIe transfer overhead versus kernel execution time. If small sequences ($T \le 64$) perform faster on CPU than GPU, it would prove that data movement latency—rather than raw arithmetic throughput—governs small-model efficiency.
<!--/answer-->
