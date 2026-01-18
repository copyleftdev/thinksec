# Tool: syzkaller

> Unsupervised kernel fuzzer that finds bugs in operating system kernels through syscall fuzzing.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [google/syzkaller](https://github.com/google/syzkaller) |
| **Language** | Go |
| **Category** | Kernel Fuzzing |
| **Maintainer** | Google |

## Philosophy

**Core Insight**: Operating system kernels are among the most security-critical and hardest-to-test software. They have massive attack surfaces (syscalls) and complex internal state. Fuzzing kernels requires:
1. Knowledge of syscall interfaces (types, constraints)
2. Ability to survive crashes (VMs)
3. Coverage feedback from kernel code

**The Approach**: Describe syscalls formally, generate valid sequences, execute in VMs, use kernel coverage to guide evolution.

**Key Innovation**: **Syzlang** — a domain-specific language for describing syscall interfaces with types, constraints, and relationships.

## Theoretical Foundations

### Grammar-Based Fuzzing
Unlike byte-level mutation, syzkaller generates **structured syscall sequences**:

```
Byte fuzzing:  random bytes → mostly invalid → waste time
Grammar fuzzing: valid syscalls → valid arguments → explore kernel
```

### Syscall Description (Syzlang)
```syzlang
# File operations
open(file filename, flags flags[open_flags], mode flags[open_mode]) fd
read(fd fd, buf buffer[out], count len[buf])
write(fd fd, buf buffer[in], count len[buf])
close(fd fd)

# Flags
open_flags = O_RDONLY, O_WRONLY, O_RDWR, O_CREAT, O_TRUNC, O_APPEND

# Resources (tracked across calls)
resource fd[int32]: 0xffffffffffffffff, AT_FDCWD
```

### Coverage-Guided Evolution
```
1. Generate syscall program P from grammar
2. Execute P in VM
3. Collect kernel coverage (KCOV)
4. If new coverage:
   - Add P to corpus
   - Mutate P to create variants
5. If crash:
   - Minimize reproducer
   - Report bug
```

### Stateful Fuzzing
Syzkaller understands **resource flow** between syscalls:

```
fd = open("/tmp/file", O_RDWR)  // Creates resource 'fd'
write(fd, buffer, 100)          // Uses resource 'fd'
mmap(addr, 4096, PROT_READ, MAP_SHARED, fd, 0)  // Uses 'fd'
close(fd)                        // Consumes resource 'fd'
```

This enables finding bugs that require specific sequences.

## Academic Papers

| Paper | Authors | Year | Key Contribution |
|-------|---------|------|------------------|
| [syzkaller: Linux Kernel Fuzzing](https://github.com/google/syzkaller/blob/master/docs/internals.md) | Vyukov | 2015+ | Core design |
| [kAFL: Hardware-Assisted Feedback Fuzzing for OS Kernels](https://www.usenix.org/conference/usenixsecurity17/technical-sessions/presentation/schumilo) | Schumilo et al. | 2017 | Hardware-assisted kernel fuzzing |
| [DIFUZE: Interface Aware Fuzzing for Kernel Drivers](https://acmccs.github.io/papers/p2123-corinaA.pdf) | Corina et al. | 2017 | Driver interface extraction |
| [Moonshine: Optimizing OS Fuzzer Seed Selection](https://www.usenix.org/conference/usenixsecurity18/presentation/pailoor) | Pailoor et al. | 2018 | Seed selection from traces |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         SYZKALLER ARCHITECTURE                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         syz-manager                              │   │
│  │                     (Orchestration, UI)                          │   │
│  │                                                                  │   │
│  │  - Manages VM pool                                               │   │
│  │  - Stores corpus                                                 │   │
│  │  - Deduplicates crashes                                         │   │
│  │  - Provides web dashboard                                       │   │
│  └─────────────┬───────────────────────────────────────────────────┘   │
│                │                                                        │
│       ┌────────┴────────┬────────────────┬────────────────┐            │
│       ▼                 ▼                ▼                ▼            │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐       │
│  │   VM    │      │   VM    │      │   VM    │      │   VM    │       │
│  │         │      │         │      │         │      │         │       │
│  │┌───────┐│      │┌───────┐│      │┌───────┐│      │┌───────┐│       │
│  ││syz-   ││      ││syz-   ││      ││syz-   ││      ││syz-   ││       │
│  ││fuzzer ││      ││fuzzer ││      ││fuzzer ││      ││fuzzer ││       │
│  │└───┬───┘│      │└───┬───┘│      │└───┬───┘│      │└───┬───┘│       │
│  │    │    │      │    │    │      │    │    │      │    │    │       │
│  │┌───┴───┐│      │┌───┴───┐│      │┌───┴───┐│      │┌───┴───┐│       │
│  ││syz-   ││      ││syz-   ││      ││syz-   ││      ││syz-   ││       │
│  ││executor│      ││executor│      ││executor│      ││executor│       │
│  │└───┬───┘│      │└───┬───┘│      │└───┬───┘│      │└───┬───┘│       │
│  │    │    │      │    │    │      │    │    │      │    │    │       │
│  │┌───┴───┐│      │┌───┴───┐│      │┌───┴───┐│      │┌───┴───┐│       │
│  ││ Kernel ││     ││ Kernel ││     ││ Kernel ││     ││ Kernel ││      │
│  ││ (KCOV) ││     ││ (KCOV) ││     ││ (KCOV) ││     ││ (KCOV) ││      │
│  │└───────┘│      │└───────┘│      │└───────┘│      │└───────┘│       │
│  └─────────┘      └─────────┘      └─────────┘      └─────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Components
- **syz-manager**: Main orchestrator, runs on host
- **syz-fuzzer**: Generates programs, runs in VM
- **syz-executor**: Executes syscalls, collects coverage
- **KCOV**: Kernel coverage infrastructure

## Syzlang Examples

### Basic Syscall
```syzlang
# Simple file read
read(fd fd, buf buffer[out], count len[buf]) len[buf]
```

### Complex Structure
```syzlang
# ioctl with structure
ioctl$DRM_IOCTL_MODE_CURSOR(fd fd_drm, cmd const[DRM_IOCTL_MODE_CURSOR], 
    arg ptr[in, drm_mode_cursor])

drm_mode_cursor {
    flags   flags[drm_mode_cursor_flags, int32]
    crtc_id int32
    x       int32
    y       int32
    width   int32
    height  int32
    handle  int32
}
```

### Resource Dependencies
```syzlang
# Socket creation and use
socket(domain flags[socket_domain], type flags[socket_type], 
       proto int32) sock

bind(fd sock, addr ptr[in, sockaddr_storage], addrlen len[addr])
listen(fd sock, backlog int32)
accept(fd sock, peer ptr[out, sockaddr_storage], 
       peerlen ptr[inout, len[peer, int32]]) sock
```

## Bug Types Found

| Bug Type | Count (Historical) | Example |
|----------|-------------------|---------|
| Use-after-free | Hundreds | CVE-2016-0728 |
| Buffer overflow | Hundreds | Various subsystems |
| Null pointer deref | Thousands | Throughout kernel |
| Race conditions | Hundreds | Found with KTSAN |
| Memory leaks | Many | Found with KMEMLEAK |
| Deadlocks | Many | Lock ordering bugs |

## Usage

### Build syzkaller
```bash
git clone https://github.com/google/syzkaller
cd syzkaller
make
```

### Build Kernel with Coverage
```bash
# Required kernel configs
CONFIG_KCOV=y
CONFIG_KCOV_INSTRUMENT_ALL=y
CONFIG_KCOV_ENABLE_COMPARISONS=y
CONFIG_DEBUG_INFO=y
CONFIG_KASAN=y  # Optional but recommended
```

### Configuration (my.cfg)
```json
{
    "target": "linux/amd64",
    "http": "127.0.0.1:56741",
    "workdir": "/syzkaller/workdir",
    "kernel_obj": "/linux/vmlinux",
    "image": "/images/stretch.img",
    "sshkey": "/images/stretch.id_rsa",
    "syzkaller": "/syzkaller",
    "procs": 8,
    "type": "qemu",
    "vm": {
        "count": 4,
        "kernel": "/linux/arch/x86/boot/bzImage",
        "cpu": 2,
        "mem": 2048
    }
}
```

### Run Fuzzer
```bash
./bin/syz-manager -config=my.cfg
# Dashboard at http://127.0.0.1:56741
```

## Key Metrics

| Metric | Value (as of 2024) |
|--------|-------------------|
| Bugs found in Linux | 5000+ |
| Bugs found in other kernels | 1000+ |
| Supported OSes | Linux, FreeBSD, NetBSD, OpenBSD, Fuchsia, Windows |
| Syscalls described | 3000+ (Linux) |
| Active instances | 100+ (syzbot) |

## syzbot (Continuous Fuzzing)

Google runs **syzbot**, continuous syzkaller against upstream kernels:
- https://syzkaller.appspot.com/
- Reports bugs directly to mailing lists
- Tracks fix status
- Provides reproducers

## When to Use

- Kernel development/testing
- Driver security testing
- Finding syscall-related bugs
- Testing security subsystems
- Research on OS security

## Limitations

- **Complexity** — Significant setup required
- **Resources** — Needs VMs, CPU, memory
- **Descriptions** — New syscalls need manual syzlang
- **Coverage** — Some kernel code hard to reach

## Related Tools

- **kAFL** — Intel PT-based kernel fuzzing
- **Trinity** — Simpler syscall fuzzer
- **DIFUZE** — Automatic driver interface extraction
- **KCOV** — Kernel coverage infrastructure

## References

- [syzkaller Documentation](https://github.com/google/syzkaller/tree/master/docs)
- [syzbot Dashboard](https://syzkaller.appspot.com/)
- [Syzlang Specification](https://github.com/google/syzkaller/blob/master/docs/syscall_descriptions.md)
- [How to Write Syscall Descriptions](https://github.com/google/syzkaller/blob/master/docs/syscall_descriptions_syntax.md)
