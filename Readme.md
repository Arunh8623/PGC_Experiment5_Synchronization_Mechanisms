# Experiment 5: Synchronization Mechanisms in Parallel Programs

## Overview

This experiment explores how synchronization mechanisms affect **correctness, execution time, and scalability** in shared-memory parallel programs.

We implemented and tested:
- An unsynchronized shared-counter update (race condition)
- OpenMP critical sections
- OpenMP atomic operations
- OpenMP barriers
- POSIX threads with a semaphore
- A comparison benchmark for unsynchronized, critical-section, atomic, and OpenMP-lock approaches

The experiment was performed in **Ubuntu 22.04 running in VMware Workstation**, using **GCC, OpenMP, POSIX threads, and the Ubuntu terminal**.

## Objectives

1. Observe what can happen when multiple threads update shared data without synchronization.
2. Compare critical sections, atomic operations, and locks for a shared-counter workload.
3. Demonstrate how a barrier coordinates threads at a synchronization point.
4. Demonstrate how a semaphore controls access to a shared resource.
5. Measure execution time and check the final counter value with different thread counts.

## Environment and Tools

- VMware Workstation
- Ubuntu 22.04 LTS
- GCC (`build-essential`)
- OpenMP (`-fopenmp`)
- POSIX threads and semaphore library (`-pthread`)
- C

## Synchronization Mechanisms

### 1. Race condition — no synchronization

The program increments a shared `counter` 10,000,000 times using an OpenMP parallel loop, without protecting the increment.

`counter++` is a read-modify-write operation. Concurrent updates can interfere with each other, causing lost increments. The final value is therefore not reliable.

### 2. Critical section

The increment is placed inside an OpenMP `critical` region. Only one thread at a time can execute that protected region, preventing simultaneous counter updates. This ensures correctness, but serializes the updates and can create substantial waiting overhead.

### 3. Atomic operation

The increment is protected using `#pragma omp atomic`. This makes the supported update indivisible with respect to competing updates to the shared counter. It is often more efficient than a general critical section for a simple operation, although contention on a single shared counter can still limit scalability.

### 4. Barrier

The barrier program prints a message before and after `#pragma omp barrier`. Every thread must reach the barrier before any thread can proceed beyond it. The order of thread messages can vary, but all “before barrier” messages should appear before any “after barrier” message. A barrier coordinates phases; it does not by itself make an unprotected shared-counter update safe.

### 5. Semaphore

The POSIX semaphore program creates two threads. Each thread increments a shared counter 100,000 times. A semaphore initialized to 1 is acquired with `sem_wait()` before the increment and released with `sem_post()` afterward. The expected final count is:

`2 × 100,000 = 200,000`

The program waits for both threads to finish before printing the result.

### 6. Comparison benchmark

The comparison program performs 10,000,000 shared-counter increments with **1, 2, 4, and 8 threads**. For each thread count, it tests four approaches:

- `None`: no synchronization
- `Critical`: OpenMP critical region
- `Atomic`: OpenMP atomic update
- `Lock`: OpenMP lock (`omp_set_lock()` / `omp_unset_lock()`)

It records the final count, whether the count matches the expected value, and elapsed wall-clock time using `omp_get_wtime()`.

## Steps Followed

1. Set up Ubuntu 22.04 in VMware Workstation.
2. Installed the compiler and OpenMP support.
3. Created the C source files for the race-condition, critical-section, atomic, barrier, semaphore, and comparison programs.
4. Compiled the OpenMP programs with GCC and `-fopenmp`.
5. Compiled the semaphore program with GCC and `-pthread`.
6. Ran the individual programs to observe their behavior.
7. Ran the comparison benchmark for 1, 2, 4, and 8 threads.
8. Compared final counter values and execution times.

### Installation and compilation

Install the required compiler tools:

```bash
sudo apt update
sudo apt install build-essential -y
```

Compile the programs (run each command from the directory containing its `.c` file):

```bash
gcc race.c -fopenmp -o race
gcc critical.c -fopenmp -o critical
gcc atomic.c -fopenmp -o atomic
gcc barrier.c -fopenmp -o barrier
gcc semaphore.c -pthread -o semaphore
gcc compare_sync.c -fopenmp -o compare_sync
```

Run the individual programs:

```bash
./race
./critical
./atomic
./barrier
./semaphore
```

Run the comparison benchmark:

```bash
./compare_sync
```

The benchmark selects its own thread counts (1, 2, 4, and 8).

## Results

The following table records the output captured in the experiment run. Execution times are in seconds.

| Mechanism | Threads | Final count | Correctness | Time (s) |
|---|---:|---:|---|---:|
| None | 1 | 10,000,000 | Correct in this run | 0.0085 |
| Critical | 1 | 10,000,000 | Correct | 0.1179 |
| Atomic | 1 | 10,000,000 | Correct | 0.0699 |
| Lock | 1 | 10,000,000 | Correct | 0.1186 |
| None | 2 | 5,651,650 | Incorrect | 0.0141 |
| Critical | 2 | 10,000,000 | Correct | 0.3955 |
| Atomic | 2 | 10,000,000 | Correct | 0.1194 |
| Lock | 2 | 10,000,000 | Correct | 0.3021 |
| None | 4 | 2,712,039 | Incorrect | 0.0254 |
| Critical | 4 | 10,000,000 | Correct | 0.7775 |
| Atomic | 4 | 10,000,000 | Correct | 0.1325 |
| Lock | 4 | 10,000,000 | Correct | 0.6286 |
| None | 8 | 3,727,307 | Incorrect | 0.0162 |
| Critical | 8 | 10,000,000 | Correct | 0.7792 |
| Atomic | 8 | 10,000,000 | Correct | 0.1319 |
| Lock | 8 | 10,000,000 | Correct | 0.7512 |

**Note:** These are measurements from one run in a virtual machine, not universal performance guarantees. Timings can vary with host load, VM configuration, CPU resources, compiler settings, and scheduling. The unsynchronized version has a data race, so its timing is not a fair correctness-preserving baseline.

### Results summary

- **No synchronization:** It produced the expected count with one thread in this run, but lost updates with 2, 4, and 8 threads.
- **Critical section:** It produced the correct count for every tested thread count, but became much slower as the thread count increased.
- **Atomic operation:** It produced the correct count for all tested thread counts and was faster than the critical-section and lock approaches in these measurements.
- **OpenMP lock:** It produced the correct count, but its execution time increased considerably with more threads.
- **Barrier:** The output showed all threads printing their “before barrier” message before any thread printed its “after barrier” message.
- **Semaphore:** The program is designed to produce a final count of 200,000 because two threads each perform 100,000 protected increments.

## What We Learned

1. **Race conditions can lose updates.** A shared variable cannot safely be updated concurrently without appropriate synchronization.
2. **Critical sections provide mutual exclusion.** They are useful for protecting a block of code, but a frequently entered critical section can become a bottleneck.
3. **Atomic operations suit simple shared updates.** They protected the counter correctly and had the lowest measured time among the synchronized approaches in this run.
4. **Locks also provide mutual exclusion.** They are useful for larger protected regions, but lock acquisition and contention have a cost.
5. **Barriers synchronize progress, not arbitrary data access.** They ensure all participating threads reach a point before any continue.
6. **Semaphores manage permits.** A semaphore can limit how many threads access a resource at the same time.
7. **More threads do not always mean better performance.** Contention and synchronization overhead can outweigh the benefits of parallel execution.
8. **Correctness matters alongside speed.** A fast result is not useful if it is incorrect or unreliable.

## Conclusion

The experiment demonstrated the trade-off between correctness, synchronization overhead, and scalability in shared-memory parallel programs. In the recorded comparison, atomic updates provided the best performance among the tested **correct** counter-update methods. Critical sections and locks preserved correctness but became increasingly expensive as more threads competed for the same counter. The barrier demonstrated phase synchronization, while the semaphore demonstrated permit-based access control.

The best synchronization mechanism depends on the operation being protected: use atomic operations for supported simple updates, critical sections or locks for larger indivisible code regions, barriers for phase coordination, and semaphores when resource access must be controlled using permits.

## Repository Contents

Keep the source files and this README in the repository, for example:

```text
.
├── race.c
├── critical.c
├── atomic.c
├── barrier.c
├── semaphore.c
├── compare_sync.c
└── README.md
```

You can also add screenshots of the terminal output to document the actual run.
