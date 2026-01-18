# Tools

> Deep documentation of security tools — the philosophy, mathematics, and research behind them.

## What Are Tool Files?

Unlike skill files (which capture *how to think*), tool files capture:

- **Philosophy** — The core insight that makes the tool work
- **Theoretical Foundations** — The CS, math, or security concepts underneath
- **Academic Papers** — Research that informed or validated the tool
- **Architecture** — How the tool works internally
- **Usage** — Practical application

## Why This Matters

Understanding *why* a tool works makes you:
1. **Better at using it** — Know its strengths and limitations
2. **Better at extending it** — Write plugins, detectors, fuzz targets
3. **Better at building alternatives** — Apply concepts elsewhere
4. **Better at defense** — Understand what attackers use

## Categories

### Fuzzing
Tools for automated vulnerability discovery through input generation.

| Tool | Purpose |
|------|---------|
| [OSS-Fuzz](fuzzing/oss-fuzz.md) | Continuous fuzzing infrastructure |
| [libFuzzer](fuzzing/libfuzzer.md) | In-process coverage-guided fuzzer |
| [syzkaller](fuzzing/syzkaller.md) | Kernel syscall fuzzer |

**Key Concepts**: Coverage-guided evolution, mutation strategies, corpus management

### Sanitizers
Runtime instrumentation for detecting memory and concurrency bugs.

| Tool | Purpose |
|------|---------|
| [AddressSanitizer](sanitizers/addresssanitizer.md) | Memory error detection |

**Key Concepts**: Shadow memory, redzones, quarantine

### Cryptography
Libraries and testing tools for secure cryptographic implementations.

| Tool | Purpose |
|------|---------|
| [Tink](cryptography/tink.md) | Secure-by-default crypto library |
| [Wycheproof](cryptography/wycheproof.md) | Crypto implementation test vectors |

**Key Concepts**: Primitive-based design, secure defaults, edge-case testing

### Network Security
Scanners and detection tools for network-level vulnerabilities.

| Tool | Purpose |
|------|---------|
| [Tsunami](network-security/tsunami.md) | High-confidence vulnerability scanner |

**Key Concepts**: Precision over coverage, verified exploitation

## Reading a Tool File

Each tool file follows this structure:

1. **Overview** — Quick facts (repo, language, maintainer)
2. **Philosophy** — The core insight
3. **Theoretical Foundations** — Underlying concepts
4. **Academic Papers** — Research foundation
5. **Architecture** — How it works
6. **Usage** — How to use it
7. **Limitations** — What it can't do
8. **References** — Further reading

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines. Tool files should:

- Explain the *why*, not just the *how*
- Include academic references where possible
- Document the mathematical/theoretical foundations
- Provide practical usage examples
