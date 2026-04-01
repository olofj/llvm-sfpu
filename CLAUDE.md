# LLVM SFPU Project

LLVM backend for Tenstorrent SFPU (Special Function Processing Unit) hardware.

## Repository Layout

- `llvm-project-sfpu/` — Fork of LLVM with SFPU extensions (custom builtins, ISel patterns, RISC-V target additions)
- `llvm-tt-sfpu/` — Test harness, compatibility headers, switchover tooling
- `tt-metal-sfpu/` — Fork of tt-metal with LLVM SFPU compiler integration

## Building LLVM/Clang

```bash
cd /work/llvm/llvm-project-sfpu
cmake -G Ninja -B build \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_TARGETS_TO_BUILD="RISCV" \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_PARALLEL_LINK_JOBS=2 \
  llvm
ninja -C build -j$(nproc) clang llc llvm-mc llvm-objdump
```

## Setting Up the Runtime

```bash
cd /work/llvm/tt-metal-sfpu
mkdir -p runtime/llvm-sfpu/bin runtime/llvm-sfpu/include
ln -sf /work/llvm/llvm-project-sfpu/build/bin/clang++ runtime/llvm-sfpu/bin/clang++
cp /work/llvm/llvm-tt-sfpu/switchover/sfpi_compat.h runtime/llvm-sfpu/include/
```

## Testing

### Toolchain verification (no hardware needed)

```bash
cd /work/llvm/llvm-tt-sfpu
LLVM_DIR=/work/llvm/llvm-project-sfpu/build/bin ./switchover/test_compile_kernel.sh bh
# Expected: 10/10 pass
```

### Full kernel smoke test (no hardware needed)

Requires tt_llk submodule: `cd tt-metal-sfpu && git submodule update --init tt_metal/third_party/tt_llk`

```bash
cd /work/llvm/llvm-tt-sfpu
TT_METAL_HOME=/work/llvm/tt-metal-sfpu \
LLVM_DIR=/work/llvm/llvm-project-sfpu/build/bin \
./tests/test_all_kernels.sh
# Expected: 87/91 pass (4 failures from SFPI header version mismatch, not LLVM issues)
```

### Check status of hardware (requires WH or BH card)

```bash
(. ~/.tenstorrent-venv/bin/activate && tt-smi -s)
```

### Hardware test (requires BH card)

```bash
cd /work/llvm/tt-metal-sfpu
export TT_METAL_USE_LLVM_SFPU=1
python -m pytest tests/tt_eager/python_api_testing/unit_testing/misc/test_eltwise_unary.py -k "gelu" -x
```

Compare output numerics with and without `TT_METAL_USE_LLVM_SFPU=1` (should be identical).

Check perf with `TT_METAL_DEVICE_PROFILER=1` to measure instruction reduction impact on wall-clock time.

## Hardware Reset

If the Tenstorrent hardware gets into a bad state, reset it with tt-smi:

```bash
(. ~/.tenstorrent-venv/bin/activate && tt-smi -r)
```

The venv must be activated first since tt-smi is installed there.

## Sysroot / Header Setup

Uses the GCC 15 SFPI toolchain at `/opt/tenstorrent/sfpi/` (riscv-tt-elf triple).
The test script auto-detects the sysroot layout. Key adaptations for clang compatibility:

- **sfpi_compat.h**: Pre-includes `<type_traits>` etc. before overriding `__has_builtin`,
  so GCC 15's libstdc++ doesn't try to use `__remove_reference` (clang 19 lacks it).
- **sfpi_builtins.h** (custom, in `llvm-tt-sfpu/switchover/`): Maps short-form GCC builtins
  directly to `__builtin_riscv_tt_*` LLVM intrinsics. Deployed to `runtime/sfpi/include/`.
- **ckernel.h**: Line 316 has `uint32_t short` (GCC extension), wrapped in `#ifndef __clang__`.

## Prerequisites

- cmake, ninja-build, gcc/g++
- Python 3.8+ with tt-metal dependencies
- GCC SFPU sysroot at `/opt/tenstorrent/sfpi/`
