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
ninja -C build -j$(nproc) clang llc llvm-mc
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
SFPI_GCC_TARGET_DIR=/work/llvm/gcc12-compat \
SFPI_GCC12_CXX=/usr/riscv64-linux-gnu/include/c++/12 \
SFPI_SYSROOT=/opt/tenstorrent/sfpi/compiler/riscv-tt-elf/include \
TT_METAL_HOME=/work/llvm/tt-metal-sfpu \
LLVM_DIR=/work/llvm/llvm-project-sfpu/build/bin \
./tests/test_all_kernels.sh
# Expected: 87/91 pass (4 failures from SFPI header version mismatch, not LLVM issues)
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
(. .tenstorrent-venv/bin/activate && tt-smi -r)
```

The venv must be activated first since tt-smi is installed there.

## Sysroot / Header Setup

This system has a GCC 15 SFPI toolchain at `/opt/tenstorrent/sfpi/` (riscv-tt-elf triple). The LLVM
SFPU project was developed against GCC 12 headers. Key adaptations:

- **C++ headers**: Use Debian's `libstdc++-12-dev-riscv64-cross` (`/usr/riscv64-linux-gnu/include/c++/12`)
  because GCC 14+/15 headers use `__remove_reference` which clang 19 doesn't support.
- **C sysroot**: Use TT's bare-metal sysroot (`/opt/tenstorrent/sfpi/compiler/riscv-tt-elf/include`),
  not the riscv64 glibc one.
- **GCC target dir**: A patched target-specific directory at `/work/llvm/gcc12-compat/` provides
  `bits/c++config.h` and a minimal `bits/os_defines.h` for newlib.
- **SFPI headers**: `/work/llvm/tt-metal-sfpu/runtime/sfpi/include/` has symlinks to system SFPI headers
  plus a custom `sfpi_builtins.h` that maps short-form GCC builtins directly to LLVM intrinsics.
- **ckernel.h**: Line 316 has `uint32_t short` (GCC extension), wrapped in `#ifndef __clang__`.

## Prerequisites

- cmake, ninja-build, gcc/g++
- `libstdc++-12-dev-riscv64-cross` (for GCC 12 C++ headers compatible with clang 19)
- Python 3.8+ with tt-metal dependencies
- GCC SFPU sysroot at `/opt/tenstorrent/sfpi/`
