# LLVM SFPU Backend — Status & Results

## Overview

The LLVM SFPU backend replaces Tenstorrent's GCC-based SFPU compiler with
LLVM/clang for Blackhole (p100a/p150) silicon. When enabled via
`TT_METAL_USE_LLVM_SFPU=1`, ALL Tensix compute kernels (SFPU eltwise,
matmul, tilize, etc.) are compiled with clang instead of GCC.

## Performance (BH p100a, after JIT warmup)

| Op | Size | GCC (µs) | LLVM (µs) | Speedup | PCC |
|---|---|---|---|---|---|
| gelu (approx) | 32×32 | 8.7 | 8.0 | **+9%** | 0.9999 |
| gelu (approx) | 128×768 | 10.1 | 10.0 | +1% | 0.9999 |
| sigmoid | 32×32 | 8.1 | 7.7 | +5% | 0.9999 |
| sigmoid | 128×768 | 12.7 | 10.1 | **+26%** | 0.9999 |
| exp | 32×32 | 8.8 | 8.5 | +4% | 0.993 |
| exp | 128×768 | 10.9 | 10.8 | +1% | 0.992 |

- **gelu_appx and sigmoid match GCC quality** (PCC > 0.999)
- **exp has a remaining gap** from the CDF polynomial path
- LLVM is **faster across the board**, up to 26% on larger tensors

## Key Bugs Fixed

### 1. `sfpwriteconfig_v` — config register corruption (PCC 0.53 → 0.9999)
**File:** `runtime/sfpi/include/sfpi_builtins.h`

The builtin mapped the vector value as `mod1` instead of writing to L0 first.
This corrupted `vConstFloatPrgm0` (0.5 for gelu), producing random output
for all kernels using config registers.

### 2. BSS layout mismatch — matmul hang
**Files:** `tt_metal/hw/firmware/src/tt-1xx/trisc.cc`, linker scripts

Clang puts initialized MMIO pointers (`reg_base`, `pc_buf_base`, `regfile`)
in `.data`. GCC folds them as constants (empty `.data`). This shifted `.bss`,
moving `dest_offset_id` to the wrong address. The matmul kernel read
the wrong DEST buffer half → deadlock.

**Fix:** Section attribute `.data.mmio_ptrs` with linker script placement
after `.bss`. Named sections for `dest_offset_id` to force BSS start.

### 3. `sfpsetsgn_v` — sign argument dropped
**Files:** `sfpi_builtins.h`, new `sfpsetsgn_v` intrinsic in LLVM backend

The builtin dropped the sign source argument. Created a new two-register
intrinsic with proper tied constraint in the LLVM backend.

### 4. `__has_builtin` override leak
**File:** `switchover/sfpi_compat.h`

The `#define __has_builtin(x) 1` override was never popped, causing
standard library headers to see unsupported builtins as available.

### 5. CCMap CC_GT/CC_LTE — used COMP mode
**File:** `RISCVISelDAGToDAG.cpp`

Hardware mod1 bit 3 (COMP=8) complements existing CC without testing
the register. Fixed to use GTE0/LT0 instead.

### 6. Missing `sfpencc` before `sfpstore`
**File:** `RISCVXttSFPUPeephole.cpp`

GCC emits `sfpencc` to re-enable all CC lanes after predication blocks.
Added peephole to insert it before every `sfpstore` in predicated functions.

### 7. BH NOP insertion for 2-cycle SFPU instructions
**File:** `RISCVXttSFPUErrata.cpp`

Extended E-004 errata handling to insert NOPs after ALL 2-cycle instructions
on BH when the next instruction reads the output register.

## Architecture

```
sfpi_compat.h (pre-included via -include)
  ↓ maps __builtin_rvtt_* → __builtin_riscv_tt_*
sfpi_builtins.h
  ↓ maps arch-independent builtins
clang/LLVM backend
  ↓ IntrinsicsRISCVXttSFPU.td → ISel → MIR passes → binary
  ↓ Passes: Combine → Synth → Liveness → Cluster → Constraints →
  ↓         PredElide → Peephole → Errata → Replay
firmware/kernel JIT compilation
  ↓ trisc.cc (firmware) + trisck.cc (kernel) → linked ELF
device execution
```

## Testing

```bash
# Toolchain verification (no hardware)
cd llvm-tt-sfpu
LLVM_DIR=/work/llvm/llvm-project-sfpu/build/bin ./switchover/test_compile_kernel.sh bh
# Expected: 10/10 pass

# Full kernel compilation (no hardware)
TT_METAL_HOME=/work/llvm/tt-metal-sfpu \
LLVM_DIR=/work/llvm/llvm-project-sfpu/build/bin \
./tests/test_all_kernels.sh
# Expected: 91/91 pass

# Silicon test
cd tt-metal-sfpu
source .venv/bin/activate
TT_METAL_USE_LLVM_SFPU=1 python -c "
import torch, ttnn, numpy as np
device = ttnn.open_device(device_id=0)
x = ttnn.from_torch(torch.randn(1,1,32,32), dtype=ttnn.bfloat16,
                     layout=ttnn.Layout.TILE, device=device)
y = ttnn.to_torch(ttnn.gelu(x, fast_and_approximate_mode=True))
ref = torch.nn.functional.gelu(torch.randn(1,1,32,32))
print(f'PCC={np.corrcoef(ref.flatten(), y.flatten())[0,1]:.6f}')
ttnn.close_device(device)
"
```

## Known Limitations

- **First-time JIT compile is slow** (~5 min for matmul kernels at -O3).
  Subsequent runs use the kernel cache (`~/.cache/tt-metal-cache/`).
- **exp/gelu_accurate PCC ~0.99** (vs GCC 0.99999) from CDF polynomial path.
- **Device close may hang** — use `signal.alarm()` + `os._exit(0)` workaround.
- **`TT_METAL_FORCE_JIT_COMPILE=1`** forces recompilation (slow but ensures
  latest compiler changes are used).
