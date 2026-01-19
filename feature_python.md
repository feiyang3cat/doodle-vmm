# Feature Proposal: Python Guest Application Support

## Overview

This document outlines the approaches to enable Python code execution within the TinyVMM guest environment.

---

## Current Architecture

### Guest Application Design

The guest application is **bare-metal ARM64 machine code** embedded directly in `main.c` (lines 86-428). It executes at ARM64 Exception Level 1 (EL1) with the following characteristics:

| Aspect | Current State |
|--------|---------------|
| Runtime Environment | No OS, no libc, no runtime |
| Code Format | Raw ARM64 machine code (uint32_t arrays) |
| Memory | 1MB flat memory space (0x0 - 0x100000) |
| Entry Point | GPA 0x10000 |
| Host Communication | HVC hypercalls only |

### Hypercall Interface

The guest communicates with the VMM through three hypercalls:

```
Hypercall #0 (EXIT)    - Terminate guest execution
Hypercall #1 (PUTCHAR) - Print single character (X1 = ASCII code)
Hypercall #2 (PUTS)    - Print string (X1 = guest physical address)
```

### Current Guest Programs

| VM | vCPU | Purpose |
|----|------|---------|
| VM 1 | 0 | Hello World + counter loop |
| VM 2 | 0 | Compute sum of even numbers (0+2+4+6+8 = 20) |
| VM 2 | 1 | Compute sum of odd numbers (1+3+5+7+9 = 25) |

---

## Challenge: Running Python in Bare-Metal Environment

Python is an **interpreted language** requiring:
- A Python interpreter (CPython, PyPy, MicroPython, etc.)
- Standard library dependencies
- Dynamic memory allocation (heap)
- System calls for I/O

The current VM provides **none of these**. The guest runs raw CPU instructions with no OS abstraction layer.

---

## Proposed Solutions

### Option 1: Embed MicroPython Runtime

**Approach**: Port MicroPython to run bare-metal on ARM64, compile it as the guest payload.

#### Implementation Steps

1. **Build MicroPython for bare-metal ARM64**
   - Configure MicroPython with `MICROPY_MINIMAL` settings
   - Disable filesystem, threading, and OS-dependent features
   - Target AArch64 with no OS (`-nostdlib`, `-ffreestanding`)

2. **Implement minimal runtime support**
   - Custom `_start` entry point (set up stack, zero BSS)
   - Heap allocator (simple bump allocator or dlmalloc port)
   - Basic libc stubs (`memcpy`, `memset`, `strlen`, etc.)

3. **Add hypercall bindings**
   - Create Python module exposing `hv_exit()`, `hv_putchar()`, `hv_puts()`
   - Register as built-in module in MicroPython config

4. **Embed Python script**
   - Freeze Python code into MicroPython binary
   - Or: Load script from guest memory region

#### VMM Changes Required

- Increase guest memory allocation (MicroPython needs ~256KB-512KB minimum)
- Load larger binary payload

#### Pros & Cons

| Pros | Cons |
|------|------|
| Real Python syntax and semantics | Binary size ~200KB-500KB |
| Supports most Python features | Requires MicroPython porting effort |
| Active community and documentation | Limited stdlib (no `os`, `socket`, etc.) |

---

### Option 2: Boot Full Linux Kernel

**Approach**: Replace bare-metal guest with a minimal Linux kernel + initramfs containing Python.

#### Implementation Steps

1. **Build minimal Linux kernel**
   - Configure for ARM64 (`ARCH=arm64`)
   - Enable virtio-console, virtio-blk drivers
   - Disable unnecessary subsystems (networking, graphics, etc.)

2. **Create initramfs with Python**
   - Use Alpine Linux or BusyBox base
   - Install Python 3 minimal package
   - Add init script to run Python program

3. **Extend VMM capabilities**
   - Implement virtio device emulation (PCI, console, block)
   - Add interrupt controller emulation (GICv3)
   - Handle PSCI calls for CPU power management
   - Implement MMIO trap handling

4. **Boot protocol**
   - Load kernel Image at 0x80000
   - Load DTB (device tree) at separate address
   - Set up initial register state per Linux boot protocol

#### VMM Changes Required

| Component | Changes |
|-----------|---------|
| Memory | Increase to 64MB-256MB |
| Interrupts | Implement GICv3 distributor/redistributor |
| Devices | Add virtio-console, virtio-blk emulation |
| Boot | DTB generation, kernel loading |

#### Pros & Cons

| Pros | Cons |
|------|------|
| Full CPython with complete stdlib | Major VMM rewrite required |
| Standard Linux environment | Large memory footprint (64MB+) |
| Access to filesystem, networking | Complex boot process |
| Easier debugging | Slower startup time |

---

### Option 3: Python-to-ARM64 Transpiler

**Approach**: Compile Python source code to native ARM64 machine code ahead-of-time.

#### Implementation Steps

1. **Select or build transpiler toolchain**
   - Options: Nuitka, Cython, mypyc, or custom
   - Configure for bare-metal output (no libc dependency)

2. **Implement runtime support library**
   - Memory allocator
   - Basic type implementations (int, str, list, dict)
   - Exception handling mechanism

3. **Handle Python semantics**
   - Dynamic typing → runtime type checks
   - Garbage collection → reference counting or arena allocator
   - Built-in functions → inline implementations

4. **Generate hypercall wrappers**
   - `print()` → `HYPERCALL_PUTS`
   - `exit()` → `HYPERCALL_EXIT`

#### VMM Changes Required

- Minimal (same as current architecture)
- May need larger code region depending on output size

#### Pros & Cons

| Pros | Cons |
|------|------|
| Native performance | Loses dynamic features (`eval`, `exec`, `import`) |
| No interpreter overhead | Complex to implement correctly |
| Small runtime footprint | Limited Python compatibility |
| Minimal VMM changes | Long compilation times |

---

### Option 4: Host-side Python with Code Generation

**Approach**: Keep guest as bare-metal machine code, but use Python on the host side to generate or control guest behavior.

#### Implementation Steps

1. **Create Python-based assembler**
   ```python
   from guest_asm import GuestProgram

   prog = GuestProgram()
   prog.mov(x0, 1)           # hypercall number
   prog.mov(x1, ord('H'))    # character
   prog.hvc(0)               # invoke hypercall
   prog.compile_to("guest.bin")
   ```

2. **Build code generation library**
   - ARM64 instruction encoder in Python
   - High-level abstractions (loops, conditionals, function calls)
   - Automatic register allocation

3. **Integrate with VMM**
   - Python script generates `guest.bin`
   - VMM loads binary at runtime
   - Or: Embed generated code in C source

#### VMM Changes Required

- Add command-line option to load external binary
- Modify `load_guest()` to read from file

#### Pros & Cons

| Pros | Cons |
|------|------|
| Minimal VMM changes | Guest still runs machine code |
| Python for development workflow | Not "Python in the guest" |
| Easy to extend and modify | Two-stage build process |
| Full Python on host side | Learning curve for code gen API |

---

### Option 5: Custom Bytecode Interpreter

**Approach**: Write a minimal Python bytecode interpreter in ARM64 assembly that runs in the guest.

#### Implementation Steps

1. **Design bytecode subset**
   - Support: LOAD_CONST, STORE_NAME, BINARY_ADD, PRINT, RETURN
   - Skip: classes, exceptions, generators, imports

2. **Implement interpreter loop**
   ```
   fetch_loop:
       ldrb w0, [x10], #1    ; fetch opcode
       ldr  x1, [x11, x0, lsl #3]  ; lookup handler
       br   x1               ; dispatch
   ```

3. **Build type system**
   - Tagged values (int, str, none)
   - Simple heap with bump allocator

4. **Create compiler (on host)**
   - Python script compiles `.py` to custom bytecode
   - Embed bytecode in guest memory

#### VMM Changes Required

- Minimal (same as current architecture)
- May need slightly more guest memory for heap

#### Pros & Cons

| Pros | Cons |
|------|------|
| Educational value | Very limited Python subset |
| Runs real Python syntax | Significant implementation effort |
| Small footprint | Poor performance |
| Full control over semantics | Must maintain custom toolchain |

---

## Comparison Matrix

| Criteria | Option 1 | Option 2 | Option 3 | Option 4 | Option 5 |
|----------|----------|----------|----------|----------|----------|
| Python Compatibility | High | Full | Low | N/A | Very Low |
| VMM Changes | Low | High | Low | Low | Low |
| Guest Memory | 512KB | 64MB+ | 64KB | 4KB | 32KB |
| Implementation Effort | Medium | High | High | Low | Medium |
| Performance | Good | Good | Excellent | Excellent | Poor |
| Maintenance | Low | Medium | High | Low | High |

---

## Recommendation

For a balance of **real Python execution** with **minimal VMM changes**:

**Option 1 (MicroPython)** is recommended as the primary path forward.

For a **quick proof-of-concept** or development workflow improvement:

**Option 4 (Host-side Python)** can be implemented first as it requires the least effort.

---

## Next Steps

1. [ ] Choose implementation approach
2. [ ] Create detailed technical design
3. [ ] Implement prototype
4. [ ] Test and validate
5. [ ] Document usage and examples
