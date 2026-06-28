# PowerPC64 Big-Endian Port

Goal: Run the Perfect Dark port on `powerpc64` Linux (big-endian).

## Status summary

| Area | File(s) | Status |
|------|---------|--------|
| Byte-swap macros (`PD_BE16/32/64`) | `src/include/platform.h:61-75` | Define `PD_BE*` as no-op on BE, swap on LE. Correct. |
| Preprocess byteswaps | `port/src/preprocess/*.c` | All use `PD_BE*` to convert N64 ROM data. Correct. |
| `packedpad` bitfields | `src/include/types.h:329-338` | **FIXED 2026-06-28:** BE order was `liftnum(4)\|flags(18)\|room(10)`, corrected to `flags(18)\|room(10)\|liftnum(4)`. |
| `struct texture` bitfields | `src/include/types.h:4935-4943` | Correct as-is on BE. Preprocess function in `misc.c:87-103` now guarded by `#ifndef PLATFORM_BIG_ENDIAN`. |
| `filepads.c` struct `padheader` | `port/src/preprocess/filepads.c:25-29` | No endian guard, but only used for `sizeof()`/pointer-casting — not a bug. |
| Crash handler | `port/src/crash.c:52-67,280-288` | **FIXED:** Added `PLATFORM_PPC64` case with `REG_NIP`. |
| `gbi.c` half-swap | `port/src/preprocess/gbi.c:186-200` | Guarded by `HOST_DWORDS_PER_CMD == 1` (32-bit only). Not a bug on 64-bit. |
| SIMD/optimization | `port/src/mixer.c` | Uses SSE4.1 (x86), NEON (ARM), C fallback. C fallback works on PPC64. |
| `system.c` yield | `port/src/system.c:38-49` | Has x86, ARM, empty fallback. Fallback is a no-op, fine. |
| `Gdma` on BE 64-bit | `include/PR/gbi.h:1316-1321` | **FIXED 2026-06-28:** BE 64-bit `Gdma` redefined to `par:32, cmd:8, len:24` so `cmd:8` reads byte 4 (N64 opcode) instead of byte 3 (padding). |
| `Gtexture` on BE 64-bit | `include/PR/gbi.h:1466-1485` | **FIXED 2026-06-28:** Added `unsigned char pad[4]` before `cmd` so opcode reads from byte 4 (N64 w0 byte 0), not byte 0 (padding). |
| `Gvtx` on BE 64-bit | `include/PR/gbi.h:1641-1668` | **FIXED 2026-06-28:** Added `pad1` before `cmd` and `pad2` before `seg` to align bitfield reads to shifted byte positions. |
| `Gtri` on 64-bit | `include/PR/gbi.h:1350-1373` | **FIXED 2026-06-28:** Added padding on 64-bit (both BE and LE) so `Tri tri` reads from N64 w1 (bytes 12-15 BE, bytes 8-11 LE) instead of w0 padding. |
| `Gtri4` on BE 64-bit | `include/PR/gbi.h:1375-1431` | **FIXED 2026-06-28:** Added `pad0[4]` + `pad2[4]` so z fields read bytes 6-7 and xy nibbles read bytes 12-15. |
| `Gline3D` | `include/PR/gbi.h:1487-1491` | NOT FIXED — `Tri line` at offset 4 is broken on 64-bit, but struct is unused in code. Not crash-critical. |
| `GunkC0` on BE 64-bit | `include/PR/gbi.h:1613-1642` | **FIXED 2026-06-28:** Added `unsigned char pad[4]` before bitfields so `subcmd` reads from byte 6 (N64 w0) instead of byte 0 (padding). |
| `gdl[i].dma.cmd` sign extension | `src/game/bg.c:3460,3462,3475,3477` | **FIXED 2026-06-28:** Added `(u8)` cast to 4 loop comparisons. `unsigned int cmd:8` on PPC64 BE's bitfield layout sign-extends when MSB set (e.g. G_ENDDL=0xB8 → 0xFFFFFFB8 mismatches 0xB8), causing infinite loops past G_ENDDL. |
| `*(s32*)&ptr` on pointer fields (BE 64-bit) | `src/game/setup.c:1161,1484,1490,1700` | **FIXED 2026-06-29:** `lift->doors[i]` and `door->sibling` are 64-bit pointers containing a 32-bit ROM offset. `*(s32*)&ptr` reads the HIGH 4 bytes (zero on BE). Replaced with `(s32)(uintptr_t)ptr` which truncates the full 64-bit value, working on all endiannesses. This caused lift door and door sibling pointers to resolve to the lift itself instead of the actual door, leading to dangling-pointer crashes. |
| IA16 palette byte order on BE | `port/fast3d/gfx_pc.cpp:795-814,1864` | **FIXED 2026-06-29:** `gfx_dp_load_tlut` applies `PD_BE16` to host-endian palette arrays, converting them consistently: on LE the byteswap moves I to low byte, on BE it's a no-op (I stays in MSB). Added `#ifdef PLATFORM_BIG_ENDIAN` conditional in IA16 `palette_to_rgba32` — reads I from MSB on BE, from low byte on LE. RGBA16 branch unchanged (bit positions are endian-independent). Text now renders correctly on BE, colors unchanged on x86_64. |
| `*(s32*)&ptr` on pointer `skel` (BE 64-bit) | `src/game/body.c:264,674` | **FIXED 2026-06-29:** `headmodeldef->skel` is `struct skeleton *` — `*(s32*)&` reads high 4 bytes on BE (zero for small skeleton IDs). Changed to `(s16)(uintptr_t)headmodeldef->skel` which value-truncates the full 64-bit pointer. Head skeleton comparison (SKEL_HEAD=0x0d) was always failing on BE, preventing sunglasses/hudpiece visibility toggle. |
| `loadmemremaining` type mismatch (BE 64-bit) | `src/include/types.h:2350-2360` | **FIXED 2026-06-29:** `loadmemremaining` is `uintptr_t*` but pointed to `handmemloadremaining` (s32, 4 bytes) or `memloadremaining` (u32, 4 bytes). On BE 64-bit, `*loadmemremaining` reads 8 bytes — correct value lands in high 32 bits, low 32 bits are garbage from adjacent field. When cast to `s32` for `texInitPool`, low 32 bits used → `len=0` or garbage → `rightpos` points to wrong address → SIGSEGV in `texLoad` at `tex->texturenum = g_TexNumToLoad`. Changed both fields to `uintptr_t` under `#ifndef PLATFORM_N64` so type matches the accessor. |

## Still needed

### 1. Cross-compile and test
After all fixes above, user tests on ppc64 if game boot + gameplay, if not returns the issues on local x86_64.

## Completed build system fixes

### PowerPC64 arch detection (`platform.h`)
Already present at `src/include/platform.h:34-38`:
```c
#elif defined(__powerpc64__) || defined(__ppc64__) || defined(_ARCH_PPC64)
    #define PLATFORM_PPC64 1
    #define PLATFORM_64BIT 1
```
No changes needed.

### `TARGET_IS_BIG_ENDIAN` CMake dead code
Removed from `CMakeLists.txt` — was hardcoded to `FALSE` with a `# TODO` and completely unused (endianness detected at compile-time in `platform.h`).

## Preprocess memory layout (BE 64-bit)

After preprocessing, each Gfx entry in host memory is 16 bytes:

```
| byte 0 | byte 1 | byte 2 | byte 3 | byte 4 | byte 5 | byte 6 | byte 7 |
| 0x00   | 0x00   | 0x00   | 0x00   | N64 w0 byte 0 (opcode) | ...N64 w0 bytes 1-3 |

| byte 8 | byte 9 | byte 10| byte 11| byte 12| byte 13| byte 14| byte 15|
| 0x00   | 0x00   | 0x00   | 0x00   | N64 w1 byte 0 | ...N64 w1 bytes 1-3 |
```

Key observation: N64 w0 bytes live at host bytes 4-7, N64 w1 bytes at host bytes 12-15.
This means **every Gfx union struct** that accesses N64-specific fields must account for this shifted layout on BE 64-bit.

## GFX_W0_BYTE / GFX_W1_BYTE macros (already correct)
- BE: `GFX_W0_BYTE(i) = 4 + i`, `GFX_W1_BYTE(i) = 12 + i`
- LE 64-bit: `GFX_W0_BYTE(i) = 3 - i`, `GFX_W1_BYTE(i) = 11 - i`
- LE 32-bit: `GFX_W0_BYTE(i) = 3 - i`, `GFX_W1_BYTE(i) = 7 - i`

## Known crash (RESOLVED): SIGSEGV in `bgPopulateVtxBatchType`

The crash read `batchvertices[j].x` where `batchvertices = vertices + UNSEGADDR(gdl[i].words.w1)`.

**Root cause:** `unsigned int cmd:8` in `Gdma` bitfield. On PPC64 BE, GCC places the 8-bit field at bit offset 32 of the 64-bit container and **sign-extends** it when reading. G_ENDDL (=0xB8, MSB=1) reads as 0xFFFFFFB8, which fails `== G_ENDDL` (0xB8). The loop never breaks, reads past the GDL, and crashes.

**Fix:** Added `(u8)` cast before each `gdl[i].dma.cmd` comparison (`bg.c:3359,3460,3462,3475,3477`). The `(u8)` forces an unsigned 8-bit read, avoiding sign extension. Also added `u8 cmd = (u8)gdl[i].dma.cmd;` pattern in the main dispatch loop at `bg.c:3359`.

## Build system notes

- `cmake/TargetArch.cmake:62-69` already detects `ppc64` via `__powerpc64__` / `__ppc64__`.
- `CMakeLists.txt:51` matches `ppc64` with `TARGET_ARCH MATCHES "64"`, so `TARGET_IS_64BIT` is set.
- Need PowerPC64 cross-compiler or native build. On Fedora: `dnf install gcc-powerpc64-linux-gnu`.

## Dependencies (ppc64 Linux)

- SDL2 (with ppc64 support)
- OpenGL (Mesa)
- zlib
