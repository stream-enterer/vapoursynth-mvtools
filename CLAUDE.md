# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MVTools is a VapourSynth plugin providing motion estimation and compensation filters for video processing. It is a port of Avisynth MVTools 2.5.11.20 plus Depan plugins. The plugin identifier is `com.nodame.mvtools`, namespace `mv.`, version 24. Licensed under GPL v2.

## Build Commands

```bash
meson setup build
ninja -C build
```

The output is a shared module (`libmvtools.so` / `libmvtools.dll`). Install with `ninja -C build install`.

**Dependencies:** VapourSynth (API v4), fftw3f (single-precision FFTW3), nasm (x86 assembly).

**Meson option:** `-Dwith_nasm=<path>` to specify the NASM executable (default: `nasm`).

There is no test suite. Verification is done manually via VapourSynth scripts.

## Architecture

**Entry point:** `src/EntryPoint.c` — `VapourSynthPluginInit2` registers all filters. Each filter has its own source file with a `Register` function.

**Core pipeline:** `mv.Super` builds a hierarchical pyramid → `mv.Analyse` performs block matching → consumer filters (`mv.Compensate`, `mv.Degrain1/2/3`, `mv.Flow*`, etc.) read serialized motion vectors from frame properties.

### Key layers

- **Filter implementations** (`MVAnalyse.c`, `MVCompensate.c`, `MVDegrains.cpp`, `MVFlow.cpp`, `MVDepan.cpp`, etc.) — VapourSynth getFrame callbacks and filter registration. Motion vectors are passed between filters via frame properties.
- **Motion search engine** (`PlaneOfBlocks.cpp`, `GroupOfPlanes.c`) — `PlaneOfBlocks` does block matching at one pyramid level; `GroupOfPlanes` manages the full coarse-to-fine hierarchical search.
- **Frame/plane management** (`MVFrame.cpp`) — `MVPlane` (single padded plane), `MVFrame` (3-plane YUV), `MVGroupOfFrames` (multi-level pyramid).
- **Vector deserialization** (`Fakery.c`) — Read-only view of serialized motion vectors stored in frame properties; used by all consumer filters.
- **Analysis metadata** (`MVAnalysisData.h`) — Shared struct carrying search parameters between filters.

### SIMD dispatch

Optimized functions (SAD, overlap, resize, mask, degrain) are selected at runtime via function pointers based on CPU capabilities detected at plugin load (`CPU.c`, `g_cpuinfo`).

- **x86 NASM assembly** (`src/asm/`) — SAD/pixel routines ported from x264.
- **AVX2 C++ variants** (`*_AVX2.cpp`) — Compiled into a separate static library with `-mavx2 -mtune=haswell`, linked into the main module.
- **AArch64 NEON** (`src/asm/aarch64-pixel-a.S`) — ARM assembly. SSE2 intrinsics are translated via `sse2neon.h`.

## Code Conventions

- **Mixed C/C++:** Most filters are C99. C++ (C++11) is used for templates (`MVDegrains.h` — pixel-type-generic degrain) and RAII. Headers use `extern "C"` guards.
- **Init/Deinit pattern:** Structs are managed with `fooInit`/`fooDeinit` pairs (e.g., `mvpInit`/`mvpDeinit`, `gopInit`/`gopDeinit`).
- **Naming:** Filter files are `MV<FilterName>.c/.cpp`. Internal modules use descriptive names (`PlaneOfBlocks`, `GroupOfPlanes`, `Fakery`).
- **Compiler flags:** `-Wall -Wextra -Wshadow -fvisibility=hidden`. LTO enabled by default.
