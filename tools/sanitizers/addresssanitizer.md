# Tool: AddressSanitizer (ASan)

> Fast memory error detector for C/C++, finding buffer overflows, use-after-free, and more.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [google/sanitizers](https://github.com/google/sanitizers) |
| **Language** | C++ (compiler + runtime) |
| **Category** | Dynamic Analysis / Sanitizer |
| **Maintainer** | Google / LLVM |

## Philosophy

**Core Insight**: Memory errors are among the most exploitable bugs. By making every memory access checkable at low cost, we can catch bugs that traditional testing misses.

**The Trade-off**: 
- ~2x slowdown (acceptable for testing)
- ~2-3x memory overhead (acceptable for testing)
- 100% of memory errors caught (worth the cost)

**Shadow Memory Principle**: For every 8 bytes of application memory, maintain 1 byte of "shadow" that describes its validity state.

## Theoretical Foundations

### Shadow Memory Mapping

The key innovation: a **compact shadow memory** encoding.

```
Application Memory:  [8 bytes] → Shadow: [1 byte]

Shadow encoding:
  0       = All 8 bytes addressable
  1-7     = First k bytes addressable, rest not
  Negative = Special (freed, redzone, etc.)
    -1 (0xFF) = Heap left redzone
    -2 (0xFE) = Heap right redzone  
    -3 (0xFD) = Freed heap region
    -5 (0xFB) = Stack left redzone
    -6 (0xFA) = Stack mid redzone
    -7 (0xF9) = Stack right redzone
    -8 (0xF8) = Stack after return
```

### Address Translation
```c
// Convert app address to shadow address
shadow_addr = (app_addr >> 3) + shadow_offset

// shadow_offset chosen so shadow region is valid memory
// On 64-bit Linux: shadow_offset = 0x7fff8000
```

### Memory Access Instrumentation
For every memory access, the compiler inserts a check:

```c
// Original code:
*ptr = value;

// Instrumented code:
shadow = (ptr >> 3) + offset;
if (*shadow != 0) {
    if (*shadow < 0 || (ptr & 7) + size > *shadow) {
        __asan_report_error(ptr, size, is_write);
    }
}
*ptr = value;
```

### Quarantine
To detect use-after-free, freed memory isn't immediately reused:

```
Allocation:  malloc(16) → [user memory] surrounded by [redzones]
Free:        free(ptr)  → memory goes to quarantine (still mapped, marked inaccessible)
Reuse:       After quarantine limit, memory can be reused

Quarantine size: ~256MB by default
Larger quarantine = better UAF detection, more memory usage
```

## Academic Papers

| Paper | Authors | Year | Key Contribution |
|-------|---------|------|------------------|
| [AddressSanitizer: A Fast Address Sanity Checker](https://www.usenix.org/conference/atc12/technical-sessions/presentation/serebryany) | Serebryany et al. | 2012 | Original ASan paper |
| [MemorySanitizer: Fast Detector of Uninitialized Memory](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43308.pdf) | Stepanov & Serebryany | 2015 | MSan design |
| [ThreadSanitizer: Data Race Detection in Practice](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35604.pdf) | Serebryany & Iskhodzhanov | 2009 | TSan design |
| [Kernel AddressSanitizer](https://www.kernel.org/doc/html/latest/dev-tools/kasan.html) | Google | 2015 | Kernel adaptation |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ADDRESSSANITIZER ARCHITECTURE                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    COMPILE TIME (Clang)                          │   │
│  │                                                                  │   │
│  │  Source → [Parser] → [AST] → [Instrumentation Pass] → [CodeGen] │   │
│  │                                      │                           │   │
│  │                        Inserts memory access checks              │   │
│  │                        Inserts stack frame poisoning             │   │
│  │                        Adds redzones to globals                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    RUNTIME (libasan)                             │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │   malloc    │  │    free     │  │    Error Reporting      │  │   │
│  │  │  Allocator  │  │  + Quarant  │  │  (symbolized stacks)    │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  │                                                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐│   │
│  │  │                    Shadow Memory                             ││   │
│  │  │                                                              ││   │
│  │  │  Application: [████████████████████████████████████]        ││   │
│  │  │                         ↓ (addr >> 3) + offset               ││   │
│  │  │  Shadow:      [░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]            ││   │
│  │  │               (1/8 the size)                                 ││   │
│  │  └─────────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Bug Types Detected

| Bug Type | How Detected |
|----------|--------------|
| **Heap buffer overflow** | Access to heap redzone |
| **Stack buffer overflow** | Access to stack redzone |
| **Global buffer overflow** | Access to global redzone |
| **Use-after-free** | Access to quarantined memory |
| **Use-after-return** | Access to returned stack frame (optional) |
| **Use-after-scope** | Access to out-of-scope stack var |
| **Double-free** | Free of quarantined memory |
| **Invalid free** | Free of non-heap memory |
| **Memory leaks** | LeakSanitizer integration |

## Usage

### Compile with ASan
```bash
# Basic
clang -fsanitize=address -g program.c -o program

# Recommended flags
clang -fsanitize=address \
      -fno-omit-frame-pointer \
      -g \
      -O1 \
      program.c -o program

# With debug info for better reports
clang -fsanitize=address -g -O1 -fno-omit-frame-pointer \
      -fsanitize-address-use-after-scope \
      program.c -o program
```

### Run with Options
```bash
# Basic run
./program

# With environment options
ASAN_OPTIONS="detect_leaks=1:check_initialization_order=1" ./program

# Useful options
ASAN_OPTIONS="
  verbosity=1
  detect_leaks=1
  check_initialization_order=1
  detect_stack_use_after_return=1
  strict_string_checks=1
  detect_invalid_pointer_pairs=2
"
```

### CMake Integration
```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)

if(ENABLE_ASAN)
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer -g)
    add_link_options(-fsanitize=address)
endif()
```

## Understanding ASan Output

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000014 at pc 0x... bp 0x... sp 0x...
WRITE of size 4 at 0x602000000014 thread T0
    #0 0x4e3f2a in main /path/to/file.c:10:5
    #1 0x7f... in __libc_start_main
    #2 0x41d289 in _start

0x602000000014 is located 0 bytes to the right of 4-byte region [0x602000000010,0x602000000014)
allocated by thread T0 here:
    #0 0x4d3e62 in malloc
    #1 0x4e3f12 in main /path/to/file.c:9:11
```

**Reading the output**:
1. **Error type**: `heap-buffer-overflow`
2. **Operation**: `WRITE of size 4`
3. **Stack trace**: Where the bad access happened
4. **Memory info**: Where the memory was allocated
5. **Offset**: `0 bytes to the right` = wrote just past the end

## The Mathematics of Shadow Memory

### Memory Layout (x86_64 Linux)
```
Virtual Address Space:
  [0x000000000000, 0x00007fffffffffff]  User space (128 TB)

ASan Layout:
  [0x00007fff8000, 0x10007fff7fff]      High shadow
  [0x00008fff7000, 0x00007fff7fff]      High memory
  [0x00007fff8000, 0x0000dfff7fff]      Low shadow  
  [0x000000000000, 0x00007fff7fff]      Low memory

Shadow mapping: shadow = (addr >> 3) + 0x7fff8000
```

### Why 1:8 Ratio?
- Minimum malloc alignment is 8 bytes
- Each shadow byte can describe 8 application bytes
- Compact: 1/8 memory overhead for shadow alone
- Fast: Single shift + add for translation

### Overhead Analysis
| Component | Overhead |
|-----------|----------|
| Shadow memory | ~12.5% of app memory |
| Redzones | 16-256 bytes per allocation |
| Quarantine | ~256MB total |
| Instrumentation | ~2x slowdown |
| **Total** | **~2x time, ~2-3x memory** |

## When to Use

- Development and testing of C/C++ code
- Continuous integration testing
- Fuzzing (pairs perfectly with libFuzzer)
- Debugging memory corruption issues
- Security auditing

## Limitations

- **Performance**: 2x slowdown (too slow for production)
- **Memory**: 2-3x memory usage
- **Compatibility**: Doesn't work with all allocators
- **False negatives**: Can miss some overflows (e.g., within struct)
- **No uninitialized reads**: Use MSan for that

## Related Tools

- **MemorySanitizer (MSan)** — Uninitialized memory detection
- **ThreadSanitizer (TSan)** — Data race detection
- **UndefinedBehaviorSanitizer (UBSan)** — Undefined behavior detection
- **LeakSanitizer (LSan)** — Memory leak detection (included in ASan)
- **HWAddressSanitizer** — Hardware-assisted, lower overhead

## References

- [ASan Wiki](https://github.com/google/sanitizers/wiki/AddressSanitizer)
- [Clang ASan Documentation](https://clang.llvm.org/docs/AddressSanitizer.html)
- [ASan Algorithm](https://github.com/google/sanitizers/wiki/AddressSanitizerAlgorithm)
- [USENIX ATC 2012 Paper](https://www.usenix.org/conference/atc12/technical-sessions/presentation/serebryany)
