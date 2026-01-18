# Tool: libFuzzer

> In-process, coverage-guided fuzzing engine built into LLVM/Clang.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [llvm/llvm-project](https://github.com/llvm/llvm-project/tree/main/compiler-rt/lib/fuzzer) |
| **Language** | C++ |
| **Category** | Fuzzing Engine |
| **Maintainer** | LLVM Project (Google-originated) |

## Philosophy

**Core Insight**: Fuzzing is most effective when tightly integrated with the compiler. By instrumenting code at compile time, we get:
1. Precise coverage feedback
2. Minimal runtime overhead
3. No external dependencies

**In-Process Model**: Unlike AFL's fork-based model, libFuzzer runs the target in the same process:
- Faster: No fork() overhead per execution
- Simpler: Single binary, no external tooling
- Trade-off: Crashes kill the fuzzer (needs restart logic)

**The Harness Contract**: Developer writes a simple function, libFuzzer does everything else:
```c
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size);
```

## Theoretical Foundations

### Edge Coverage
libFuzzer tracks **edge coverage**, not just basic block coverage:

```
Basic Block A ──→ Basic Block B ──→ Basic Block C
                        ↓
                  Basic Block D

Edges: A→B, B→C, B→D
```

**Why edges matter**: Two different paths through the same blocks represent different program behaviors.

### Coverage Representation
```c
// Simplified: Each edge gets a counter
uint8_t edge_counters[65536];

// Edge ID = hash(caller_block, callee_block)
edge_counters[edge_id]++;

// Counters are bucketed: 1, 2, 3, 4-7, 8-15, 16-31, 32-127, 128+
// This captures "executed once" vs "executed many times"
```

### Corpus Evolution Algorithm
```
1. Start with seed corpus (or empty)
2. Pick input from corpus (weighted by recency/size)
3. Mutate input
4. Execute target with mutated input
5. If new coverage discovered:
   - Add to corpus
   - Minimize if possible
6. If crash:
   - Save crash input
   - Minimize reproducer
7. Goto 2
```

## Academic Papers

| Paper | Authors | Year | Key Contribution |
|-------|---------|------|------------------|
| [libFuzzer: A Library for Coverage-Guided Fuzz Testing](https://llvm.org/docs/LibFuzzer.html) | LLVM | 2015 | Core design |
| [Coverage-Directed Greybox Fuzzing](https://dl.acm.org/doi/10.1145/2976749.2978428) | Böhme et al. | 2016 | AFLFast improvements to mutation scheduling |
| [REDQUEEN: Fuzzing with Input-to-State Correspondence](https://www.ndss-symposium.org/ndss-paper/redqueen-fuzzing-with-input-to-state-correspondence/) | Aschermann et al. | 2019 | Magic byte handling |
| [Angora: Efficient Fuzzing by Principled Search](https://ieeexplore.ieee.org/document/8418633) | Chen & Chen | 2018 | Gradient-descent for constraint solving |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        libFuzzer Process                        │
│                                                                 │
│  ┌─────────────┐                                               │
│  │   Corpus    │ ← Interesting inputs                          │
│  │  (in-memory │                                               │
│  │   + disk)   │                                               │
│  └──────┬──────┘                                               │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                               │
│  │  Scheduler  │ ← Pick next input to mutate                   │
│  │  (energy-   │                                               │
│  │   based)    │                                               │
│  └──────┬──────┘                                               │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐     ┌─────────────────────────────────────┐  │
│  │  Mutator    │ ──▶ │  LLVMFuzzerTestOneInput(data, size) │  │
│  │  (11+ ops)  │     │                                     │  │
│  └─────────────┘     │  ┌─────────────────────────────┐   │  │
│                      │  │     Target Function          │   │  │
│                      │  │  (instrumented by Clang)     │   │  │
│                      │  └───────────────┬─────────────┘   │  │
│                      └──────────────────┼─────────────────┘  │
│                                         │                     │
│                                         ▼                     │
│                             ┌─────────────────┐              │
│                             │ Coverage Map    │              │
│                             │ (edge counters) │              │
│                             └────────┬────────┘              │
│                                      │                        │
│                          New edges? ─┼─ Yes → Add to corpus   │
│                                      │                        │
│                          Crash? ─────┼─ Yes → Save & minimize │
│                                      │                        │
└──────────────────────────────────────┴────────────────────────┘
```

## Mutation Operations

libFuzzer applies these mutations (and combinations):

| Mutation | Description | Example |
|----------|-------------|---------|
| **EraseBytes** | Delete random bytes | `ABCDEF` → `ABEF` |
| **InsertByte** | Insert random byte | `ABCD` → `ABXCD` |
| **InsertRepeatedBytes** | Insert repeated byte | `AB` → `AXXXXB` |
| **ChangeByte** | Change one byte | `ABCD` → `ABXD` |
| **ChangeBit** | Flip one bit | `0x41` → `0x40` |
| **ShuffleBytes** | Reorder bytes | `ABCD` → `BADC` |
| **ChangeASCIIInteger** | Modify ASCII number | `"123"` → `"124"` |
| **ChangeBinaryInteger** | Modify binary int | Increment/decrement |
| **CopyPart** | Copy within input | `ABCD` → `ABABCD` |
| **CrossOver** | Combine two inputs | `AB` + `XY` → `AXY` |
| **Dictionary** | Insert known tokens | Insert `"<html>"` |

## Usage

### Basic Fuzz Target
```cpp
#include <stdint.h>
#include <stddef.h>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // Call your code with fuzzer-generated data
    parse_input(data, size);
    return 0;  // Always return 0
}
```

### Compile & Run
```bash
# Compile with coverage + sanitizer
clang++ -fsanitize=fuzzer,address -g fuzz_target.cc -o fuzzer

# Run
./fuzzer corpus_dir/

# With options
./fuzzer corpus_dir/ \
    -max_len=10000 \
    -timeout=5 \
    -jobs=8 \
    -workers=8 \
    -dict=my.dict
```

### Key Options
```bash
-max_len=N          # Max input size (default: 4096)
-timeout=N          # Seconds per input (default: 1200)
-max_total_time=N   # Total fuzzing time in seconds
-jobs=N             # Number of fuzzing jobs
-workers=N          # Number of worker processes
-dict=FILE          # Dictionary of tokens
-seed=N             # Random seed for reproducibility
-only_ascii=1       # Only generate ASCII inputs
-artifact_prefix=X  # Prefix for crash files
-print_final_stats  # Print stats at exit
```

### Using FuzzedDataProvider
```cpp
#include <fuzzer/FuzzedDataProvider.h>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    FuzzedDataProvider fdp(data, size);
    
    // Consume structured data from fuzzer input
    int x = fdp.ConsumeIntegral<int>();
    std::string s = fdp.ConsumeRandomLengthString(100);
    bool flag = fdp.ConsumeBool();
    std::vector<uint8_t> bytes = fdp.ConsumeBytes<uint8_t>(10);
    
    my_function(x, s, flag, bytes);
    return 0;
}
```

## The Mathematics of Corpus Management

### Feature Minimization
For each feature (edge, edge count bucket), keep only the smallest input that exercises it:

```
Feature F1 exercised by: {input_a (1000 bytes), input_b (50 bytes)}
Keep: input_b

Result: Minimal corpus with maximum coverage
```

### Merge Algorithm
```bash
# Merge new findings into existing corpus
./fuzzer -merge=1 corpus_minimal/ corpus_new/

# This:
# 1. Runs all inputs from both directories
# 2. Keeps only inputs that contribute unique coverage
# 3. Prefers smaller inputs
```

## When to Use

- C/C++ code with well-defined entry points
- Projects using Clang/LLVM
- Need fast, in-process fuzzing
- OSS-Fuzz integration (default engine)
- When AFL's fork overhead is too high

## Limitations

- **Clang-only** — Requires LLVM toolchain
- **In-process** — Crashes kill fuzzer, need restart wrapper
- **Single-threaded target** — Target function should be thread-safe or single-threaded
- **No network fuzzing** — Designed for library fuzzing, not network protocols

## Related Tools

- **AFL++** — Fork-based alternative, more features
- **Honggfuzz** — Another in-process alternative
- **OSS-Fuzz** — Infrastructure that uses libFuzzer
- **ClusterFuzz** — Orchestration for libFuzzer at scale

## References

- [Official libFuzzer Documentation](https://llvm.org/docs/LibFuzzer.html)
- [libFuzzer Tutorial](https://github.com/google/fuzzing/blob/master/tutorial/libFuzzerTutorial.md)
- [Structure-Aware Fuzzing](https://github.com/google/fuzzing/blob/master/docs/structure-aware-fuzzing.md)
