# Release Notes

## v1.0.0 - Initial Release

**Release Date**: January 2026

### Overview

TinyVMM is a minimal Virtual Machine Monitor (VMM) for macOS demonstrating how to create a hypervisor using Apple's Hypervisor.framework on Apple Silicon.

### Features

#### Core VMM Implementation
- VM creation and initialization using Hypervisor.framework
- Guest memory allocation (1MB) with proper GPA mapping
- vCPU management with full register configuration
- ARM64 guest code execution at EL1 (kernel mode)

#### VM Exit Handling
- Exception-based exit handling via ESR_EL2
- Support for HVC64, SYS64, DABORT, and IABORT exceptions
- Virtual timer support

#### Hypercall Interface
| Number | Name    | Description                    |
|--------|---------|--------------------------------|
| 0      | EXIT    | Terminate the VM               |
| 1      | PUTCHAR | Print a single character       |
| 2      | PUTS    | Print a string from guest memory |

#### Guest Code
- Embedded ARM64 machine code (240 bytes)
- Demonstrates string output, loop control, and hypercall usage
- Outputs: "Hello from VM!" and counts 0-4

### Files Added
- `main.c` - Core VMM implementation (~550 lines)
- `guest.S` - ARM64 assembly for guest experimentation
- `Makefile` - Build configuration
- `entitlements.plist` - macOS code signing entitlements

### Requirements
- macOS 11.0 (Big Sur) or later
- Apple Silicon (M1/M2/M3)
- Xcode Command Line Tools

### Building & Running
```bash
make
codesign --entitlements entitlements.plist --force -s - tinyvmm
./tinyvmm
```

### Known Limitations
- Single vCPU only
- No device emulation (virtio, etc.)
- Bare metal guest only (no OS support)
- 1MB fixed memory size

### Contributors
- Initial implementation with Claude Code

---

## Commit History

| Commit  | Description                    |
|---------|--------------------------------|
| 1b735e3 | Core VMM implementation        |
| a91103b | Project structure setup        |
| 093468b | Initial commit                 |
