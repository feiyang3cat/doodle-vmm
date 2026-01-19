# DOODLE-VMM

Learning and Hello World Project for Hypervisor/VMM/Sandbox Development.

---

## Chapter 1: Knowledge

### Key Terms

**Virtualization Concepts:**
- Hypervisor, VMM, VM, Guest OS, Bare Metal

**Kernel Architectures:**
- Microkernel, Unikernel, Monolithic Kernel, Hybrid Kernel

**System Interfaces:**
- Hypercalls, System calls, Traps/Interrupts/Exceptions, Privilege Levels (mode/ring/exception level)

---

### Traditional vs Virtualization Stack

```
Traditional Stack:                   Virtualization Stack:
┌─────────────┐                      ┌─────────────┐
│ Application │                      │ Application │
└─────┬───────┘                      └─────┬───────┘
      │ (syscall)                          │ (syscall)
┌─────▼───────┐                      ┌─────▼─────────────┐
│ OS/Kernel   │                      │ Guest OS/Kernel   │
└─────┬───────┘                      └─────┬─────────────┘
      │ (direct HW access)                 │ (hypercall)
┌─────▼───────┐                      ┌─────▼─────────────┐
│ Hardware    │                      │ Hypervisor + VMM  │
│ (CPU/Mem)   │                      └─────┬─────────────┘
└─────────────┘                            │ (HW instructions)
                                     ┌─────▼─────────────┐
                                     │ CPU / Memory /    │
                                     │ Devices (Hardware)│
                                     └───────────────────┘
```

---

### Core Problems to Solve

Managing multiple OSes on the same machine:
- Thread scheduling
- Memory management
- I/O device management
- Security isolation
- Performance optimization

---

### Players in the Field

| Project | Description |
|---------|-------------|
| **KVM** | Virtualization infrastructure for Linux kernel |
| **QEMU** | Emulator and virtualizer on top of Linux kernel |
| **Firecracker** | Lightweight VM manager with I/O virtualization, built on KVM |
| **Xen** | Type-1 hypervisor |
| **Wine** | Runtime compatibility layer for Windows apps on Linux |
| **macOS Hypervisor.framework** | Virtualization infrastructure for macOS kernel |

---

### Hypervisor Paradigm Comparison

| Category         | Linux KVM                      | macOS Hypervisor              |
|------------------|--------------------------------|-------------------------------|
| Architecture     | OS as Hypervisor, Monolithic   | User Space API                |
| Host OS          | Linux only                     | macOS only                    |
| Code Size        | Large; part of Linux kernel    | Large; part of macOS kernel   |
| Hardware Drivers | Unified; uses Linux drivers    | Handled by macOS kernel       |
| HW Support       | VT-x / AMD-V                   | VT-x (Intel) / ARM (Apple)    |
| Isolation        | Process-level (strong)         | Sandbox-level (very strong)   |

<img src="./assets/kvm-vs-hypervisor.jpg" alt="KVM vs Hypervisor.framework Comparison" width="600"/>

---

## Chapter 2: Projects

### 2.1 Learning the Firecracker Solution

**Reading:**
- [KVM Paper](https://www.kernel.org/doc/ols/2007/ols2007v1-pages-225-230.pdf)
- [Firecracker Paper](https://www.usenix.org/system/files/nsdi20-paper-agache.pdf) (NSDI '20)

### 2.2 macOS Hello World - TinyVMM

See the [tinyvmm/](./tinyvmm/) folder for a minimal VMM implementation using macOS Hypervisor.framework.
