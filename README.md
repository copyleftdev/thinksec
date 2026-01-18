# thinksec

> A curated collection of security skill files — capturing how experts think and work.

## What is thinksec?

thinksec is an open repository of **security skill files**: structured, actionable documents that capture the methodologies, mental models, and procedures used by security professionals across different specialties.

Each skill file answers: *"How would an expert approach this problem?"*

## Why Skill Files?

Traditional documentation tells you *what* to do. Skill files capture *how* to think:

- **Procedural** — Step-by-step workflows
- **Contextual** — When to use (and when not to)
- **Practical** — Real examples and anti-patterns
- **Measurable** — Metrics for success

## Repository Structure

```
thinksec/
├── README.md              # You are here
├── CONTRIBUTING.md        # How to contribute
├── templates/
│   ├── skill-template.md  # Template for new skills
│   └── tool-template.md   # Template for new tools
├── skills/                # How to think (procedural knowledge)
│   ├── offensive/         # Red team, pentesting, exploitation
│   ├── defensive/         # Blue team, detection, response
│   ├── analysis/          # Malware, forensics, reverse engineering
│   ├── intel/             # Threat intelligence, attribution
│   ├── engineering/       # Secure development, architecture
│   └── operations/        # IR, SOC, hunting
└── tools/                 # Deep dives (philosophy, math, research)
    ├── fuzzing/           # OSS-Fuzz, libFuzzer, syzkaller
    ├── sanitizers/        # ASan, MSan, TSan, UBSan
    ├── cryptography/      # Tink, Wycheproof
    └── network-security/  # Tsunami
```

## Skill File Format

Every skill file follows a consistent structure:

| Section | Purpose |
|---------|---------|
| **When to Use** | Trigger conditions — when this skill applies |
| **Prerequisites** | What you need before starting |
| **Steps** | Ordered procedure with checkboxes |
| **Examples** | Good execution patterns |
| **Anti-patterns** | What NOT to do and why |
| **Metrics** | How to measure success |
| **References** | Source material and further reading |

## Quick Start

### Using a Skill

1. Browse `skills/` by category
2. Find a relevant skill file
3. Review "When to Use" to confirm fit
4. Follow the steps, adapting to your context
5. Check anti-patterns to avoid common mistakes

### Contributing a Skill

1. Read [CONTRIBUTING.md](CONTRIBUTING.md)
2. Copy `templates/skill-template.md`
3. Fill in all sections
4. Submit a pull request

## Skill Categories

### Offensive Security
Skills for red team operations, penetration testing, vulnerability research, and exploitation.

### Defensive Security
Skills for blue team operations, detection engineering, hardening, and monitoring.

### Analysis
Skills for malware analysis, forensics, reverse engineering, and incident investigation.

### Intelligence
Skills for threat intelligence, attribution, campaign tracking, and reporting.

### Engineering
Skills for secure development, cryptography, architecture, and tooling.

### Operations
Skills for incident response, SOC operations, threat hunting, and security operations.

## Tools

Beyond skills, thinksec includes **tool files** — deep documentation capturing the philosophy, mathematics, and research behind security tools.

| Category | Tools |
|----------|-------|
| **Fuzzing** | OSS-Fuzz, libFuzzer, syzkaller |
| **Sanitizers** | AddressSanitizer |
| **Cryptography** | Tink, Wycheproof |
| **Network Security** | Tsunami |

Each tool file includes:
- **Philosophy** — The core insight that makes it work
- **Theoretical Foundations** — CS, math, security concepts
- **Academic Papers** — Research behind the tool
- **Architecture** — How it works internally

See [tools/README.md](tools/README.md) for details.

## Principles

1. **Practical over theoretical** — Every skill should be actionable
2. **Explicit over implicit** — Capture the "obvious" steps experts skip
3. **Humble over heroic** — Include failures, edge cases, limitations
4. **Evolving over static** — Skills improve with community input

## License

MIT License — use freely, contribute back.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

*"The best security professionals don't just know tools — they know how to think."*
