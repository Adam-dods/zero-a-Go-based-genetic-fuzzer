# go genetic-fuzzer

> Autonomous vulnerability discovery engine using evolutionary algorithms Go + Rust + C research framework.

![Go](https://img.shields.io/badge/Go-1.22-00ADD8?style=flat&logo=go)
![Rust](https://img.shields.io/badge/Rust-stable-orange?style=flat&logo=rust)
![C](https://img.shields.io/badge/C-research-blue?style=flat&logo=c)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-research-yellow)

---

## Research Overview

This project is an academic research implementation of a **Genetic Fuzzing Engine** a system that autonomously discovers software vulnerabilities using evolutionary algorithms.

Unlike traditional fuzzers that send random data blindly, this engine **learns and evolves**. Each generation of payloads is smarter than the last, guided by execution time feedback and memory crash signals.

> All testing is performed exclusively against intentionally vulnerable programs built specifically for this research. No real-world systems are targeted.

---

## How It Works

### The Core Idea Natural Selection for Bugs

The engine treats input payloads like DNA. It applies the principles of natural selection to evolve payloads toward vulnerability triggers:

```
Generation 0:  [random payloads] → evaluate → rank by fitness
Generation 1:  [top performers breed] → mutate → evaluate → rank
Generation N:  [evolved payloads] → CRASH DETECTED → Zero-Day Found
```

---

## System Architecture

```
go-genetic-fuzzer/
├── targets/
│   └── dummy_target.c        # Intentionally vulnerable C program (the prey)
├── pkg/
│   ├── engine/
│   │   └── fuzzer.go         # Genetic algorithm core
│   └── evaluator/
│       └── runner.go         # Execution monitor & telemetry
├── cmd/
│   └── zero/
│       └── main.go           # Orchestrator entry point
└── memory_oracle/            # Rust-based RAM monitor (Phase 2)
    └── src/
        └── main.rs
```

---

## Phase 1 — The Genetic Engine (Go)

The brain of the system. Generates, evaluates, and evolves payloads using a genetic algorithm:

```go
// Payload represents a single genetic entity
type Payload struct {
    Data    []byte
    Fitness int64 // Measured in nanoseconds of execution time
    Crashed bool  // True if the payload caused a Segmentation Fault
}

const (
    PopulationSize = 50
    Generations    = 500
    PayloadLength  = 40
    MutationRate   = 0.1 // 10% chance to mutate each byte
)

// Evaluate executes the target and measures its response
func Evaluate(payload *Payload) {
    start := time.Now()
    cmd := exec.CommandContext(ctx, TargetBinary, string(payload.Data))
    err := cmd.Run()
    payload.Fitness = time.Since(start).Nanoseconds()

    // Detect Segmentation Fault (SIGSEGV = signal 11)
    if status.Signaled() && status.Signal() == syscall.SIGSEGV {
        payload.Crashed = true
        payload.Fitness = 999999999 // Maximum fitness for a crash
    }
}
```

---

## Phase 2 — The Memory Oracle (Rust)

A Rust node embedded alongside the target that monitors RAM in real-time. When Go's payload causes memory corruption, Rust signals the crash instantly via shared memory:

```
Go Engine ──── payload ────► Target Process
    ▲                              │
    │                         [SIGSEGV]
    │                              │
Rust Oracle ◄──── crash signal ───┘
    │
    └──► "Crash detected at Generation N — payload logged"
```

**Why Rust?** Zero-cost abstractions and direct memory access make it ideal for monitoring without adding overhead that would skew timing measurements.

---

## Phase 3 — Shared Memory IPC

Go and Rust communicate through **memory-mapped files** (`mmap`) zero latency, no network overhead, no disk I/O:

```
┌─────────────┐    mmap    ┌─────────────────┐
│  Go Engine  │ ◄────────► │  Rust Oracle    │
│  (Fuzzer)   │  0 latency │  (RAM Monitor)  │
└─────────────┘            └─────────────────┘
```

---

## Phase 4 — The Evolution Loop

The core genetic algorithm survival of the fittest payloads:

```go
// Crossover: combine DNA from two parent payloads
func Crossover(parent1, parent2 []byte) []byte {
    child := make([]byte, len(parent1))
    midpoint := len(parent1) / 2
    copy(child[:midpoint], parent1[:midpoint])
    copy(child[midpoint:], parent2[midpoint:])
    return child
}

// Mutate: introduce random genetic alterations
func Mutate(data []byte) []byte {
    for i := range mutated {
        if rand.Float64() < MutationRate {
            mutated[i] = byte(rand.Intn(94) + 33)
        }
    }
    return mutated
}

// Evolution: keep top 20%, breed the rest
eliteCount := PopulationSize / 5
// Top performers pass directly to next generation (Elitism)
// Remaining slots filled via Crossover + Mutation
```

---

## The Research Target dummy_target.c

An intentionally vulnerable C program built specifically for this research. It contains a **Stack-based Buffer Overflow (CWE-121)** with a guided execution path that rewards the fuzzer with timing delays as it approaches the vulnerability:

```c
void process_payload(const char *input) {
    char buffer[32]; // 32-byte stack buffer

    // Guided path — each correct byte triggers a delay
    // This delay acts as a "fitness reward" for the genetic algorithm
    if (input[0] == 'Z') {
        simulated_delay(); // Reward signal to fuzzer
        if (input[1] == 'R') {
            simulated_delay();
            if (input[2] == 'O') {
                simulated_delay();
                strcpy(buffer, input); // VULNERABILITY: no bounds check
                // If payload > 32 bytes → Stack Overflow → SIGSEGV
            }
        }
    }
}
```

The fuzzer doesn't know about `ZRO` it discovers the path purely through evolutionary pressure on execution timing.

---

## Live Research Results

```
[*] INITIALIZING PROJECT ZERO: GENETIC ENGINE
[*] Target: ./dummy_target | Population: 50 | Max Generations: 500

[~] Gen 001 | Max Fitness: 142301 ns | Best Genes: !Kp#mQ9...
[~] Gen 008 | Max Fitness: 891204 ns | Best Genes: Zm3#pQ9...
[~] Gen 015 | Max Fitness: 5021847 ns | Best Genes: ZR9#pQ9...
[~] Gen 023 | Max Fitness: 15089234 ns | Best Genes: ZRO#pQ9...

[🚨] ZERO-DAY DISCOVERED AT GENERATION 23!
[+] Payload (Hex): 5a524f41414141414141414141414141...
[+] Payload (Str): ZROAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

The engine evolved from random noise to the exact crash payload in **23 generations** without any prior knowledge of the target's internal logic.

---

## Comparison with Existing Fuzzers

| Tool | Language | Strategy | Memory Monitoring |
|------|----------|----------|-------------------|
| AFL++ | C | Coverage-guided | Via instrumentation |
| libFuzzer | C++ | Coverage-guided | Via sanitizers |
| Boofuzz | Python | Protocol-aware | External |
| **This Engine** | **Go + Rust** | **Genetic + Time-feedback** | **Rust Memory Oracle** |

The key differentiator is the **Rust Memory Oracle** a dedicated real-time RAM monitor that operates independently of the target process, providing crash signals with zero overhead on the fuzzing loop.

---

## Research Applications

- Buffer overflow discovery in C/C++ programs
- Format string vulnerability detection
- Integer overflow edge case identification
- Protocol parser vulnerability research
- Academic study of evolutionary algorithms in security

---

## Build & Run

```bash
# Compile the vulnerable target
gcc -o dummy_target targets/dummy_target.c -fno-stack-protector

# Run the genetic fuzzer
go run cmd/zero/main.go

# Build the Rust memory oracle
cd memory_oracle && cargo build --release
```

---

## Academic References

- Evolutionary Computation in Vulnerability Discovery
- AFL: American Fuzzy Lop Michał Zalewski
- Genetic Algorithms in Software Testing Fraser & Arcuri
- CWE-121: Stack-based Buffer Overflow

---

## About

Built by **Adam El Outtassi** Systems Engineer and Security Researcher from Morocco.

Research conducted in isolated lab environments on intentionally vulnerable targets only.

| Platform | Link |
|----------|------|
| Twitter | [@AdamSecDev](https://x.com/AdamSecDev) |

---

> *Evolution finds what brute force cannot.*
EOF
