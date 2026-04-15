# Tensix TRISC Runtime Environment — Reference

This document describes the runtime environment for Tensix TRISC (T0/T1/T2)
processors on Tenstorrent Blackhole (p100a/p150) silicon.  It covers memory
layout, hardware mechanisms, initialization sequences, inter-kernel state,
and the compiler-specific differences between GCC (`riscv-tt-elf`) and
LLVM/clang that must be accounted for when targeting these processors.

Everything here was learned the hard way — through silicon hangs, binary
comparisons, and iterative debugging.  If you are writing a compiler backend,
firmware, or low-level kernel code for Tensix, this document should save you
weeks of pain.

---

## 1  Processor Architecture Overview

Each Tensix core has **five 32-bit RISC-V processors**:

| Processor | Name    | Role                | Firmware source          |
|-----------|---------|---------------------|--------------------------|
| BRISC     | B       | Dataflow reader     | `brisc.cc` / `brisck.cc` |
| NCRISC    | NC      | Dataflow writer/NOC | `ncrisc.cc` / `nrisck.cc`|
| TRISC0    | T0      | UNPACK (unpacker)   | `trisc.cc` / `trisck.cc` |
| TRISC1    | T1      | MATH (FPU/SFPU)     | `trisc.cc` / `trisck.cc` |
| TRISC2    | T2      | PACK (packer)       | `trisc.cc` / `trisck.cc` |

The three TRISCs share the Tensix **coprocessor**, a 3-way threaded execution
engine with shared backend units (Unpackers, Matrix FPU, Packers, SFPU,
Scalar/Config units).  Each TRISC feeds its corresponding coprocessor thread.

TRISC code is split into two compilation units:

- **Firmware** (`trisc.cc`) — compiled once per program, runs the boot
  sequence and dispatch loop.  The same source is compiled three times with
  `COMPILE_FOR_TRISC=0/1/2`.
- **Kernel** (`trisck.cc`) — compiled per-kernel at JIT time, linked against
  the firmware's "weakened" ELF for symbol addresses.

---

## 2  Memory Map

### 2.1  MMIO Address Space

| Region              | Address          | Size    | Purpose                           |
|---------------------|------------------|---------|-----------------------------------|
| REGFILE_BASE        | `0xFFE0_0000`    | 256 KB  | Coprocessor register file (64 x 32-bit GPRs) |
| INSTRN_BUF_BASE     | `0xFFE4_0000`    | 64 KB   | Instruction dispatch FIFO         |
| PC_BUF_BASE (T0)    | `0xFFE8_0000`    | 64 KB   | PC buffer + semaphores (TRISC0)   |
| PC1_BUF_BASE (T1)   | `0xFFE9_0000`    | 64 KB   | PC buffer (TRISC1)                |
| PC2_BUF_BASE (T2)   | `0xFFEA_0000`    | 64 KB   | PC buffer (TRISC2)                |
| TENSIX_MAILBOX0-3   | `0xFFEC_{0-3}000`| 4 KB ea | Inter-processor mailboxes         |
| TENSIX_CFG_BASE     | `0xFFEF_0000`    | 64 KB   | Coprocessor config registers      |
| TENSIX_MOP_CFG_BASE | `0xFFB8_0000`    | 256 B   | MOP (Micro-Operation) config      |

### 2.2  Local Data Memory (LDM)

Each TRISC has **4 KB** of local data memory (scratchpad):

| Component          | Address          | Notes |
|--------------------|------------------|-------|
| LDM start          | `0xFFB0_0000`    | Same for all 3 TRISCs (aliased) |
| LDM end            | `0xFFB0_1000`    | 4096 bytes total |
| Stack top          | `0xFFB0_1000`    | Grows downward |
| Min stack reserved | 192 bytes (T0/T1), 256 bytes (T2) | |

### 2.3  L1 Memory

| Component          | Address  | Size     | Notes |
|--------------------|----------|----------|-------|
| L1 base            | `0x0`    | 1.5 MB   | Shared across all processors |
| Mailbox region     | `0x60`   | ~12.6 KB | `MEM_MAILBOX_BASE` |
| Circular buffers   | Dynamic  | Dynamic  | Configured per-kernel by host |

---

## 3  LDM Layout — Firmware vs Kernel

Understanding LDM layout is **critical**.  The firmware and kernel share the
same 4 KB LDM space.  The firmware occupies the first portion; the kernel
gets whatever remains.

### 3.1  Firmware LDM Layout

```
0xFFB0_0000  ┌─────────────────────┐
             │ .data (VMA)         │  ← Initialized globals
             │   (GCC: empty)      │     GCC constant-folds MMIO ptrs
             │   (LLVM: empty)     │     LLVM puts them in .data.mmio_ptrs
             ├─────────────────────┤
             │ .bss                │  ← Zero-initialized globals
             │   dest_offset_id    │  ← MUST be at 0xFFB0_0000 (.bss.0_dest)
             │   op_info_offset    │  ← .bss.1_opinfo
             │   cfg_state_id      │
             │   cb_interface[64]  │  ← 2048 bytes (biggest BSS item)
             │   ...               │
             ├─────────────────────┤
             │ .data.mmio_ptrs     │  ← LLVM only: reg_base, pc_buf_base,
             │   (GCC: absent)     │     regfile, instrn_buffer (~12-16 bytes)
             ├─────────────────────┤
__fw_export_ldm_end                   ← Kernel data starts here
             │ [Kernel .data]      │  ← Kernel initialized data
             │ [Kernel .bss]       │  ← Kernel zero-initialized data
             │ [Kernel stack]      │  ← Grows downward from end
             │                     │
0xFFB0_1000  └─────────────────────┘  ← Stack top (4 KB boundary)
```

### 3.2  Why BSS Order Matters

`dest_offset_id` **must** be at `0xFFB0_0000`.  The pre-compiled firmware ELF
exports this address as a weak symbol.  If clang reorders BSS and puts another
variable first, the kernel reads `dest_offset_id` from the wrong address and
the matmul kernel deadlocks (reads wrong DEST buffer half).

**Fix**: Named sections force ordering:
```c
uint32_t dest_offset_id __attribute__((used, section(".bss.0_dest"))) = 0;
uint32_t op_info_offset __attribute__((used, section(".bss.1_opinfo"))) = 0;
```

The linker script sorts `.bss.0_*` before `.bss.1_*` before generic `.bss`.

### 3.3  Why MMIO Pointers Need `.data.mmio_ptrs`

GCC constant-folds MMIO pointer initializers like
`reinterpret_cast<volatile uint*>(0xFFB10000)` — they produce no `.data`
section at all.  Clang does not fold these; they go into `.data`, shifting
`.bss` forward and breaking the `dest_offset_id` address.

**Fix**: On clang, MMIO pointers get `__attribute__((section(".data.mmio_ptrs")))`.
The firmware linker script places this section **after** `.bss`:
```
.bss : { *(.bss.0_dest) *(.bss.1_opinfo) *(.bss .bss.*) }
.data.mmio_ptrs : ALIGN(4) { *(.data.mmio_ptrs) }
```

### 3.4  Kernel LDM Budget

The kernel gets `(4096 - 192) - (firmware LDM used)` bytes.
Typical: **1500-1800 bytes** for kernel .data + .bss + stack.

Clang kernels use more .data than GCC because `constexpr` arrays
(e.g., `unpack_dst_format[64]` = 256 bytes) go to `.rodata` which the kernel
linker script maps into the `.data` output section.  GCC constant-folds them.

---

## 4  Firmware Boot Sequence

`trisc.cc::main()` runs once at device initialization:

```
1. configure_csr()           — Enable/disable L1 cache, gathering, memory ordering
2. do_crt1(data_lma)         — Zero .bss, copy .data from text image to LDM
3. Zero REGFILE              — 64 GPRs set to 0
4. reset_cfg_state_id()      — cfg_state_id = 0
5. Init PRNG seed            — cfg[PRNG_SEED] = 0; wait 600 cycles
6. Set mailbox coords        — my_logical_x/y from core_info
7. Signal DONE               — *trisc_run = RUN_SYNC_MSG_DONE
8. Enter dispatch loop       — while(1) { wait GO → setup CBs → call kernel → tensix_sync → DONE }
```

### 4.1  `do_crt1()` — C Runtime Initialization

```c
void do_crt1(uint32_t* data_image) {
    wzerorange(__ldm_bss_start, __ldm_bss_end);              // Zero BSS
    if (__ldm_data_start != data_image)
        l1_to_local_mem_copy(__ldm_data_start, data_image,   // Copy .data
                             __ldm_data_end - __ldm_data_start);
}
```

The `data_image` (LMA) is computed via `auipc` — PC-relative addressing that
gives the correct runtime address regardless of where the binary is loaded.
This is critical because kernels are loaded into `kernel_config_buffer` at an
address different from the linker's VMA.

### 4.2  `init_sync_registers()` — CB Counter Reset

Called by TRISC0 only, before the first kernel dispatch:
```c
void init_sync_registers() {
    for (uint32_t op = 0; op < NUM_CIRCULAR_BUFFERS; op++) {
        *get_cb_tiles_received_ptr(op) = 0;
        *get_cb_tiles_acked_ptr(op) = 0;
    }
}
```

These are NOC stream scratch registers used as CB synchronization mailboxes.

---

## 5  Inter-Kernel State

**What persists across kernel dispatches** (firmware does NOT reset these):

| State                    | Persists? | Why it matters |
|--------------------------|-----------|----------------|
| Tensix config registers  | YES       | tilize/matmul/SFPU modes persist |
| Coprocessor semaphores   | YES       | Imbalanced semaphores deadlock next kernel |
| MOP configuration        | YES       | MOP loop/start/end ops persist |
| Replay buffer contents   | YES       | **Root cause of tilize→matmul hang** |
| REGFILE (GPR) values     | YES       | Address counters, DMA regs persist |
| `cfg_state_id`           | YES       | In firmware BSS, not reset per-kernel |
| `dest_offset_id`         | YES       | In firmware BSS |
| CB interface pointers    | Overwritten | Firmware re-initializes from launch_msg |

**What is reset per kernel** (by the kernel's `do_crt1`):

| State                    | Reset? | How |
|--------------------------|--------|-----|
| Kernel BSS               | YES    | `wzerorange()` in `_start` |
| Kernel .data             | YES    | `l1_to_local_mem_copy()` from text image |
| `unp_cfg_context`        | YES    | Kernel BSS (zeroed) |

**What `tensix_sync()` does**: Emits `TTI_STALLWAIT(STALL_SYNC, ALL)` which
waits for **all** pending coprocessor operations to complete.  It does NOT
reset semaphores, config registers, MOP state, or the replay buffer.

---

## 6  Semaphores

### 6.1  Hardware Semaphores

Eight 4-bit saturating counters at `PC_BUF_BASE + PC_BUF_SEMAPHORE_BASE`:

| Index | Name                | Purpose |
|-------|---------------------|---------|
| 0     | FPU_SFPU            | FPU ↔ SFPU synchronization |
| 1     | MATH_PACK           | MATH ↔ PACK dest register handoff |
| 2     | UNPACK_TO_DEST      | UNPACK ↔ MATH dest sync |
| 3     | UNPACK_OPERAND_SYNC | Multi-operand sync |
| 4     | PACK_DONE           | Pack iteration complete |
| 5     | UNPACK_SYNC         | TRISC ↔ UNPACK hardware sync |
| 6     | UNPACK_MATH_DONE    | Unpack or math iteration sync |
| 7     | MATH_DONE           | Math completion signal |

### 6.2  Access Patterns

```c
// Read: returns current 4-bit value (0-15)
uint8_t val = pc_buf_base[PC_BUF_SEMAPHORE_BASE + index];

// SEMPOST (increment): write 0 (LSB clear)
pc_buf_base[PC_BUF_SEMAPHORE_BASE + index] = 0;

// SEMGET (decrement): write 1 (LSB set)
pc_buf_base[PC_BUF_SEMAPHORE_BASE + index] = 1;

// Tensix T6 SEMPOST: TTI_SEMPOST(sem_sel)
// Tensix T6 SEMGET:  TTI_SEMGET(sem_sel)
// Tensix SEMINIT:    TTI_SEMINIT(max_value, init_value, sem_sel)
// Tensix SEMWAIT:    TTI_SEMWAIT(stall_res, sem_sel, condition)
```

### 6.3  Blocking Store Pattern

`mop_sync()` and other sync primitives use `store_blocking()`:
```c
void store_blocking(volatile T *ptr, U val) {
    asm volatile(
        "sw %[raw], (%[ptr])\n\t"   // Store
        "lw %[raw], (%[ptr])\n\t"   // Load-back (waits for store to complete)
        "and x0, x0, %[raw]"        // Use loaded value (creates pipeline dependency)
        : [raw] "+r"(raw) : [ptr] "r"(ptr) : "memory");
}
```
This pattern guarantees the store is visible before execution continues.
Required because BH RISC-V has relaxed memory ordering for MMIO writes.

---

## 7  MOP (Micro-Operation) System

The MOP expander auto-generates Tensix instruction sequences from a compact
configuration.  It implements a double-nested loop:

```
start_op0
for outer in 0..outer_loop_len:
    for inner in 0..inner_loop_len:
        loop_op0
        loop_op1
    end_op0
end_op1
```

Config registers at `TENSIX_MOP_CFG_BASE` (`0xFFB8_0000`):

| Offset | Field              | Purpose |
|--------|--------------------|---------|
| [0]    | outer_loop_len     | Outer loop iteration count |
| [1]    | inner_loop_len     | Inner loop iteration count |
| [2]    | start_op0          | Instruction before loops |
| [3]    | end_op0            | Instruction after inner loop |
| [4]    | end_op1            | Instruction after outer loop |
| [5]    | loop_op0           | First loop body instruction |
| [6]    | loop_op1           | Second loop body instruction |
| [7]    | loop0_last_instr   | Outer loop last-iteration replacement |
| [8]    | loop1_last_instr   | Inner loop last-iteration replacement |

MOP config values are Tensix instruction words **without ROL2 encoding** —
the MOP hardware applies encoding when generating instructions.

Trigger: `TT_MOP(execute, count-1, zmask)` or `ckernel_template::run()`.

---

## 8  Replay Buffer

The Tensix replay buffer caches instruction sequences for repeated execution.

| Property        | Value |
|-----------------|-------|
| Entries         | 32    |
| Entry size      | 32-bit instruction word |
| Total           | 128 bytes |

### 8.1  Operations

```
RECORD:  ttreplay start=S, len=N, exec=0, record=1
         Captures the next N inline Tensix instructions into buffer[S..S+N-1]

RECORD+EXEC: ttreplay start=S, len=N, exec=1, record=1
         Captures AND executes the next N instructions

REPLAY:  ttreplay start=S, len=N, exec=0, record=0
         Executes buffer[S..S+N-1] from the replay buffer

REPLAY_INSN: Embedded in MOP config (loop_op0/loop_op1)
         MOP auto-generates a replay instruction during the loop
```

### 8.2  The Stale Replay Buffer Bug

**Root cause of the tilize→matmul hang** (fixed by `lltt.h`):

The matmul UNPACK MOP uses `ttunpacr` with replay flag=1, which triggers
replay execution after the unpack operation.  The replay buffer must contain
the matmul's unpack config sequence (loaded via `lltt::record()`).

GCC's `lltt.h` uses `__builtin_rvtt_ttreplay` (a GCC-specific intrinsic).
LLVM had no equivalent, so `lltt::record()` was never called, and the replay
buffer was never loaded.

After a tilize kernel, the replay buffer contained tilize-specific unpack
instructions.  The matmul kernel replayed these stale instructions instead of
matmul unpack instructions → hardware deadlock.

**Why it only manifested after tilize**: The tilize kernel uses the replay
buffer for its own UNPACK sequence.  Other kernels (gelu, sigmoid) don't use
the replay buffer, leaving it empty/benign.

---

## 9  Tensix Instruction Encoding

### 9.1  `.ttinsn` and ROL2

All Tensix inline instructions are emitted via:
```c
#define INSTRUCTION_WORD(x) __asm__ __volatile__(".ttinsn %0" : : "i"((x)))
```

The `.ttinsn` assembler directive applies **ROL2 encoding** (rotate left by 2):
```
encoded = (value << 2) | (value >> 30)
```

Both GCC and LLVM apply ROL2 identically — verified by test.

### 9.2  Instruction Dispatch Mechanisms

| Mechanism          | How                                  | When used |
|--------------------|--------------------------------------|-----------|
| Inline             | `INSTRUCTION_WORD(op)` → `.ttinsn`   | Fixed instruction sequences |
| instrn_buffer      | `sw value, 0(instrn_buf_ptr)`        | Dynamic instructions (SETC16 with runtime values) |
| MOP auto-generate  | Write to `TENSIX_MOP_CFG_BASE`, trigger with `TT_MOP` | Repeated loop sequences |

### 9.3  Key Opcodes

| Opcode | Instruction | Purpose |
|--------|-------------|---------|
| 0x02   | NOP         | No operation |
| 0x04   | REPLAY      | Record/replay instruction buffer |
| 0x42   | UNPACR      | Unpack operation |
| 0x43   | UNPACR_NOP  | Unpack NOP variant |
| 0xa3   | SEMINIT     | Initialize semaphore |
| 0xa4   | SEMPOST     | Increment semaphore (T6) |
| 0xa6   | SEMWAIT     | Wait on semaphore condition |
| 0xa2   | STALLWAIT   | Stall until resource idle |
| 0xb0   | WRCFG       | Write config register |
| 0xb1   | RDCFG       | Read config register |
| 0xb2   | SETC16      | Set 16-bit config value |

---

## 10  GCC vs LLVM Compilation Differences

### 10.1  Target Specification

| Property      | GCC                          | LLVM/clang |
|---------------|------------------------------|------------|
| Target triple | `riscv-tt-elf`               | `riscv32-unknown-elf` |
| Architecture  | `-mcpu=tt-bh-tensix`         | `-march=rv32im_zba_zbb_xttsfpu_xttsfpubh` |
| ABI           | (implicit)                   | `-mabi=ilp32` |
| SFPU builtins | `__builtin_rvtt_*`           | `__builtin_riscv_tt_*` (via `sfpi_compat.h`) |
| Tensix builtins| `__builtin_rvtt_ttreplay`   | Not supported → use `lltt.h` shim |

### 10.2  Constant Folding Differences

GCC at `-O3` aggressively constant-folds:
- MMIO pointer initializers → no `.data` section
- `constexpr` arrays (format tables) → inlined into code

LLVM at `-Os` is more conservative:
- MMIO pointers → stored in `.data` (28+ bytes)
- `constexpr` arrays → stored in `.rodata` → linker maps to `.data` output section

This consumes **up to 300+ bytes** of precious kernel LDM that GCC doesn't use.

### 10.3  Filtered/Added Compilation Flags

**GCC flags filtered for clang:**
`-fno-tree-loop-distribute-patterns`, `-flto=auto`, `-g/-g1/-g2/-g3`,
`-fdump-*`, `-Wno-error=multistatement-macros`,
`-Wno-error=unused-but-set-variable`

**Clang-specific flags added:**
`-Wno-unknown-attributes`, `-Wno-empty-body`, `-Wno-c99-designator`,
`-Wno-c++11-narrowing`, `-Wno-unused-but-set-variable`,
`-Wno-constant-logical-operand`, `-Wno-unused-private-field`,
`-Wno-mismatched-tags`, `-Wno-incompatible-function-pointer-types`,
`-Wno-ignored-attributes`, `-Wno-missing-braces`,
`-Wno-unused-lambda-capture`, `-Wno-nan-infinity-disabled`

### 10.4  `cb_interface` MATH Placeholder

GCC optimizes away the `cb_interface` reference in MATH kernels.  Clang emits
it.  The firmware provides a `uint8_t cb_interface[2048]` placeholder for MATH
builds to satisfy the linker:
```c
#if defined(UCK_CHLKC_MATH)
uint8_t cb_interface[2048] __attribute__((used, aligned(4)));
#else
CBInterface cb_interface[NUM_CIRCULAR_BUFFERS] __attribute__((used));
#endif
```

### 10.5  Include Paths

| Purpose          | GCC path                       | LLVM path |
|------------------|--------------------------------|-----------|
| SFPU builtins    | `runtime/sfpi/include/`        | `runtime/llvm-sfpu/include/` |
| Clang wrappers   | N/A                            | `runtime/llvm-sfpu/clang_include/` |
| Compatibility    | N/A                            | `sfpi_compat.h` (pre-included via `-include`) |
| Tensix replay    | `runtime/sfpi/include/lltt.h`  | `runtime/llvm-sfpu/clang_include/lltt.h` |

### 10.6  Device Profiler

The device profiler (`TT_METAL_DEVICE_PROFILER=1`) emits TRISC-KERNEL zone
markers only when `-g` debug info is present.  LLVM compilation filters out
`-g`, so **TRISC profiler zones are absent from LLVM builds**.  BRISC/NCRISC
zones work in both.

---

## 11  Debugging Checklist

When a kernel hangs on silicon with LLVM but works with GCC:

1. **Check BSS layout**: `llvm-objdump -t` — is `dest_offset_id` at
   `0xFFB0_0000`?  Are MMIO pointers in `.data.mmio_ptrs` (not `.data`)?

2. **Check .data size**: Compare `.data` section size.  LLVM kernels can have
   100-300 bytes more.  Verify `(data + bss + min_stack) < LDM budget`.

3. **Check replay buffer**: Does the kernel use MOP with replay?  Look for
   `ttunpacr` with replay flag (bit 25).  If present, ensure `lltt::record()`
   was called to populate the buffer.

4. **Check semaphore balance**: Count semaphore POST and GET operations in
   the kernel.  They must be balanced per kernel invocation.

5. **Check Tensix instruction encoding**: Use GCC's `riscv-tt-elf-objdump`
   to decode Tensix instructions (LLVM's objdump shows `<unknown>`).

6. **Check inter-kernel state**: Run the failing kernel FIRST after device
   reset.  If it works alone but fails after another kernel, the previous
   kernel left dirty config/replay/semaphore state.

7. **Check volatile semantics**: Ensure all MMIO accesses go through
   `volatile` pointers.  `tt_reg_ptr` and `tt_l1_ptr` are GCC attributes
   that clang ignores — the `volatile` keyword on the pointed-to type is
   what prevents optimization.

---

## 12  Bugs Found and Fixed

| Bug | Symptom | Root Cause | Fix |
|-----|---------|------------|-----|
| BSS layout mismatch | Matmul deadlock | Clang put MMIO ptrs in `.data`, shifting `dest_offset_id` | `.data.mmio_ptrs` section + `.bss.0_dest` named sections |
| Missing `lltt.h` | tilize→matmul hang | Replay buffer never loaded; MOP replayed stale tilize instructions | Created `runtime/llvm-sfpu/clang_include/lltt.h` |
| `cb_interface` undefined | Link error on MATH builds | Clang emits reference that GCC optimizes away | `uint8_t cb_interface[2048]` placeholder |
| `sfpwriteconfig_v` corruption | PCC 0.53 (random output) | Builtin mapped vector as mod1 instead of writing to L0 | Fixed `sfpi_builtins.h` two-step sequence |
| `sfpsetsgn_v` sign dropped | Wrong gelu results | Builtin dropped sign source argument | New two-register intrinsic in LLVM backend |
| `__has_builtin` leak | stdlib compilation errors | Override never popped after SFPI mappings | `#pragma pop_macro("__has_builtin")` |
| CCMap CC_GT/CC_LTE | Wrong predication | Used COMP mode (flips CC) instead of testing register | Map to GTE0/LT0 |
