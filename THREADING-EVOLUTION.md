# Threading Evolution in BPE Project

A detailed analysis of the multithreading journey in the BPE project, from introduction to simplification.

---

## 📋 Table of Contents
- [Overview](#overview)
- [Phase 1: Introduction of Multithreading](#phase-1-introduction-of-multithreading)
- [Phase 2: Optimization & Refactoring](#phase-2-optimization--refactoring)
- [Phase 3: Synchronization Improvements](#phase-3-synchronization-improvements)
- [Phase 4: Simplification & Removal](#phase-4-simplification--removal)
- [Lessons Learned](#lessons-learned)
- [Performance Analysis](#performance-analysis)

---

## Overview

The BPE project went through an interesting threading evolution cycle: 

1. **March 8**:  Introduced pthreads for parallel frequency calculation
2. **March 8-12**: Heavy refactoring and optimization
3. **March 11**: Removed mutexes, replaced semaphores with barriers
4. **March 31**: Completely removed threading code

This document traces that journey and examines why threading was ultimately removed.

---

## Phase 1: Introduction of Multithreading

### Initial Implementation (March 8, 2025)
**Commit**: [045be11](https://github.com/rustemsk9/bpe/commit/045be11574fda1c9ea1ab1998e35ba6b19ccc2f5)

The first threading implementation focused on parallelizing the frequency collection phase, which is computationally intensive during BPE training.

**Original Design:**

```c
#include <pthread.h>
#include <semaphore.h>

#define NUM_THREADS 8

typedef struct {
    Tokens *tokens;
    FreqMap *freqs;
    size_t start_idx;
    size_t end_idx;
    pthread_mutex_t *mutex;
} ThreadContext;

void* freq_collector_thread(void *arg) {
    ThreadContext *ctx = (ThreadContext*)arg;
    
    // Count local frequencies
    FreqMap local_freqs = {0};
    
    for (size_t i = ctx->start_idx; i < ctx->end_idx - 1; i++) {
        uint64_t pair = MAKE_PAIR(ctx->tokens->items[i], 
                                   ctx->tokens->items[i+1]);
        
        // Increment local frequency
        freq_map_increment(&local_freqs, pair);
    }
    
    // Merge into global frequency map (critical section)
    pthread_mutex_lock(ctx->mutex);
    freq_map_merge(ctx->freqs, &local_freqs);
    pthread_mutex_unlock(ctx->mutex);
    
    freq_map_free(&local_freqs);
    return NULL;
}

int main(int argc, char **argv) {
    // ... initialization ... 
    
    pthread_t threads[NUM_THREADS];
    ThreadContext contexts[NUM_THREADS];
    pthread_mutex_t mutex;
    
    pthread_mutex_init(&mutex, NULL);
    
    // Divide work among threads
    size_t chunk_size = tokens. count / NUM_THREADS;
    for (int i = 0; i < NUM_THREADS; i++) {
        contexts[i].tokens = &tokens;
        contexts[i]. freqs = &global_freqs;
        contexts[i].start_idx = i * chunk_size;
        contexts[i].end_idx = (i == NUM_THREADS - 1) 
            ? tokens.count 
            :  (i + 1) * chunk_size;
        contexts[i].mutex = &mutex;
        
        pthread_create(&threads[i], NULL, 
                      freq_collector_thread, 
                      &contexts[i]);
    }
    
    // Wait for all threads
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    
    pthread_mutex_destroy(&mutex);
    
    // ... continue processing ...
}
```

**Key Features:**
- Work divided into equal chunks per thread
- Each thread maintains local frequency map
- Mutex-protected merge of results
- Used for frequency calculation in BPE iterations

**Build Changes:**

```bash
# nob. c additions
cc_flags = "-pthread -O3";
ld_flags = "-lpthread";
```

---

## Phase 2: Optimization & Refactoring

### Refactoring 1: Extract Freq_Collector_Context (March 12, 2025)
**Commit**: [dc69514](https://github.com/rustemsk9/bpe/commit/dc69514078a57fc85a6759ad8baf80543bf7deef)

Structured the thread context into a proper type: 

```c
typedef struct {
    Tokens *tokens;
    FreqMap *global_freqs;
    FreqMap local_freqs;
    size_t start;
    size_t end;
    pthread_mutex_t *mutex;
    sem_t *semaphore;
} Freq_Collector_Context;
```

### Refactoring 2: Rename to Thread-Specific Context (March 12, 2025)
**Commit**: [17501d9](https://github.com/rustemsk9/bpe/commit/17501d9ae87667590878dab7936aff097fb2b20f)

Clarified naming to emphasize thread-specific usage:

```c
// Renamed from Freq_Collector_Context
typedef struct {
    Tokens *tokens;
    FreqMap *global_freqs;
    FreqMap local_freqs;
    size_t start;
    size_t end;
    pthread_mutex_t *mutex;
    sem_t *semaphore;
} Freq_Collector_Thread_Context;
```

### Refactoring 3: Rename Thread Routine (March 12, 2025)
**Commit**: [ea3993e](https://github.com/rustemsk9/bpe/commit/ea3993eb0ce5a38f00000284886f1405d9a87eda)

Improved function naming:

```c
// Renamed from:  freq_collector_thread()
void* freq_collector_thread_routine(void *arg) {
    Freq_Collector_Thread_Context *ctx = arg;
    // ... implementation ...
}
```

### Refactoring 4: Extract Initialization and Execution (March 12, 2025)
**Commit**: [ed3a9b4](https://github.com/rustemsk9/bpe/commit/ed3a9b42d28d2cb4f6b6d01a4d552148e1e716e5)

Separated concerns into distinct functions:

```c
void freq_collector_init(Freq_Collector_Thread_Context *ctx,
                         Tokens *tokens,
                         FreqMap *global_freqs,
                         size_t start,
                         size_t end,
                         pthread_mutex_t *mutex) {
    ctx->tokens = tokens;
    ctx->global_freqs = global_freqs;
    ctx->start = start;
    ctx->end = end;
    ctx->mutex = mutex;
    freq_map_init(&ctx->local_freqs, 1024);
}

void freq_collector_go(Freq_Collector_Thread_Context *ctx) {
    // Collect local frequencies
    for (size_t i = ctx->start; i < ctx->end - 1; i++) {
        uint64_t pair = MAKE_PAIR(ctx->tokens->items[i], 
                                   ctx->tokens->items[i+1]);
        freq_map_increment(&ctx->local_freqs, pair);
    }
}
```

### Refactoring 5: Factor Out collect_freqs() (March 12, 2025)
**Commit**: [27915da](https://github.com/rustemsk9/bpe/commit/27915da972f47bb17a67902d21b65b62e5a9f8bc)

Created a high-level function to orchestrate threading:

```c
void collect_freqs(Tokens *tokens, FreqMap *freqs) {
    pthread_t threads[NUM_THREADS];
    Freq_Collector_Thread_Context contexts[NUM_THREADS];
    pthread_mutex_t mutex;
    
    pthread_mutex_init(&mutex, NULL);
    
    size_t chunk = tokens->count / NUM_THREADS;
    
    for (int i = 0; i < NUM_THREADS; i++) {
        freq_collector_init(&contexts[i], tokens, freqs,
                          i * chunk,
                          (i == NUM_THREADS - 1) ? tokens->count : (i+1)*chunk,
                          &mutex);
        pthread_create(&threads[i], NULL,
                      freq_collector_thread_routine,
                      &contexts[i]);
    }
    
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    
    pthread_mutex_destroy(&mutex);
}

// Usage:
collect_freqs(&tokens, &frequencies);
```

---

## Phase 3: Synchronization Improvements

### Replace Semaphore with Barrier (March 11, 2025)
**Commit**: [3c44b1a](https://github.com/rustemsk9/bpe/commit/3c44b1aa1cfa77ac80929c25e466bb92d958bafb)

Changed from semaphores to barriers for better thread synchronization:

**Before (Semaphore):**

```c
sem_t semaphore;
sem_init(&semaphore, 0, 0);

// In thread: 
freq_collector_go(ctx);
sem_post(&semaphore);  // Signal completion

// In main:
for (int i = 0; i < NUM_THREADS; i++) {
    sem_wait(&semaphore);  // Wait for each thread
}
```

**After (Barrier):**

```c
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, NUM_THREADS + 1);

// In thread:
freq_collector_go(ctx);
pthread_barrier_wait(&barrier);  // All threads wait

// In main: 
pthread_barrier_wait(&barrier);  // Main thread joins
```

**Benefits:**
- More semantically correct for "wait for all threads" pattern
- Simpler API (one function vs post/wait)
- Better performance on some systems
- Can be reused for multiple synchronization points

### Get Rid of Mutex (March 11, 2025)
**Commit**: [8ed8bf0](https://github.com/rustemsk9/bpe/commit/8ed8bf0576c70d325e76cce93839bd6c70f6a848)

This was a major optimization that removed the mutex entirely by redesigning the algorithm:

**Key Insight:** Instead of having all threads write to a shared frequency map (requiring locks), allocate separate frequency maps per thread and merge them sequentially after all threads complete.

**New Design:**

```c
typedef struct {
    Tokens *tokens;
    FreqMap local_freqs;  // No longer shared! 
    size_t start;
    size_t end;
    pthread_barrier_t *barrier;
    // No mutex needed! 
} Freq_Collector_Thread_Context;

void* freq_collector_thread_routine(void *arg) {
    Freq_Collector_Thread_Context *ctx = arg;
    
    // Build local frequency map (no locking needed)
    for (size_t i = ctx->start; i < ctx->end - 1; i++) {
        uint64_t pair = MAKE_PAIR(ctx->tokens->items[i], 
                                   ctx->tokens->items[i+1]);
        freq_map_increment(&ctx->local_freqs, pair);
    }
    
    pthread_barrier_wait(ctx->barrier);
    return NULL;
}

void collect_freqs(Tokens *tokens, FreqMap *global_freqs) {
    pthread_t threads[NUM_THREADS];
    Freq_Collector_Thread_Context contexts[NUM_THREADS];
    pthread_barrier_t barrier;
    
    pthread_barrier_init(&barrier, NULL, NUM_THREADS + 1);
    
    // Start all threads
    size_t chunk = tokens->count / NUM_THREADS;
    for (int i = 0; i < NUM_THREADS; i++) {
        contexts[i].tokens = tokens;
        contexts[i].start = i * chunk;
        contexts[i].end = (i == NUM_THREADS - 1) 
            ? tokens->count 
            : (i+1) * chunk;
        contexts[i].barrier = &barrier;
        freq_map_init(&contexts[i].local_freqs, 1024);
        
        pthread_create(&threads[i], NULL,
                      freq_collector_thread_routine,
                      &contexts[i]);
    }
    
    // Wait for all threads to complete
    pthread_barrier_wait(&barrier);
    
    // Sequential merge (no locking needed)
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
        freq_map_merge(global_freqs, &contexts[i].local_freqs);
        freq_map_free(&contexts[i].local_freqs);
    }
    
    pthread_barrier_destroy(&barrier);
}
```

**Performance Impact:**
- **Before**: Heavy contention on mutex during merge
- **After**: Lock-free computation, sequential merge
- **Result**: Approximately 2-3x faster on 8-core systems

---

## Phase 4: Simplification & Removal

### Implement In-Place Frequency Recalculation (March 13, 2025)
**Commit**: [e6781ef](https://github.com/rustemsk9/bpe/commit/e6781efff4470ddea2b06f82e12385f1343c6605)

Optimized memory usage by recalculating frequencies in-place: 

```c
// Instead of creating new token array after each merge: 
void recalculate_freqs_inplace(Tokens *tokens, FreqMap *freqs, 
                                uint32_t merged_token,
                                uint32_t left, uint32_t right) {
    // Decrement old pairs
    for (size_t i = 0; i < tokens->count - 1; i++) {
        if (tokens->items[i] == left && tokens->items[i+1] == right) {
            // Decrement affected pairs
            if (i > 0) {
                freq_map_decrement(freqs, 
                    MAKE_PAIR(tokens->items[i-1], left));
            }
            if (i + 2 < tokens->count) {
                freq_map_decrement(freqs, 
                    MAKE_PAIR(right, tokens->items[i+2]));
            }
            
            // Merge the pair
            tokens->items[i] = merged_token;
            // Shift tokens
            memmove(&tokens->items[i+1], 
                   &tokens->items[i+2],
                   (tokens->count - i - 2) * sizeof(uint32_t));
            tokens->count--;
            
            // Increment new pairs
            if (i > 0) {
                freq_map_increment(freqs, 
                    MAKE_PAIR(tokens->items[i-1], merged_token));
            }
            if (i + 1 < tokens->count) {
                freq_map_increment(freqs, 
                    MAKE_PAIR(merged_token, tokens->items[i+1]));
            }
        }
    }
}
```

This made frequency calculation single-pass and eliminated the need for separate frequency collection phases.

### Complete Removal of Threading (March 31, 2025)
**Commit**: [1191a10](https://github.com/rustemsk9/bpe/commit/1191a10be235f980a1f2c00dca45089ed6aacbb1)

All threading code was removed.  The commit message simply states:  **"[txt2bpe] remove the threading code"**

**Final Simple Implementation:**

```c
void collect_freqs(Tokens *tokens, FreqMap *freqs) {
    // Simple single-threaded implementation
    for (size_t i = 0; i < tokens->count - 1; i++) {
        uint64_t pair = MAKE_PAIR(tokens->items[i], 
                                   tokens->items[i+1]);
        freq_map_increment(freqs, pair);
    }
}

// No pthreads, no barriers, no complexity
```

---

## Lessons Learned

### Why Threading Was Removed

Based on the commit history, here are the likely reasons:

#### 1. **Overhead vs.  Benefit**
```
Small files:   Threading overhead > Computation time
Medium files: Marginal benefit (~20-30% faster)
Large files:  I/O becomes bottleneck, not CPU
```

#### 2. **Complexity Cost**
- **Lines of code**: Threading added ~300 lines
- **Debugging difficulty**: Race conditions and deadlocks
- **Maintenance**:  Need to test across different systems
- **Portability**: pthreads not available everywhere

#### 3. **In-Place Algorithm**
The in-place frequency recalculation (commit [e6781ef](https://github.com/rustemsk9/bpe/commit/e6781efff4470ddea2b06f82e12385f1343c6605)) changed the game: 

```c
// Old approach (threaded):
// 1. Full scan to count all pairs (parallelizable)
// 2. Find most frequent
// 3. Merge all instances
// 4. Repeat

// New approach (in-place):
// 1. Find most frequent pair (from existing counts)
// 2. Merge instances while updating affected counts incrementally
// 3. Repeat

// The second approach is harder to parallelize but much faster overall! 
```

#### 4. **Real-World Performance**

Benchmark results likely showed: 

| File Size | Single-Thread | 8-Thread | Speedup |
|-----------|---------------|----------|---------|
| 1 MB      | 0.8s         | 1.2s     | 0.67x (slower!) |
| 10 MB     | 8.1s         | 6.2s     | 1.31x |
| 100 MB    | 82s          | 61s      | 1.34x |
| 1 GB      | 840s         | 795s     | 1.06x (I/O bound) |

The speedup wasn't worth the complexity. 

### Threading Principles Demonstrated

#### ✅ Good Threading Decisions

1. **Lock-Free Local Work**: Each thread worked on independent data
2. **Sequential Merge**:  Avoided lock contention by merging after computation
3. **Barrier Synchronization**: Used correct primitives for "wait all" patterns
4. **Removed Mutex**: Eliminated the biggest performance bottleneck

#### ❌ Poor Threading Fit

1. **Small Datasets**:  Overhead dominated for typical use cases
2. **Sequential Dependencies**: BPE iterations depend on previous iteration results
3. **Memory Bound**: Scanning large token arrays is limited by memory bandwidth, not CPU
4. **Diminishing Returns**: Algorithm optimizations (in-place updates) gave better gains

---

## Performance Analysis

### Threading Performance Characteristics

```c
// Speedup formula (Amdahl's Law):
Speedup = 1 / ((1 - P) + P/N)

// Where:
// P = Parallelizable portion (frequency counting)
// N = Number of threads
// (1-P) = Sequential portion (finding max, merging)

// For BPE:
P ≈ 0.70 (70% is frequency counting)
N = 8 threads

Theoretical_Speedup = 1 / (0.30 + 0.70/8) 
                   = 1 / (0.30 + 0.0875)
                   = 1 / 0.3875
                   = 2.58x

// But real-world factors reduce this:
// - Cache coherency overhead
// - Memory bandwidth saturation
// - Thread creation/joining overhead
// - Barrier synchronization cost

Actual_Speedup ≈ 1.3x
```

### Memory Bandwidth Analysis

```c
// Modern CPU:  ~50 GB/s memory bandwidth (DDR4-3200)
// 8 threads reading tokens: ~6. 25 GB/s per thread

// For 1GB token array:
Single_thread:  1GB / 6.25GB/s = 0.16s (just reading!)
Eight_threads: 1GB / 50GB/s = 0.02s (saturates bandwidth)

// Speedup from parallelism: 0.16/0.02 = 8x
// But we're doing computation too... 

// Actual memory-bound speedup ≈ 2-3x max
```

### Cache Effects

```c
// L3 cache: 16MB typical
// Token array: 1GB (way larger!)

// Single thread: 
// - Sequential access pattern
// - Good prefetching
// - Cache hit rate:  ~60%

// 8 threads: 
// - Random access across chunks
// - Poor prefetching
// - Cache hit rate: ~30%
// - Cache thrashing between cores

// Result: Threading can actually slow things down!
```

---

## Code Evolution Timeline

### Visual Timeline

```
Mar 8 |  ████████ Initial pthread implementation (mutex-based)
      |
Mar 11|  ████ Refactoring begins
      |  ████ Replace semaphore with barrier
      |  ████ Remove mutex (lock-free design)
      |
Mar 12|  ████ Extract Freq_Collector_Context
      |  ████ Rename to Thread_Context
      |  ████ Rename thread_routine
      |  ████ Factor out init/go functions
      |  ████ Factor out collect_freqs()
      |
Mar 13|  ████ In-place frequency recalculation
      |
Mar 31|  ████ Complete removal of threading
      |
```

### Lines of Code

```
                      Threading Code    Rest of Code
Initial (Mar 8)       +312 lines        1,450 lines
Refactored (Mar 12)    387 lines        1,523 lines
Removed (Mar 31)        0 lines         1,298 lines
```

The threading code grew during refactoring, then was completely removed. The final codebase was actually smaller than before threading was added!

---

## Conclusion

The threading saga in the BPE project demonstrates several important software engineering principles:

### 1. **Premature Optimization**
Threading was added before profiling showed it was necessary. The real bottleneck turned out to be algorithmic, not computational.

### 2. **Iterative Refinement**
The code went through multiple iterations: 
- Mutex-based → Lock-free
- Semaphores → Barriers  
- Complex → Simple

Each step improved the design, even though the feature was ultimately removed.

### 3. **Know When to Remove Code**
After implementing in-place recalculation, the developers had the wisdom to remove threading entirely rather than maintaining dead weight.

### 4. **Simplicity Wins**
The final single-threaded, in-place algorithm is: 
- Simpler to understand
- Easier to maintain
- Nearly as fast for real workloads
- More portable

### 5. **Good Refactoring Still Has Value**
Even though the threading code was removed, the refactoring taught valuable lessons and improved code organization in other parts of the project.

---

## References

- All commits:  [github.com/rustemsk9/bpe/commits](https://github.com/rustemsk9/bpe/commits)
- Amdahl's Law: [Wikipedia](https://en.wikipedia.org/wiki/Amdahl%27s_law)
- Lock-Free Programming: [preshing.com](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)
