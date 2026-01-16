# BPE Project - Commit History

A comprehensive overview of the development history of the BPE (Byte Pair Encoding) project, organized by development themes and features.

---

## 📋 Table of Contents
- [Project Initialization](#project-initialization)
- [Project Structure & Refactoring](#project-structure--refactoring)
- [Core BPE Model Generation](#core-bpe-model-generation)
- [Frequency & Data Management](#frequency--data-management)
- [Inspection & Debugging Tools](#inspection--debugging-tools)
- [Performance & Threading](#performance--threading)
- [Build System](#build-system)
- [Code Quality & Fixes](#code-quality--fixes)
- [Documentation & References](#documentation--references)

---

## Project Initialization

### Ready. Set. Go! 
**Date**: 2025-03-07  
**Commit**: [82a39c0](https://github.com/rustemsk9/bpe/commit/82a39c01ec0c821e97cb96e424d8fe1e5ba281ee)

Initial commit - project creation and basic setup. 

---

## Project Structure & Refactoring

### Introduce folders src/ and thirdparty/
**Date**: 2025-03-07  
**Commit**: [d6c6184](https://github.com/rustemsk9/bpe/commit/d6c6184b92e8dad2307f5d65922e1592b64105bf)

Organized the codebase into logical directories:
- `src/` for source files
- `thirdparty/` for external dependencies

### bpe → txt2bpe
**Date**:  2025-03-07  
**Commit**: [743695b](https://github.com/rustemsk9/bpe/commit/743695bec630ec4023d0a19752c5220bc1a8f5fb)

Renamed the main tool to `txt2bpe` to better reflect its purpose (converting text files to BPE format).

### bin → bpe
**Date**: 2025-03-08  
**Commit**: [68f2ee5](https://github.com/rustemsk9/bpe/commit/68f2ee5cb9dd28354e89cc53dbd36069297d2431)

Changed file extension from `.bin` to `.bpe` for tokenized output files.

### Move dump_pairs and Tokens to common bpe unit
**Date**: 2025-03-11  
**Commit**: [10bc0ba](https://github.com/rustemsk9/bpe/commit/10bc0ba20037989d7a5875994bff2d6c5413f0f9)

Refactored common functionality into a shared BPE module for better code reuse.

```c
// Moved to shared bpe.h
void dump_pairs(... );
typedef struct Tokens { ... } Tokens;
```

### Move render_token() to shared bpe unit
**Date**: 2025-03-11  
**Commit**: [fb82437](https://github.com/rustemsk9/bpe/commit/fb8243780223e7eef0a6c4b2b4a8f585e7502ac7)

Consolidated token rendering functionality into the shared library.

### Factor out swap() operation to bpe. h
**Date**: 2025-03-22  
**Commit**: [91d6867](https://github.com/rustemsk9/bpe/commit/91d686769cc1ce3582343b1e1516c0dd2837df90)

Extracted generic swap operation to header file for reusability across tools.

### Factor out c_strlit_escape_bytes()
**Date**: 2025-03-22  
**Commit**: [564d955](https://github.com/rustemsk9/bpe/commit/564d95541632b9a3b9beea882c55b96c84c86e9d)

Created utility function for escaping bytes in C string literals.

---

## Core BPE Model Generation

### Introduce bpe_gen
**Date**: 2025-03-20  
**Commit**: [98e51a5](https://github.com/rustemsk9/bpe/commit/98e51a58116d0ff38801649229a3338cf3841abe)

Introduced `bpe_gen` tool for generating text using trained BPE models.

### [bpe_gen] take into account frequencies
**Date**: 2025-04-04  
**Commit**: [98d3c9e](https://github.com/rustemsk9/bpe/commit/98d3c9e56cfb1fb6b06b99afa5268256a0fc7deb)

Enhanced `bpe_gen` to use token frequencies for better generation quality.

```c
// Now considers pair frequencies when generating
if (pair->freq > threshold) {
    // Use this pair
}
```

### Include frequency into the Pair definition
**Date**: 2025-03-31  
**Commit**: [93274dd](https://github.com/rustemsk9/bpe/commit/93274dd3eeb8aaf70bb963b887a66634dc2cef51)

Modified the `Pair` structure to include frequency information: 

```c
typedef struct {
    uint32_t left;
    uint32_t right;
    uint32_t freq;  // Added frequency field
} Pair;
```

### [txt2bpe] save the frequencies in the bpe file
**Date**: 2025-03-31  
**Commit**:  [769f50d](https://github.com/rustemsk9/bpe/commit/769f50d21f091f8a2e40f0e27d69081b0a23591d)

Modified the BPE file format to store frequency information for each token pair.

### Added random seed flag
**Date**: 2025-03-21  
**Contributor**:  Iselgrey  
**Commit**: [5b1c3cd](https://github.com/rustemsk9/bpe/commit/5b1c3cd08ab474820b5f0da24162516281ae9db4)

Added `-seed` flag for reproducible random generation.

```bash
./bpe_gen -seed 42 model.bpe
```

---

## Frequency & Data Management

### [txt2bpe] implement in-place frequency recalculation
**Date**: 2025-03-13  
**Commit**: [e6781ef](https://github.com/rustemsk9/bpe/commit/e6781efff4470ddea2b06f82e12385f1343c6605)

Optimized memory usage by recalculating frequencies in-place instead of creating new structures.

### [txt2bpe] factor out collect_freqs()
**Date**: 2025-03-12  
**Commit**: [27915da](https://github.com/rustemsk9/bpe/commit/27915da972f47bb17a67902d21b65b62e5a9f8bc)

Extracted frequency collection into a separate function for better modularity.

```c
void collect_freqs(const Tokens *tokens, FreqMap *freqs) {
    // Frequency collection logic
}
```

### [txt2bpe] factor out freq_collector_init() and freq_collector_go()
**Date**: 2025-03-12  
**Commit**: [ed3a9b4](https://github.com/rustemsk9/bpe/commit/ed3a9b42d28d2cb4f6b6d01a4d552148e1e716e5)

Split frequency collector into initialization and execution functions.

### [txt2bpe] Factor out Freq_Collector_Context
**Date**: 2025-03-12  
**Commit**:  [dc69514](https://github.com/rustemsk9/bpe/commit/dc69514078a57fc85a6759ad8baf80543bf7deef)

Created context structure for frequency collection state.

```c
typedef struct {
    Tokens *tokens;
    FreqMap *freqs;
    size_t start;
    size_t end;
} Freq_Collector_Context;
```

### [txt2bpe] Freq_Collector_Context → Freq_Collector_Thread_Context
**Date**: 2025-03-12  
**Commit**: [17501d9](https://github.com/rustemsk9/bpe/commit/17501d9ae87667590878dab7936aff097fb2b20f)

Renamed to clarify thread-specific usage.

### [txt2bpe] freq_collector_thread → freq_collector_thread_routine
**Date**: 2025-03-12  
**Commit**: [ea3993e](https://github.com/rustemsk9/bpe/commit/ea3993eb0ce5a38f00000284886f1405d9a87eda)

Renamed for clearer function naming convention.

---

## Inspection & Debugging Tools

### Introduce bpe_inspect
**Date**: 2025-03-08  
**Commit**: [f2a2db4](https://github.com/rustemsk9/bpe/commit/f2a2db46d9af0ecbc894f382ac64508037d18631)

Created `bpe_inspect` tool for examining BPE model files.

```bash
./bpe_inspect model. bpe
```

### [bpe_inspect] print rendered tokens as properly escaped C strings
**Date**: 2025-03-08  
**Commit**: [dc8dce2](https://github.com/rustemsk9/bpe/commit/dc8dce224463f7581080cf89d8e94e5ba94d6d6c)

Improved output formatting to show tokens as valid C string literals.

### [bpe_inspect] introduce --no-ids
**Date**: 2025-03-11  
**Commit**: [154e23d](https://github.com/rustemsk9/bpe/commit/154e23d02f72426f9254698f67b4f362edf7db16)

Added flag to hide token IDs in output.

### [bpe] stronger verification for load_pairs()
**Date**: 2025-03-11  
**Commit**: [e9fe9ed](https://github.com/rustemsk9/bpe/commit/e9fe9edeef795a86a87d962829e8d9a79e366871)

Enhanced validation when loading BPE pair data from files.

### Introduce tkn_inspect
**Date**: 2025-03-13  
**Commit**: [ebfded1](https://github.com/rustemsk9/bpe/commit/ebfded114c9906c4aded4d8849c2a663fb08ba45)

Created `tkn_inspect` tool for inspecting tokenized files.

### [tkn_inspect] add -ids, -render and -split flags
**Date**: 2025-03-13  
**Commit**: [be7978e](https://github.com/rustemsk9/bpe/commit/be7978e173863cdfe631dfe784654649bd9acd06)

Added multiple display options: 
- `-ids`: Show token IDs
- `-render`: Render tokens as text
- `-split`: Split output by tokens

```bash
./tkn_inspect -ids -render tokens.tkn
```

### Document the tools of the suite
**Date**: 2025-03-22  
**Commit**: [a1cadcb](https://github.com/rustemsk9/bpe/commit/a1cadcb68c3522f17f8a70e84c18f4b4496d9de8)

Added comprehensive documentation for all tools in the suite.

---

## Performance & Threading

### Introduce multithreading with pthreads
**Date**: 2025-03-08  
**Commit**: [045be11](https://github.com/rustemsk9/bpe/commit/045be11574fda1c9ea1ab1998e35ba6b19ccc2f5)

Added POSIX threads support for parallel frequency calculation.

```c
#include <pthread.h>

pthread_t threads[NUM_THREADS];
for (int i = 0; i < NUM_THREADS; i++) {
    pthread_create(&threads[i], NULL, freq_collector_thread_routine, &contexts[i]);
}
```

### [txt2bpe] replace semaphore with barrier
**Date**: 2025-03-11  
**Commit**: [3c44b1a](https://github.com/rustemsk9/bpe/commit/3c44b1aa1cfa77ac80929c25e466bb92d958bafb)

Improved thread synchronization using barriers instead of semaphores.

```c
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, NUM_THREADS);
```

### [txt2bpe] Get rid of mutex
**Date**: 2025-03-11  
**Commit**: [8ed8bf0](https://github.com/rustemsk9/bpe/commit/8ed8bf0576c70d325e76cce93839bd6c70f6a848)

Removed mutex locks by redesigning thread-safe data structures, improving performance.

### [txt2bpe] remove the threading code
**Date**: 2025-03-31  
**Commit**: [1191a10](https://github.com/rustemsk9/bpe/commit/1191a10be235f980a1f2c00dca45089ed6aacbb1)

Simplified the codebase by removing threading complexity (likely replaced with better approach or proved unnecessary for typical use cases).

---

## Build System

### [nob] build all the tools in parallel
**Date**: 2025-03-08  
**Commit**: [fbf6b08](https://github.com/rustemsk9/bpe/commit/fbf6b085076b6795792bad42cf16bc3434f3a1ce)

Optimized build process using parallel compilation with nob build system.

### [nob] enable running tools from nob
**Date**: 2025-03-11  
**Commit**: [c8d4ae5](https://github.com/rustemsk9/bpe/commit/c8d4ae59c81546164d842db8b24846a873bef682)

Extended nob to allow running built tools directly. 

```bash
./nob txt2bpe input.txt output/
```

### Upgrade nob
**Multiple dates**: 2025-03-22, 2025-03-25  
**Commits**: 
- [333050b](https://github.com/rustemsk9/bpe/commit/333050b1481ed7fdbbad6e5110be81e50da9f0f9)
- [ccb32f3](https://github.com/rustemsk9/bpe/commit/ccb32f3e15bfa963f8cd7024fd91352f2e5c4d79)

Updated nob build system to latest versions.

---

## Code Quality & Fixes

### Fix bug with casting signed chars to unsigned 32bit integers
**Date**: 2025-03-11  
**Commit**: [9cfa19a](https://github.com/rustemsk9/bpe/commit/9cfa19a9afa72db41b30233eb7a7988935c0bb94)

Fixed critical type casting bug: 

```c
// Before (bug):
uint32_t token = (uint32_t)signed_char;

// After (fixed):
uint32_t token = (uint32_t)(unsigned char)signed_char;
```

### [txt2bpe] fix memory leak
**Date**: 2025-03-13  
**Commit**: [da08c47](https://github.com/rustemsk9/bpe/commit/da08c47080737b5432d1993a7263a4f5c29d4523)

Fixed memory leak in frequency calculation. 

### [txt2bpe] fix usage
**Date**: 2025-03-13  
**Commit**:  [ea36fb5](https://github.com/rustemsk9/bpe/commit/ea36fb5cbbb791f5f96b9953cba058a4df365866)

Corrected usage message for txt2bpe tool.

### Factor out usage
**Date**: 2025-03-08  
**Commit**: [5fe54b3](https://github.com/rustemsk9/bpe/commit/5fe54b3f9c2b3c7a040c7910c90e7093ecd2e465)

Extracted usage/help text into separate function.

### TODO: support unicode
**Date**: 2025-03-08  
**Commit**: [d511ab5](https://github.com/rustemsk9/bpe/commit/d511ab5a2d73ad7729446cc01c156f7cf74296c5)

Added TODO comment about Unicode support.

### Extend upon the current unicode situation
**Date**: 2025-03-25  
**Commit**: [d48a0f2](https://github.com/rustemsk9/bpe/commit/d48a0f2210186d687d79d7af616ad918261a8303)

Documented the current state of Unicode handling in the codebase.

### TODOs
**Multiple dates**: 2025-03-08  
**Commits**:
- [90a7988](https://github.com/rustemsk9/bpe/commit/90a7988c8c73ab7e51c80df18f18d1ee7a3a06d4)
- [307c08a](https://github.com/rustemsk9/bpe/commit/307c08acd6e458da4022f1a1f1e81928247950d2)

Added TODO comments for future improvements.

---

## Command Line Interface & Configuration

### [txt2bpe] report progress
**Date**: 2025-03-08  
**Commit**: [369bc5e](https://github.com/rustemsk9/bpe/commit/369bc5e8fe3880e16a7f4e74f4091d18c9213087)

Added progress reporting during BPE training.

```
Processing:  45%  [=========>          ]
```

### [txt2bpe] add -report-freq flag
**Date**: 2025-03-12  
**Commit**: [ee3afe8](https://github.com/rustemsk9/bpe/commit/ee3afe8713d5bc824139c8b1e99e9c095cc07d3e)

Added flag to control how often progress is reported.

```bash
./txt2bpe -report-freq 1000 input.txt output/
```

### [txt2bpe] dump the state and customize termination frequency
**Date**: 2025-03-12  
**Commit**: [4a92888](https://github.com/rustemsk9/bpe/commit/4a92888e43a7d8ca51c4f441a2ee379a02e81773)

Added state dumping and customizable termination criteria.

### [txt2bpe] add -max-iterations flag
**Date**: 2025-03-12  
**Commit**: [97bf5a3](https://github.com/rustemsk9/bpe/commit/97bf5a3422c76059102ab4ee7f314a1ed7d9b7e3)

Added flag to limit the number of BPE merge iterations.

```bash
./txt2bpe -max-iterations 10000 input.txt output/
```

### [txt2bpe] do not proceed if output_dir already exists
**Date**: 2025-03-12  
**Commit**: [e4cc541](https://github.com/rustemsk9/bpe/commit/e4cc5414a00d4cf9bceea7fd780a19bb63284b50)

Added safety check to prevent overwriting existing output directories.

### [txt2bpe] clean up command line parsing
**Date**: 2025-03-12  
**Commit**: [794765d](https://github.com/rustemsk9/bpe/commit/794765d2e76aa87024bef4640488fc21a19c61a9)

Refactored and improved command-line argument parsing.

### [txt2bpe] make profiling part of the reporting
**Date**: 2025-03-13  
**Commit**: [a9f8217](https://github.com/rustemsk9/bpe/commit/a9f82171d3b457ecfcd2a5f05fe483bf51fdaf97)

Integrated performance profiling into progress reports.

```
Iteration 1000: 45.2s (freqs: 12.3s, merge: 32.9s)
```

---

## Documentation & References

### Add wikipedia article to the references
**Date**: 2025-03-07  
**Commit**: [282789f](https://github.com/rustemsk9/bpe/commit/282789fb59e39ccfaa7da6cf5d17a1be7824066d)

Added reference to Wikipedia article on Byte Pair Encoding in README.

---

## Development Timeline Summary

**Total Commits**: 50+  
**Development Period**: March 2025 - April 2025  
**Key Contributors**: rexim (primary), Iselgrey

### Major Milestones: 

1. **Week 1 (Mar 7-8)**: Project initialization, basic structure, and first tools
2. **Week 2 (Mar 11-13)**: Threading implementation, inspection tools, refactoring
3. **Week 3 (Mar 20-22)**: Code generation, documentation, utilities
4. **Week 4 (Mar 25)**: Build system improvements, Unicode considerations
5. **Week 5 (Mar 31 - Apr 4)**: Frequency management, threading simplification

---

## View More

**Note**: This history includes 50+ commits.  For the complete commit history with all details, visit: 
[https://github.com/rustemsk9/bpe/commits](https://github.com/rustemsk9/bpe/commits)
