# Tool: OSS-Fuzz

> Continuous fuzzing infrastructure for open source software, finding thousands of vulnerabilities automatically.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [google/oss-fuzz](https://github.com/google/oss-fuzz) |
| **Language** | Python, C/C++, Go, Rust, Java, Python |
| **Category** | Fuzzing Infrastructure |
| **Maintainer** | Google |

## Philosophy

**Core Insight**: Vulnerabilities hide in code paths that developers don't anticipate. By generating millions of random-but-guided inputs continuously, we can explore code paths humans never would.

**The Economics**: Security bugs found automatically cost orders of magnitude less than bugs found by attackers. OSS-Fuzz makes fuzzing economically viable at scale by:
1. Amortizing infrastructure costs across 1000+ projects
2. Running 24/7 on massive compute
3. Automating triage and reporting

**The Bet**: Most security bugs are reachable through automated input generation if you run long enough with good coverage guidance.

## Theoretical Foundations

### Coverage-Guided Fuzzing
The key innovation that makes modern fuzzing effective:

```
Traditional fuzzing:  random inputs → hope for crash
Coverage-guided:      random inputs → measure coverage → evolve toward new coverage
```

**Genetic Algorithm Analogy**:
- Population = corpus of interesting inputs
- Fitness = code coverage achieved
- Mutation = bit flips, insertions, deletions
- Selection = keep inputs that find new paths

### Information Theory Perspective
Fuzzing is a **search problem** in input space:
- Input space is astronomically large (2^n for n-bit inputs)
- Crash-inducing inputs are sparse
- Coverage feedback provides **gradient information** to guide search

### Kolmogorov Complexity
Effective fuzzers find inputs with **low complexity that trigger high complexity behavior**:
- Simple inputs that cause deep code paths
- Minimal reproducer = most informative bug report

## Academic Papers

| Paper | Authors | Year | Key Contribution |
|-------|---------|------|------------------|
| [AFL: Technical Whitepaper](https://lcamtuf.coredump.cx/afl/technical_details.txt) | Michał Zalewski | 2014 | Coverage-guided fuzzing with genetic algorithms |
| [OSS-Fuzz: Five Years of Continuous Fuzzing](https://storage.googleapis.com/gweb-research2023-media/pubtools/pdf/928df8c4f1f7ed7dab2b8a3d3b34b61a4bfc.pdf) | Google | 2021 | Scale and impact analysis |
| [FuzzBench: Fuzzer Benchmarking](https://dl.acm.org/doi/10.1145/3468264.3468536) | Metzman et al. | 2021 | Rigorous fuzzer comparison methodology |
| [Evaluating Fuzz Testing](https://www.cs.umd.edu/~mwh/papers/fuzzeval.pdf) | Klees et al. | 2018 | Statistical rigor in fuzzing evaluation |
| [NEZHA: Differential Fuzzing](https://ieeexplore.ieee.org/document/7958601) | Petsios et al. | 2017 | Finding bugs via behavioral differences |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            OSS-FUZZ INFRASTRUCTURE                       │
│                                                                         │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐               │
│  │   Project   │     │   Project   │     │   Project   │   ...1000+    │
│  │  Dockerfile │     │  Dockerfile │     │  Dockerfile │               │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘               │
│         │                   │                   │                       │
│         └─────────────┬─────┴───────────────────┘                       │
│                       ▼                                                 │
│              ┌─────────────────┐                                       │
│              │  ClusterFuzz    │ ← Orchestration layer                 │
│              │  (scheduling,   │                                       │
│              │   dedup, triage)│                                       │
│              └────────┬────────┘                                       │
│                       │                                                 │
│         ┌─────────────┼─────────────┐                                  │
│         ▼             ▼             ▼                                  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐                            │
│  │ libFuzzer │ │    AFL    │ │ Honggfuzz │  ← Fuzzing engines         │
│  └───────────┘ └───────────┘ └───────────┘                            │
│         │             │             │                                  │
│         └─────────────┴─────────────┘                                  │
│                       │                                                 │
│              ┌────────┴────────┐                                       │
│              │   Sanitizers    │ ← Bug detection                       │
│              │ ASan/MSan/UBSan │                                       │
│              └────────┬────────┘                                       │
│                       │                                                 │
│                       ▼                                                 │
│              ┌─────────────────┐                                       │
│              │  Bug Tracker    │ → Monorail/GitHub Issues              │
│              │  (auto-file,    │                                       │
│              │   auto-close)   │                                       │
│              └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

## The Mathematics of Fuzzing

### Coverage Explosion Problem
For a program with `n` branches:
- Theoretical paths: up to `2^n`
- Practical paths: much fewer due to constraints
- Fuzzer's job: explore as many **feasible** paths as possible

### Mutation Strategies
```
BIT FLIP:     input[i] ^= (1 << j)           # Flip single bit
BYTE FLIP:    input[i] ^= 0xFF               # Flip entire byte  
ARITHMETIC:   input[i] += delta              # Add/subtract small values
INTERESTING:  input[i] = {0, 1, -1, MAX, MIN} # Boundary values
HAVOC:        Combine multiple mutations     # Random chaos
SPLICE:       Combine two corpus entries     # Crossover
```

### Corpus Distillation
Given corpus C, find minimal subset C' such that:
- coverage(C') = coverage(C)
- |C'| is minimized

This is a **set cover problem** (NP-hard), solved greedily.

## Usage

### Adding a Project
```bash
# Clone OSS-Fuzz
git clone https://github.com/google/oss-fuzz

# Create project structure
mkdir projects/myproject
```

### Project Structure
```
projects/myproject/
├── Dockerfile          # Build environment
├── build.sh           # Compile fuzz targets
├── project.yaml       # Metadata
└── fuzz_*.cc          # Fuzz target source (optional)
```

### Dockerfile
```dockerfile
FROM gcr.io/oss-fuzz-base/base-builder

RUN apt-get update && apt-get install -y cmake
RUN git clone --depth 1 https://github.com/org/project $SRC/project

COPY build.sh $SRC/
WORKDIR $SRC/project
```

### build.sh
```bash
#!/bin/bash -eu

mkdir build && cd build
cmake .. \
  -DCMAKE_C_COMPILER=$CC \
  -DCMAKE_CXX_COMPILER=$CXX \
  -DCMAKE_C_FLAGS="$CFLAGS" \
  -DCMAKE_CXX_FLAGS="$CXXFLAGS"
  
make -j$(nproc)

# Build fuzz target
$CXX $CXXFLAGS $LIB_FUZZING_ENGINE \
  fuzz_target.cc -o $OUT/fuzz_target \
  libproject.a
```

### Local Testing
```bash
# Build locally
python infra/helper.py build_image myproject
python infra/helper.py build_fuzzers myproject

# Run fuzzer
python infra/helper.py run_fuzzer myproject fuzz_target

# Check coverage
python infra/helper.py coverage myproject
```

## Key Metrics (as of 2024)

| Metric | Value |
|--------|-------|
| Projects integrated | 1,000+ |
| Bugs found | 10,000+ |
| Security vulnerabilities | 3,500+ |
| CPU cores running | 100,000+ |
| Languages supported | C/C++, Go, Rust, Java, Python, Swift |

## When to Use

- Open source project accepting contributions
- C/C++/Go/Rust project with parsers, decoders, protocol handlers
- Project handling untrusted input
- Want continuous, free fuzzing infrastructure

## Limitations

- **Open source only** (or Google-partnered)
- **Build complexity** — Projects must build with Clang + sanitizers
- **Target quality** — Garbage in, garbage out; need good fuzz targets
- **Language support** — Best for C/C++; other languages have limitations

## Related Tools

- **ClusterFuzz** — The orchestration layer OSS-Fuzz builds on
- **libFuzzer** — Default fuzzing engine
- **AFL++** — Alternative fuzzing engine
- **Honggfuzz** — Another alternative engine
- **FuzzBench** — Benchmarking fuzzer effectiveness

## References

- [OSS-Fuzz Documentation](https://google.github.io/oss-fuzz/)
- [Ideal Integration Guide](https://google.github.io/oss-fuzz/getting-started/new-project-guide/)
- [FuzzBench](https://google.github.io/fuzzbench/)
- [Google Security Blog: OSS-Fuzz Posts](https://security.googleblog.com/search/label/fuzzing)
