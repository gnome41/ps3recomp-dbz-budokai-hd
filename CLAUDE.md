# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

**ps3recomp** is a static recompilation SDK for PlayStation 3 games. It translates PS3 PowerPC (PPU) binaries to native C/C++ ahead-of-time via a Python pipeline, then links the generated code against a runtime library of HLE (High-Level Emulation) stubs for all PS3 system libraries.

The repo has two distinct layers:
1. **SDK / Runtime** — this root directory: Python tools, HLE libs, runtime core, public headers
2. **Game projects** — subdirectories like `dbz-budokai-hd/`, one per game being ported

## Build Commands

### Build the SDK runtime library
```bash
cmake -B build -G Ninja
cmake --build build
```
Produces `build/ps3recomp_runtime.lib` (Windows) or `build/libps3recomp_runtime.a`.

**Prerequisites:** CMake 3.20+, C17/C++20 compiler (MSVC 19.x, GCC 12+, or Clang 14+), Python 3.9+.

On Windows: launch from a VS 2022 Developer Command Prompt for the correct MSVC environment.

### Build a game project (e.g., dbz-budokai-hd)
```bat
cd dbz-budokai-hd
build_run.bat        # builds via Ninja, echoes BUILD_EXIT=
run.bat              # runs build\dbz-budokai-hd.exe ..\game\EBOOT.elf
```
Or manually:
```bash
cmake -B build -G Ninja -DPS3RECOMP_DIR=/path/to/ps3recomp
cmake --build build
```

### Recompiler pipeline (Python tools)
```bash
# 1. Parse the ELF
python tools/elf_parser.py path/to/EBOOT.ELF --output analysis/

# 2. Disassemble PPU code
python tools/ppu_disasm.py path/to/EBOOT.ELF --output disasm/

# 3. Lift to C (generates recompiled/ppu_recomp.cpp + ppu_recomp.h)
python tools/ppu_lifter.py disasm/ --output recompiled/
```

There are no automated tests (`PS3RECOMP_BUILD_TESTS` option exists but is `OFF` by default). Verification is done by running the game exe and reading stderr diagnostics.

## Repository Structure

```
ps3recomp/
├── tools/              # Python pipeline: elf_parser, ppu_disasm, ppu_lifter, spu_*,
│                       #   find_functions, nid_database, prx_analyzer, generate_stubs
├── runtime/            # C/C++ runtime core: ppu/, spu/, memory/, syscalls/
├── libs/               # HLE module implementations (97 complete, C files)
│   ├── audio/          # cellAudio, cellMic, cellVoice
│   ├── video/          # cellGcmSys, cellResc, cellVideoOut, RSX backends
│   ├── input/          # cellPad (XInput/SDL2), cellKb, cellMouse, cellGem
│   ├── network/        # sys_net, cellNet, cellHttp/s, sceNp*
│   ├── filesystem/     # cellFs, cellGame, cellSaveData
│   ├── system/         # cellSysutil, cellSysmodule, sysPrxForUser
│   ├── spurs/          # cellSpurs, cellFiber, cellSpursJq, cellDaisy
│   ├── sync/           # cellSync, cellSync2
│   ├── codec/          # cellPngDec, cellJpgDec, cellPamf, cellDmux, cellVdec, cellAdec
│   ├── font/           # cellFont, cellFreeType, cellL10n
│   ├── hardware/       # cellUsbd, cellCamera, cellGem
│   └── misc/           # cellScreenshot, cellImeJp, cellVideoExport, etc.
├── include/ps3emu/     # Public headers: ps3types.h, endian.h, module.h, nid.h, error_codes.h
├── config/             # example.toml — reference recompiler config
├── templates/project/  # Starter template for new game ports
├── docs/               # 15 docs covering architecture, syscalls, modules, porting guide
├── BLES01658/          # Game disc image (PS3_GAME/USRDIR/ contains EBOOT.BIN + data)
└── dbz-budokai-hd/     # Active game port project (has its own CLAUDE.md)
```

**Adding a new HLE module:** create `.h` + `.c` in the appropriate `libs/<category>/` dir — CMake uses `GLOB_RECURSE` so no CMakeLists.txt edits needed.

## Key Architecture Concepts

### Memory Model
Guest uses 32-bit effective addresses despite the 64-bit PPU. The runtime reserves a 4 GB host region (`VirtualAlloc`/`mmap`) and translates: `host_ptr = vm_base + (uint32_t)guest_addr`. All memory reads/writes byte-swap for big-endian. **Never raw-cast through guest pointers** — always use `vm_read*/vm_write*` helpers.

Guest address map:
| Range | Region |
|---|---|
| `0x00010000–0x0FFFFFFF` | ELF text + data |
| `0x10000000–0x1FFFFFFF` | PRX module space |
| `0x20000000–0x2FFFFFFF` | Stack / TLS |
| `0x30000000–0x3FFFFFFF` | Heap (user malloc) |
| `0xD0000000–0xDFFFFFFF` | RSX IO-mapped main memory |

### PPU Context
Every lifted function has signature `void func_XXXXXXXX(ppu_context* ctx)`. The struct (`include/ps3emu/` / `recompiled/ppu_recomp.h`) holds `gpr[32]`, `fpr[32]`, `vr[32]` (VMX/AltiVec), `cr`, `lr`, `ctr`, `xer`, `fpscr`, `vscr`, `pc`, `cia`, plus atomic reservation fields and `thread_id`.

### Trampoline Pattern
Cross-function tail calls set a TLS function pointer (`g_trampoline_fn`) instead of calling directly. **Always drain it** with `DRAIN_TRAMPOLINE(ctx)` after any call that might set it — failing to do so causes unbounded host stack growth.

### NID System
PS3 imports are identified by 32-bit NIDs (`SHA-1(function_name)[:4]`). The HLE framework resolves NIDs at load time via the module registry (`include/ps3emu/module.h`). To register a new HLE function, use the `DECLARE_PS3_MODULE` / `REGISTER_FUNC` / `REGISTER_FNID` macros in your `.c` file.

### Indirect Calls
`bctr`/`bctrl` PPU instructions emit `ps3_indirect_call(ctx)` with `ctx->ctr` = guest target address. Resolution chain: generated address table (`ppu_resolve_addr`) → hand-lifted extras (`ppu_resolve_extra`) → log + stub. Unknown addresses log once and return `gpr[3] = 0`.

### RSX Graphics Pipeline
```
cellGcmSys (HLE)  →  RSX Command Processor  →  Graphics Backend
                        NV47xx FIFO parsing       D3D12 / Vulkan / null
```
RSX MMIO control registers (put/get/ref) are accessed in **host-endian** (via `lwbrx`/`stwbrx` in recompiled code) — do not pass them through `vm_write32` (which byte-swaps).

### Module Registration
Modules self-register at static init time via C++ constructors in the `DECLARE_PS3_MODULE` macro. The global `g_ps3_module_registry` (max 128 modules) is the lookup table for all NID resolution.

## Types and Conventions

- `ps3types.h`: `u8`/`u16`/`u32`/`u64`, `s8`–`s64`, `be_t<T>` (C++ big-endian wrapper)
- `be_t<T>` is C++ only — C files use raw `u32`/`u64` with explicit byte-swapping via `endian.h`
- Error codes: `CELL_OK = 0`, `CELL_IS_ERROR(rc)` macro, error constants in `error_codes.h`
- `-fno-strict-aliasing` is required for `be_t<T>` endian conversion through unions

## Docs Index

| Document | When to read it |
|---|---|
| `docs/ARCHITECTURE.md` | Cell CPU overview, pipeline stages, memory map, RPCS3 comparison |
| `docs/RUNTIME.md` | VM, PPU/SPU contexts, type system, syscall dispatch, DMA |
| `docs/SYSCALLS.md` | All LV2 kernel syscall implementations |
| `docs/NID_SYSTEM.md` | NID computation, module registration framework |
| `docs/CUSTOM_MODULES.md` | Step-by-step: writing a new HLE module |
| `docs/GAME_PORTING_GUIDE.md` | Full 12-phase porting walkthrough with flOw case study |
| `docs/MODULE_STATUS.md` | Which of the 97 modules are complete/partial |
| `docs/RSX_GRAPHICS.md` | RSX command processor, D3D12/Vulkan backends, shader strategy |
| `docs/TOOLS.md` | Python pipeline tools reference |
| `docs/BUILDING.md` | Build system, compiler notes, link dependencies, troubleshooting |
| `docs/FAQ.md` | Common build/runtime failures and fixes |
