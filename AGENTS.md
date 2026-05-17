# ps3recomp — Agent Notes

## What this is
PS3 static recompilation SDK. Translates PowerPC/SPU binaries into C/C++ that compiles to native x86-64 executables. No emulation at runtime.

**Two layers:** This repo is the SDK/runtime. Game port projects live in subdirectories (e.g., `dbz-budokai-hd/`), each with their own `CLAUDE.md`.

## Directory map
| Path | Purpose |
|---|---|
| `tools/` | Python recompiler pipeline (ELF parse, disasm, lift, NID resolution) |
| `runtime/` | C/C++ runtime core — VM, PPU/SPU contexts, LV2 syscalls |
| `libs/` | HLE module implementations (97 modules) — **add new `.c` files here** |
| `include/ps3emu/` | Public API headers for downstream game projects |
| `templates/project/` | Starter template for new game ports |
| `docs/` | 15 docs, authoritative reference |
| `build/` | CMake output (gitignored) |
| `recompiled/` | Generated lifter output (gitignored) |

## Developer commands
```bash
# Python tool dependencies (run once)
pip install -r requirements.txt

# Build the runtime library
cmake -B build -G Ninja
cmake --build build

# Verify output
ls build/ps3recomp_runtime.lib      # Windows (MSVC)
ls build/libps3recomp_runtime.a     # Linux/macOS/MinGW

# Build a game project (e.g., dbz-budokai-hd)
cd dbz-budokai-hd
build_run.bat        # builds via Ninja, echoes BUILD_EXIT=
run.bat              # runs build\dbz-budokai-hd.exe ..\game\EBOOT.elf
```

On Windows: launch from a **VS 2022 Developer Command Prompt** for the correct MSVC environment.

## Adding a new HLE module
1. Create `libs/<category>/cellFoo.h` and `libs/<category>/cellFoo.c`
2. CMake uses `GLOB_RECURSE` — the new `.c` file is picked up automatically on next configure (no CMakeLists.txt edit needed)
3. Register the module's NIDs in the game project's module registration file using `DECLARE_PS3_MODULE` / `REGISTER_FUNC` / `REGISTER_FNID` macros
4. Modules self-register at static init time via C++ constructors (max 128 modules in `g_ps3_module_registry`)

## Code conventions
- **C17** for `runtime/` and `libs/`; **C++20** for game projects and templates
- snake_case for functions/vars; HLE names must match PS3 SDK (e.g. `cellAudioInit`)
- `SCREAMING_CASE` for constants/macros
- `static` module-local state prefixed with `s_`
- 4-space indent, LF endings, `/* C-style */` comments
- `extern "C"` blocks in headers for C++ compatibility
- Every lifted function signature: `void func_XXXXXXXX(ppu_context* ctx)`

## Compiler quirks
- **MSVC**: needs `/experimental:c11atomics` for `<stdatomic.h>`, `/Zc:__cplusplus`, suppressed warnings `/wd4100 /wd4244 /wd4267 /wd4996`, `_CRT_SECURE_NO_WARNINGS`
- **GCC/Clang**: needs `-fno-strict-aliasing` (required for `be_t<T>` endian conversion through unions)
- `be_t<T>` is a C++ template — C files must use plain types (`u32`, `u64`, etc.) with explicit byte-swapping via `endian.h`
- Vendored deps: `stb_image.h` v2.30, `stb_truetype.h` v1.26 (public domain, in-tree)

## Link libraries (Windows game projects)
`ws2_32 xinput ole32 bcrypt d3d12 dxgi d3dcompiler uuid`

## Recompilation pipeline order
```
decrypt SELF → elf_parser.py → find_functions.py → ppu_disasm.py → ppu_lifter.py → post_lift.py → cmake build
```
- Lifter output can be **100MB+** (100K+ functions) — never commit
- `post_lift.py` applies split-function fallthrough fix and renames `.c` → `.cpp` (if the game project has it)

## Never commit
- `build/`, `recompiled/` (gitignored)
- Game binaries: `*.elf`, `*.self`, `*.pkg`, `*.sprx`, `*.sfo` (gitignored)
- Decryption keys: `*.keys`, `*.rap` (gitignored)
- Generated `.cpp`/`.c` from the lifter

## Critical architecture rules

### Memory access — NEVER raw-cast guest pointers
Guest uses 32-bit effective addresses despite 64-bit PPU. Runtime reserves a 4 GB host region (`VirtualAlloc`/`mmap`) and translates: `host_ptr = vm_base + (uint32_t)guest_addr`. **Always use `vm_read*/vm_write*` helpers** — they handle big-endian byte-swap. Raw pointer derefs through guest addresses will silently produce wrong data.

Guest address map:
| Range | Region |
|---|---|
| `0x00010000–0x0FFFFFFF` | ELF text + data |
| `0x10000000–0x1FFFFFFF` | PRX module space |
| `0x20000000–0x2FFFFFFF` | Stack / TLS |
| `0x30000000–0x3FFFFFFF` | Heap (user malloc) |
| `0xD0000000–0xDFFFFFFF` | RSX IO-mapped main memory |

### Trampoline draining — mandatory after calls
Cross-function tail calls set a TLS function pointer (`g_trampoline_fn`) instead of calling directly. **Always emit `DRAIN_TRAMPOLINE(ctx)` after any call** that might set it — failing to do so causes unbounded host stack growth (stack overflow). The lifter emits this after every `bl`/`bctrl` site.

### RSX MMIO endianness
RSX DMA control registers (put/get/ref) are accessed via `lwbrx`/`stwbrx` in recompiled code (host-endian, no byte-swap). **Do NOT pass them through `vm_write32`** — it byte-swaps for big-endian and corrupts the values. Use direct `uint32_t*` access.

### Indirect call dispatch
`bctr`/`bctrl` emit `ps3_indirect_call(ctx)` with `ctx->ctr` = guest target address. Resolution chain: generated address table (`ppu_resolve_addr`) → hand-lifted extras (`ppu_resolve_extra`) → log + stub. Unknown addresses log once and return `gpr[3] = 0`.

## Testing approach
- No automated test suite currently exists (CMake has `PS3RECOMP_BUILD_TESTS=ON` but no tests)
- Verify changes by building the runtime and linking against a game port
- Verification is done by running the game exe and reading stderr diagnostics
- Compare behavior against RPCS3 logs for HLE call sequences

## Key docs to read
- `docs/GAME_PORTING_GUIDE.md` — Full porting walkthrough with flOw case study
- `docs/CUSTOM_MODULES.md` — How to write new HLE modules (NID, calling convention, registration)
- `docs/ARCHITECTURE.md` — Cell processor and pipeline deep-dive
- `docs/MODULE_STATUS.md` — Implementation status of all 97 modules
- `docs/FAQ.md` — Common build errors, runtime crashes, lifter issues
