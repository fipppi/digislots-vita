# DigiSlots Vita — Implementation Plan, Part II: UI / UX

> Scope note: this document covers §6–§10. Sections 1–5 (data-source strategy, core-library extraction, save parsing, evaluation engine, plugin injection mechanics) are specified elsewhere. Everything here assumes the core has already been extracted into a freestanding, allocation-free `digislots_core` static library exposing `ds_save_t`, `ds_eval_t`, `ds_check_t`, `ds_species_t` — the UI never parses bytes and the core never draws pixels.
>
> Every technical claim below is tagged **[V]** verified against a file I read, **[L]** likely (strong indirect evidence), or **[S]** speculation. Anything untagged is a design decision, not a fact.

---

## 6. UI stack decision

### 6.1 The decision in one paragraph

**Standalone app → `libvita2d` (already linked, already in CI). Overlay plugin → a hand-written CPU software blitter, written from scratch, sharing zero renderer code with the app.** This is not a preference; it is forced. `sceGxmInitialize` is once-per-process **[V]** (`SCE_GXM_ERROR_ALREADY_INITIALIZED = 0x805B0001`, `/usr/local/vitasdk/arm-vita-eabi/include/psp2/gxm.h:21`), and inside the Adrenaline `ScePspemu` process that context is already owned by `adrenaline_user.suprx`, which calls `vita2d_init()` itself **[V]** (`TheOfficialFloW/Adrenaline user/menu.c:389`, and `isage/Adrenaline user/CMakeLists.txt` links a `libvita2dfbo` fork). A second `vita2d_init()` in an injected plugin would silently ignore the error return **[V]** (`xerpi/libvita2dlib libvita2d/source/vita2d.c` does not check most return values) and then build a competing display queue that fights the host's presentation. The overlay therefore gets no GPU at all — it composes an RGBA panel in ordinary memory and copies it into the framebuffer the host is about to present.

Two renderers, one design system. The bridge is a ~10-function backend interface (§9.1) implemented twice.

### 6.2 Standalone app — chosen stack

| Layer | Library | Status |
|---|---|---|
| 2D renderer | `vita2d` (xerpi/libvita2d, vdpm package `libvita2d`, pkgver=9999 → git HEAD) | Already in `target_link_libraries` **[V]** |
| PNG decode | `libpng` + `zlib` (`png`, `z`) | Already linked **[V]**, currently unused |
| Text | `vita2d_font` (FreeType/FTC path) + one bundled TTF | `vita2d_load_font_file()` / `vita2d_font_draw_text()` **[V]**, `/usr/local/vitasdk/arm-vita-eabi/include/vita2d.h:149-152` |
| Button glyphs | Second `vita2d_font` handle over a PromptFont subset | UTF-8 decoded in the draw loop **[V]** (`vita2d_font.c`: `i += utf8_to_ucs2(&text[i], &character)`) |
| Input | `SceCtrl_stub` + `SceTouch_stub` | Both already linked **[V]** |
| Gradients | `vita2d_draw_array(SCE_GXM_PRIMITIVE_TRIANGLE_STRIP, vita2d_color_vertex[4], 4)` | **[V]** `vita2d.h:98`, `SCE_GXM_PRIMITIVE_TRIANGLE_STRIP = 0x0C000000u` (`gxm.h:94`) |
| Rounded panels | 9-slice via `vita2d_draw_texture_tint_part_scale()` | **[V]** `vita2d.h:135` |
| Clipping (reel mask) | `vita2d_set_clip_rectangle()` / `vita2d_enable_clipping()` | **[V]** `vita2d.h:84-87` |
| Glow | `vita2d_set_blend_mode_add(1)` | **[V]** `vita2d.h:88` |
| Pixel-art scaling | `vita2d_texture_set_filters(t, SCE_GXM_TEXTURE_FILTER_POINT, …)` | **[V]** `vita2d.h:118`; `SCE_GXM_TEXTURE_FILTER_POINT = 0x00000000u` (`gxm.h:864`) |

**Net new dependencies: zero.** The entire visual uplift ships against libraries the project already links.

#### Rejected alternatives — standalone

| Stack | Real artifact | Health | Why rejected |
|---|---|---|---|
| **Dear ImGui via `imgui-vita2d`** (cy33hc/imgui-vita2d, vdpm `imgui-vita2d`, ImGui 1.79 WIP) | `/usr/local/vitasdk/arm-vita-eabi/include/imgui_vita2d/imgui_impl_vita2d.h` **[V]** | Stale (last push 2022-05-11, 7★) **[V]** | No transitions, no 9-slice skinning, no motion language. Its polish ceiling is a restyled `ImGuiStyle`. Goal #3 is explicitly "feel like a real PS Vita title"; ImGui reads as a dev tool. **Keep as an optional hidden debug overlay only** — it composes with vita2d, so it costs nothing to add later behind a button chord. |
| **Dear ImGui via `imgui-vita`** (Rinnegatamante/imgui-vita, vdpm `imgui`) | `/usr/local/vitasdk/arm-vita-eabi/include/imgui_impl_vitagl.h` **[V]** | Healthy (pushed 2026-07-27) **[V]** | Same aesthetic objection, **plus** it drags in `vitaGL`, `vitashark`, `SceShaccCgExt`, `mathneon`. Flagship consumer is `Rinnegatamante/VitaDB-Downloader` **[V]** — which looks like a package manager, because it is one. |
| **vitaGL** (Rinnegatamante/vitaGL, 607★, pushed 2026-07-28) **[V]** | `/usr/local/vitasdk/arm-vita-eabi/include/vitaGL.h` **[V]** | Healthiest library in the ecosystem | A GL state machine and a runtime Cg shader compiler for a 2D card/stat/sprite UI. Zero visual benefit over vita2d, four new link deps. |
| **SDL3** (vdpm `sdl3` 3.4.12, `sdl3_ttf` 3.2.2) **[V]** | `vitasdk/packages/sdl3/VITABUILD`, `vitasdk/vdpm/include/install-packages.sh` **[V]** | Current and real in 2026 | Buys portability the project does not want (Vita-only, reads a Vita-only VMC path). Its Vita renderer is a thin layer over the same GXM path vita2d uses; `SDL_ttf` is the same FreeType. Also **absent from the locally installed 2023 SDK snapshot** — only CI's fresh `vdpm install-all` has it, so local dev would break. |
| **libvita2d_ext** (frangarcj, vdpm `libvita2d_ext`) **[V]** | `/usr/local/vitasdk/arm-vita-eabi/include/vita2d_ext.h` **[V]** | Installed, additive | **Not rejected — deferred.** It is a one-line addition on top of vita2d (`vita2d_ext_init(SceGxmContext*, SceGxmShaderPatcher*)`, `vita2d_create_shader`, `vita2d_set_fragment_uniform`). Adopt only if a CRT/scanline/reel-motion-blur shader is wanted after M7. Not a migration. |
| **System PGF/PVF instead of a bundled TTF** | `vita2d.h` carries the literal comment `/* PGF functions are weak imports at the moment, they have to be resolved manually */` **[V]** | — | Also: `vita2d_pgf_draw_text` takes a `float scale`, not a pixel size — no control over weight or tracking, and no button glyphs. See §6.4. |

### 6.3 Overlay plugin — chosen stack

**A user-mode `.suprx` injected into `ScePspemu` via a `*PSPEMUCFW` section, rendering with a hand-written software blitter.**

| Concern | Decision | Grounding |
|---|---|---|
| Renderer | Hand-written CPU blitter into a private RGBA8888 scratch buffer, then per-scanline copy into the presented framebuffer | Universal technique across every shipping overlay plugin **[V]** |
| Framebuffer format | `SCE_DISPLAY_PIXELFORMAT_A8B8G8R8 == 0x00000000`, 32bpp, **pitch is in PIXELS not bytes** | **[V]** `/usr/local/vitasdk/arm-vita-eabi/include/psp2common/display.h:33-64`, comment literally reads `pitch pixels` |
| **Byte layout is identical to vita2d's `RGBA8()`** | one palette header serves both targets, no conversion | **[V]** `RGBA8(r,g,b,a) = (a<<24)\|(b<<16)\|(g<<8)\|r` (`vita2d.h:15`) → byte0=R,1=G,2=B,3=A, which is A8B8G8R8 little-endian |
| Display seam | `taiHookFunctionImport(&ref, "ScePspemu", 0x4FAACD11 /*SceDisplayUser*/, 0x8C36B628 /*sceDisplaySetFrameBufForCompat*/, hook)` | **[V]** exact hook Adrenaline installs at `user/main.c:571`; NIDs confirmed in `vita-headers db/360/SceDriverUser.yml:140-152` |
| Input read | `sceCtrlPeekBufferPositive` (non-destructive) from the plugin's own thread | **[V]** Adrenaline `user/utils.c:95 readPad()` does exactly this inside pspemu, and uses it in `pops_mode` (`menu.c:528-545`) |
| Input **swallow** | `taiHookFunctionImport(&ref, "ScePspemu", 0x0F35909D /*SceCompat*/, 0x2306FFED /*sceCompatWriteSharedCtrl*/, hook)`, zero `pad_data->Buttons/Lx/Ly/Rx/Ry` | **[V]** Adrenaline's own menu suppression, `user/main.c:403-424` |
| Text | Baked 8-bpp coverage glyph atlases as `const` C arrays, generated at build time from the *same* TTF the app uses | No FreeType, no GXM; guarantees typographic identity across targets |
| Sprites | Single baked raw RGBA blob (`64 × 16 × 16 × 4 = 65,536 B`) embedded via `ld -r -b binary` | **[V]** pattern demonstrated by Adrenaline `user/CMakeLists.txt` `ADD_RESOURCES` |
| PNG decode | **None.** libpng is never linked into the plugin | Decode happens on the build host |

#### Rejected alternatives — overlay

| Approach | Real artifact | Verdict |
|---|---|---|
| **`vita2d` / `vitaGL` / SDL inside the plugin** | `sceGxmInitialize` once-per-process **[V]**; `adrenaline_user` already holds the context **[V]** | **IMPOSSIBLE.** Not hard — impossible. A second `vita2d_init()` builds a competing display queue. |
| **Borrow the host's GXM context** by hooking `sceGxmCreateRenderTarget`/`sceGxmBeginScene` | `SonicMastr/Sonic-Overlay SO_user_shell/src/patches.c:96-130, 250-300` **[V]** (MIT) | Technically real — it does create its own shader programs in a foreign process — but it hooks **ScePaf**, i.e. SceShell only (`hooks.c:21`), it is WIP with last commit 2021-05-26, and it would mean racing Adrenaline's own draw thread. Rejected. |
| **`ScePaf` + RCO inside SceShell** | `GrapheneCt/PSVshellPlus PSVshellPlus_Shell/hud.cpp` **[V]**, installs under `*main` + `*KERNEL` | The genuine aspirational ceiling — real GPU-accelerated, animated, Quick-Menu-integrated first-party UI. Costs: reverse-engineered ScePaf, a hand-authored `.rco` with hashed page/template IDs, and **MSVC `.vcxproj` against DolceSDK, not VitaSDK/CMake** **[V]**. Also unclear it would even appear over Adrenaline, which locks `SCE_SHELL_UTIL_LOCK_TYPE_PS_BTN_2` **[V]** (`user/main.c:373`). Rejected. |
| **`SceSharedFb`** (the "system mode application" compositor) | `/usr/local/vitasdk/arm-vita-eabi/include/psp2/sharedfb.h` exists **[V]**; used by upstream `xerpi/libvita2d` `system_app_mode` and `Rinnegatamante/vitaGL source/gxm.c` **[V]** | Real API, **but no shipping precedent of a taiHEN plugin using it to overlay a game** — code search returns only SDK and library sources. Requires being a system-budget app (`sceKernelIsGameBudget() != 1`), `sceGxmInitializeInternal` from `SceGxmInternalForVsh`, and PVRSRV driver calls; `GrapheneCt/BG-App-PSV` README states DolceSDK is required. **[S]** Rejected — but this is the *only* unexplored lead if the blitter route dies. |
| **Kernel `.skprx` hooking `ksceDisplaySetFrameBufInternal`** (0x9FED47AC / 0x16466675) | `Electry/PSVshell src/main.c:307`, `MERLev/reVita src/main.c:735` **[V]** | **Demoted to fallback.** Two decisive reasons for user-mode instead: (a) the *only* place `ScePspemuConvertAddress` is reachable is inside the pspemu process (it's resolved at `text_addr + 0x6364 + 1` **[V]**), which is the door to live PS1 RAM; (b) `PSVshell` explicitly refuses to draw under pspemu — `if (g_is_in_pspemu) goto DISPLAY_HOOK_RET;` **[V]** `src/main.c:68` — and its README lists *"Does not crash Adrenaline"* as a **feature**. Keep the kernel route documented as Plan B only. |
| **PSP-side `.prx` under the `pops` runlevel** | Adrenaline v7 `cef/systemctrl/init_patch.c:207-208` **[V]**; v8 `cef/core/systemctrl/src/plugin.c:295` (`pops`/`ps1`/`psx` tokens) **[V]**; `cwcheatpops.prx` precedent **[L]** | Real, but: a second MIPS toolchain (pspsdk), a 320×240 RGBA5551 or 480×272 canvas, and — decisively — the only way to make the PSP framebuffer visible is to **suspend the `popsmain` thread**, which pauses the game **[V]** (`isage cef/core/pentazemin/src/modpatches.c:50-62` → `ADRENALINE_VITA_CMD_PAUSE_POPS`). That is a modal takeover screen, not an overlay. Rejected for the HUD; noted as the fallback if you need a guaranteed-visible full panel. |
| **Copying `PSVshell/src/gui.c` as the blitter** | `Electry/PSVshell` has **no LICENSE file**; `gh api repos/Electry/PSVshell/license` → 404 **[V]** | **LEGAL BLOCKER.** Default is all-rights-reserved. `reVita`, `VitaGrafix`, `PSV-VSH-Menu` are all GPL-3.0 **[V]** — lifting their renderers makes `digislots.suprx` GPLv3. `digislots-vita` currently has no LICENSE at all. **Decision: write the blitter from scratch (~250 LOC against a documented struct), and add an explicit LICENSE to this repo.** Read those repos for architecture, copy no code. |

#### Non-negotiable constraints the overlay design must absorb

1. **[V] The overlay is invisible when Adrenaline Graphics Filtering = "Original".** `isage/Adrenaline user/menu.c:657` (identical logic to FloW `menu.c:549`):
   ```c
   if (sceCommonDialogIsRunning() || (config.graphics_filtering == 0 && menu_open == 0 && draw_native == 0)) {
       sceDisplayWaitVblankStart(); continue;
   }
   ```
   The whole `vita2d_start_drawing()/swap_buffers()` block is skipped, and `user/pops.c:172-186` only substitutes a real A8B8G8R8 buffer `if (config.graphics_filtering != 0)` — in Original mode a NULL `pParam` passes straight through. Independently documented by `MERLev/reVita README.md:74` and by the maintained fork `JSilva6/reVita-extended` (pushed 2026-07-22) **[V]**.
   **UI consequence:** the *standalone app* owns overlay onboarding. It ships an "Overlay Setup" screen that states the requirement, and the plugin writes a heartbeat to `ur0:data/digislots/overlay.log` that the app reads back to report "overlay saw N frames in the last session / overlay saw 0 frames — check Graphics Filtering". The overlay can never report its own invisibility, so the app must.
2. **[V] Combo collisions.** `L+SELECT` is Adrenaline's POPS fast-forward (`menu.c:640-653`), PS-double-tap is Adrenaline's menu (`utils.c:205-230`). Both are off-limits.
3. **[V] Adrenaline reads the pad through two paths** — `sceCtrlPeekBufferPositive` *and* its own `kuCtrlPeekBufferPositive` syscall (`kernel/main.c:169-189`). A kernel hook set on the user `SceCtrl` library cannot gate the `ku` path. Another reason user-mode + `sceCompatWriteSharedCtrl` is the right seam.
4. **[S] The rear touchpad is probably free** (`SCE_TOUCH_PORT_BACK = 1`, `psp2/touch.h` **[V]**; PSX hardware has no touch, so POPS cannot consume it). Whether *Adrenaline* claims it is unverified. Design a three-tier input ladder (§8.7) so the design survives if it doesn't.

### 6.4 Font strategy

| | App | Overlay |
|---|---|---|
| Source | `MPLUSRounded1c-Regular.ttf` (SIL OFL) — closest free analogue to `SCE_PVF_FAMILY_ROUNDED` **[V]** (`psp2/pvf.h`), which is *the* characteristic Vita look | same TTF, rasterized on the host |
| Shipped form | `pyftsubset` → **49,880 B** (Latin, Latin-1, quotes, bullet, ellipsis, arrows, ✓ ✗ ▲ ▼ ● ×) | six `const uint8_t[]` coverage atlases in `.rodata` |
| Button glyphs | `promptfont.ttf` (Shinmera/Yukari Hafner, SIL OFL, based on Xolonium) subset to 18 codepoints → **3,372 B** | baked into the same atlases |
| Handles | **Two separate `vita2d_font*`.** They cannot be merged — `fontTools.merge.Merger` raises `AssertionError: Expected all items to be equal: [0, NotImplemented]` because M PLUS carries GDEF/GPOS/GSUB/vhea/vmtx and PromptFont does not **[V]**. Two handles is also cheaper: each owns its own atlas. | n/a — one baked atlas per size |
| Fallback if 50 KB is too much | `TitilliumWeb-Regular.ttf` subset = **26,992 B**; colder, reads more "PS4" than "Vita" | same |

**Verified PromptFont codepoints** (from `glyphs.json`, release v1.10) — the Vita has no L2/R2, no touchpad, no Options button, so only these are legitimate:

| Glyph | CP | Glyph | CP | Glyph | CP |
|---|---|---|---|---|---|
| Cross ✕ | `U+21E3` | Circle ○ | `U+21E2` | Square □ | `U+21E0` |
| Triangle △ | `U+21E1` | L1 | `U+21B0` | R1 | `U+21B1` |
| Select | `U+21F7` | Start | `U+21F8` | D-pad | `U+21CE` |
| D-pad ↔ | `U+21A2` | D-pad ↕ | `U+21A3` | Analog stick | `U+21CD` |
| ← ↑ → ↓ | `U+219E` `U+219F` `U+21A0` `U+21A1` | L3 | `U+21BA` | R3 | `U+21BB` |

Every PromptFont advance is exactly 1.0 em (24.0 px at size 24) **[V]**, so hint-group widths are `24 + 8 + ui_text_w(UI_BODY, label)` with no measurement call for the glyph.

> **[V] HARD CAP — the glyph atlas.** `libvita2d/source/vita2d_font.c` defines `ATLAS_DEFAULT_W 512` / `ATLAS_DEFAULT_H 512`, format `SCE_GXM_TEXTURE_FORMAT_U8_R111`, and **on overflow `atlas_add_glyph()` returns 0 and the glyph is silently not drawn** — no error, no resize. Every distinct `size` argument allocates a distinct glyph set.
>
> Measured occupancy for the six-step scale in §7 using real M PLUS bounding boxes: 14px 7,811 px² · 18px 11,949 · 22px 16,897 · 28px 25,706 · 40px 25,042 (A–Z 0–9 !?- only) · 56px 47,090 (same restricted set) = **134,495 px² = 51.3 % of 262,144; ≈68 % after shelf-packer waste.** It fits with room for exactly one more size.
>
> **Two rules that follow, and they are absolute:**
> 1. Do not add a seventh size. "Large text" accessibility is implemented as a **swap** of the six loaded sizes, never an addition.
> 2. **Never animate the `size` argument.** Animating scale by ramping `size` allocates a new atlas row every frame and exhausts the atlas in seconds. Animate scale by drawing into a render target (`vita2d_create_empty_texture_rendertarget` **[V]** `vita2d.h:105`), or animate position/alpha instead.

### 6.5 Asset pipeline

A single host-side generator, `tools/mkassets.py`, run by a CMake custom command. It is the only thing that touches raw art, and it emits both targets from one source of truth.

```
src/resources/*.png (63 sprites)  ─┐
assets/MPLUSRounded1c-Regular.ttf ─┼─► tools/mkassets.py ─┬─► build/gen/atlas.png        (128x128, app)
assets/promptfont.ttf             ─┤                      ├─► build/gen/mplus-sub.ttf    (49,880 B, app)
assets/panel9.png shadow9.png     ─┤                      ├─► build/gen/promptfont-sub.ttf(3,372 B, app)
assets/glow.png lcd.png           ─┘                      ├─► build/gen/atlas_rgba.bin   (65,536 B, plugin)
                                                          ├─► build/gen/ds_font_14..56.h (plugin, 8bpp)
                                                          ├─► build/gen/ds_panel9.h      (plugin, alpha)
                                                          └─► build/gen/ds_sprite_ids.h  (SHARED)
```

**Pipeline stages, in order:**

1. **Delete `src/resources/image.png`.** 63,164 B, 460×460 16-bit RGBA, a stock "plant in a box" placeholder, referenced by nothing **[V]** (`grep` over all `.c/.h/.txt/.xml/.md` returns zero hits). That is 61 % of the resource directory and ~15 % of the shipped VPK.
2. **Normalise the two 32×32 outliers.** `Machinedramon.png` and `Player.png` are *genuinely* 32×32 (content bboxes 31×28 and 30×28, not padded 16×16) **[V]**. Downsample or re-draw at 16×16 so the atlas is uniform. Regardless of outcome, **always derive scale as `TARGET_PX / vita2d_texture_get_width(tex)`** **[V]** (`vita2d.h:110`) — never a hardcoded constant.
3. **Threshold alpha at 50 %.** The 16×16 set carries anti-aliased edges (16 distinct alpha values in `Agumon.png`) while the 32×32 pair is hard 0/255 **[V]**. Under `POINT` filtering at 10×, each semi-transparent edge pixel becomes a 10×10 haze block. Hard alpha is both crisper and smaller.
4. **Pack into one 128×128 atlas**, 8×8 grid of 16×16 cells. 63 sprites ≤ 64 cells — it fits exactly. Replaces 63 `vita2d_load_PNG_file()` calls and 63 libpng decodes at startup with one. Draw with `vita2d_draw_texture_part_scale(tex, x, y, (id%8)*16, (id/8)*16, 16, 16, s, s)` **[V]** (`vita2d.h:126`).
5. **Emit `ds_sprite_ids.h`** — the stable `species_id → atlas cell` map, `#include`d by app, plugin and core alike. This closes a real gap: `struct Digimon` **[V]** (`src/models/digimon.h`) has no sprite field today; add `int spriteIndex`.
6. **Strip PNG metadata** — each sprite carries `bKGD`+`pHYs`+`tIME`+`iTXt` ≈ 99 B of junk × 63 = 6,237 B **[V]**. `oxipng -o max --strip all`.
7. **Subset the fonts** (`pyftsubset`), and **rasterize** the six sizes to 8-bpp coverage arrays for the plugin.
8. **Fix `CMakeLists.txt`.** The `file(COPY ${CMAKE_SOURCE_DIR}/resources/ ...)` block appears **twice, verbatim** **[V]**, copies the reverse-engineering notes (including the 27,080-line `DMW_New.CT`) into the VPK, and its destination disagrees with the `FILE ${CMAKE_BINARY_DIR}/src/resources resources` directive — which itself only resolves because the project is currently built in-source **[V]**.

**VPK budget:** 423 KB today − 63 (image.png) − 6 (metadata) − ~25 (63 PNGs → 1 atlas) + 53 (fonts) + 5 (9-slices) ≈ **387 KB**. *The entire visual overhaul ships smaller than the current text-mode build.* There is no size argument against doing this properly.

### 6.6 CMakeLists.txt — exact changes

```cmake
# ── 0. UNBREAK. Current file says VERSION 2.8, which is a HARD ERROR on CMake >= 4.0.
cmake_minimum_required(VERSION 3.16)

# ── 1. Core is platform-free and builds on the host too.
file(GLOB DS_CORE_SRC core/src/*.c)
add_library(digislots_core STATIC ${DS_CORE_SRC})
target_include_directories(digislots_core PUBLIC core/include ${CMAKE_BINARY_DIR}/gen)
set_target_properties(digislots_core PROPERTIES C_STANDARD 99 C_STANDARD_REQUIRED ON)
target_compile_options(digislots_core PRIVATE
    -Wall -Wextra -Wconversion -Wshadow -Wformat=2 -Wvla -Werror=return-type)
#   ^ -Wvla and -Wformat=2 would each have caught a live bug in the current tree.

# ── 2. Asset generation (host python3 + fonttools + pillow), one command, both targets.
find_package(Python3 COMPONENTS Interpreter REQUIRED)
set(DS_GEN ${CMAKE_BINARY_DIR}/gen)
add_custom_command(
    OUTPUT ${DS_GEN}/atlas.png ${DS_GEN}/atlas_rgba.bin ${DS_GEN}/ds_sprite_ids.h
           ${DS_GEN}/mplus-sub.ttf ${DS_GEN}/promptfont-sub.ttf
           ${DS_GEN}/ds_font_14.h ${DS_GEN}/ds_font_18.h ${DS_GEN}/ds_font_22.h
           ${DS_GEN}/ds_font_28.h ${DS_GEN}/ds_font_40.h ${DS_GEN}/ds_font_56.h
    COMMAND ${Python3_EXECUTABLE} ${CMAKE_SOURCE_DIR}/tools/mkassets.py
            --sprites ${CMAKE_SOURCE_DIR}/src/resources
            --fonts   ${CMAKE_SOURCE_DIR}/assets
            --out     ${DS_GEN}
    DEPENDS ${CMAKE_SOURCE_DIR}/tools/mkassets.py
    COMMENT "Generating DigiSlots sprite atlas, font subsets and baked glyph atlases")
add_custom_target(ds_assets DEPENDS ${DS_GEN}/atlas.png)

# ── 3. The app. Backend = vita2d.
add_executable(${PROJECT_NAME}
    ui/main.c ui/app.c ui/io_vita.c ui/input_vita.c
    ui/render/ui_backend_vita2d.c ui/render/gfx.c
    ui/render/widgets.c
    ui/screens/scr_status.c ui/screens/scr_roll.c ui/screens/scr_reqs.c
    ui/screens/scr_dex.c    ui/screens/scr_settings.c ui/screens/scr_setup.c)
add_dependencies(${PROJECT_NAME} ds_assets)
target_link_libraries(${PROJECT_NAME}
    digislots_core
    vita2d png z m                       # already present
    SceGxm_stub SceDisplay_stub SceCommonDialog_stub SceAppMgr_stub
    SceCtrl_stub SceTouch_stub
    SceIofilemgr_stub SceLibKernel_stub SceSysmodule_stub ScePower_stub)
#   DROP the ~15 stubs linked today and never called: SceHttp, SceNet, SceNetCtl,
#   SceSsl, SceMtpIfDriver, SceMusicExport, ScePhotoExport, SceUdcd,
#   SceUsbstorVStorDriver, SceVshBridge, ScePromoterUtil, SceShellSvc,
#   SceAudiodec, SceAudio, SceIme, SceAppUtil, ScePgf, ScePvf.
#   ScePgf/ScePvf go because we bundle a TTF (see 6.4).

# ── 4. The overlay plugin. Separate target, NO vita2d, NO png.
add_executable(digislots_plugin
    plugin/main.c plugin/hooks.c
    plugin/render/ui_backend_blit.c   # written from scratch — see licensing note in 6.3
    plugin/render/blit.c plugin/render/font_baked.c
    plugin/screens/ovl_strip.c plugin/screens/ovl_panel.c
    ui/render/widgets.c)              # SHARED widget layer, backend-agnostic
add_dependencies(digislots_plugin ds_assets)
target_compile_definitions(digislots_plugin PRIVATE DS_TARGET_PLUGIN=1)
target_link_libraries(digislots_plugin
    digislots_core
    taihen_stub
    SceDisplay_stub SceCtrl_stub SceIofilemgr_stub
    SceLibKernel_stub SceKernelModulemgr_stub SceSysmem_stub SceThreadmgr_stub
    SceLibc_stub gcc)
vita_create_self(digislots.suprx digislots_plugin CONFIG plugin/digislots.yml UNSAFE)

# ── 5. Package. Explicit FILE entries; the two duplicated file(COPY) blocks are DELETED.
vita_create_self(${PROJECT_NAME}.self ${PROJECT_NAME} UNSAFE)
vita_create_vpk(${PROJECT_NAME}.vpk ${VITA_TITLEID} ${PROJECT_NAME}.self
  VERSION ${VITA_VERSION} NAME ${VITA_APP_NAME}
  FILE sce_sys/icon0.png                          sce_sys/icon0.png
  FILE sce_sys/livearea/contents/bg.png           sce_sys/livearea/contents/bg.png
  FILE sce_sys/livearea/contents/startup.png      sce_sys/livearea/contents/startup.png
  FILE sce_sys/livearea/contents/template.xml     sce_sys/livearea/contents/template.xml
  FILE ${DS_GEN}/atlas.png                        assets/atlas.png
  FILE ${DS_GEN}/mplus-sub.ttf                    assets/ui.ttf
  FILE ${DS_GEN}/promptfont-sub.ttf               assets/glyphs.ttf
  FILE assets/panel9.png                          assets/panel9.png
  FILE assets/shadow9.png                         assets/shadow9.png
  FILE assets/glow.png                            assets/glow.png
  FILE assets/lcd.png                             assets/lcd.png
  FILE assets/OFL-MPLUS.txt                       assets/OFL-MPLUS.txt
  FILE assets/OFL-PromptFont.txt                  assets/OFL-PromptFont.txt
  FILE ${CMAKE_BINARY_DIR}/digislots.suprx        plugin/digislots.suprx)

# ── 6. Forbid in-source builds (the current VPK only works by accident of in-source).
if(CMAKE_BINARY_DIR STREQUAL CMAKE_SOURCE_DIR)
  message(FATAL_ERROR "In-source builds are not supported. Use: cmake -B build .")
endif()
```

Also: `git rm --cached CMakeCache.txt Makefile cmake_install.cmake digislots_vita digislots_vita.vpk hello.txt` and extend `.gitignore` **[V]** — all of these are currently tracked, and `CMakeCache.txt` hardcodes the original author's absolute paths.

---

## 7. Visual identity — "DIGIVICE DUSK"

### 7.1 Concept

Vita system chrome (deep navy field, soft vertical gradients, rounded translucent panels, one cyan-leaning blue accent) as the frame; a pale back-lit **Digivice LCD plate** as the single warm/light surface, used wherever a sprite is displayed.

The LCD plate is not decoration — it is a measured fix. The darkest sprite pixel (`#1A0800`) against the app background (`#0A0F1A`) measures **1.02:1** — the silhouettes are literally invisible. Against a `#C9D8EE` plate it measures **13.48:1**. 48–63 % of each sprite's opaque pixels are near-black **[V]** (measured by decoding IDAT: Agumon 48 %, Greymon 57 %, Machinedramon 63 %, Player 54 %). Every sprite must sit on a plate or wear a synthesized rim (§9.4).

### 7.2 Palette

All contrast ratios below are WCAG relative-luminance, computed against `--surface-1 #162032`.

**Base / structure**

| Token | Hex | Role |
|---|---|---|
| `bg-void` | `#05070D` | scrim base, overlay backdrop |
| `bg-base` | `#0A0F1A` | app background, bottom of gradient |
| `bg-raise` | `#111827` | top of the app background gradient |
| `surface-1` | `#162032` | panel fill (1.17:1 vs bg — a deliberate whisper) |
| `surface-2` | `#1E2B42` | row / raised panel / reel body |
| `surface-3` | `#2A3A57` | bar track, input track |
| `line` | `#33456B` | 2 px visible border (1.71:1 vs surface-1) |
| `line-soft` | `#24314C` | 1 px divider |

**Text**

| Token | Hex | Ratio | |
|---|---|---|---|
| `text-hi` | `#F2F6FF` | **15.08:1** | AAA |
| `text-mid` | `#A8B7D4` | **8.07:1** | AAA |
| `text-lo` | `#7D8DAD` | **4.88:1** | AA |

> `#6E7E9E` was the original candidate for `text-lo` and measures **4.00:1 — it FAILS**. Use `#7D8DAD`. Also: `text-lo` is licensed for `surface-1` and `bg-base` **only** — on `surface-2` it drops to 3.48:1.

**Accent**

| Token | Hex | Ratio |
|---|---|---|
| `accent` | `#2E9BFF` | 5.63:1 |
| `accent-hi` | `#7CC4FF` | 8.70:1 (focus ring / hover / glow) |
| `accent-dim` | `#14416E` | selected-row fill, payline halo base |

**Semantic state**

| Token | Hex | Ratio |
|---|---|---|
| `ok` | `#3DD68C` | 8.70:1 |
| `warn` | `#FFC24B` | 10.16:1 |
| `bad` | `#FF5C6C` | 5.44:1 |

**Stat categories** — 1:1 with the web app's Bootstrap row classes **[V]** (read from `digislots.fippi.io/static/js/main.6849a8c2.chunk.js`), so the two products share one information architecture.

| Token | Hex | Ratio | Was | Stats |
|---|---|---|---|---|
| `cat-ident` | `#94A3B8` | 6.36:1 | `table-secondary` | Name, Type, Level |
| `cat-battle` | `#38BDF8` | 7.62:1 | `table-info` | HP, MP, Offense, Defense, Speed, Brains |
| `cat-care` | `#818CF8` | 5.47:1 | `table-primary` | Care Mistakes |
| `cat-weight` | `#FB7185` | 6.06:1 | `table-danger` | Weight |
| `cat-raise` | `#FBBF24` | 9.78:1 | `table-warning` | Discipline, Happiness, Battles, Techniques, Digimon Bonus |

**Digivice LCD plate**

| Token | Hex |
|---|---|
| `lcd-face` | `#C9D8EE` (blue-white default) |
| `lcd-face-alt` | `#CBDCC0` (classic pale-green Digivice, user-selectable) |
| `lcd-edge` | `#8FA8CF` (1 px inner bevel, top-left) |
| `lcd-scan` | `#05070D` @ 10 %, 1 px horizontal every 3 px |

**Colour-blind mode** (Okabe-Ito derived, Settings toggle): `cvd-ok #56B4E9` · `cvd-bad #E69F00` · `cvd-warn #F0E442`.

> **MEASURED CAVEAT — read this before designing any state indicator.** `ok` vs `bad` have a greyscale delta of only **17** (need ≥25). The Okabe-Ito pair `cvd-ok` vs `cvd-bad` has a delta of **4**. *Neither pair survives achromatopsia or a greyscale photo.* Redundant non-colour encoding is therefore **mandatory, not optional**: every requirement row carries four simultaneous signals — category rail hue, a **shape-distinct** glyph (`✓` U+2713 / `▲` U+25B2 / `✕` U+2717), a bar fill length, and a signed numeric delta.

### 7.3 Typography scale

Six steps. The count is dictated by the 512×512 atlas budget (§6.4), not by taste.

| Token | px | Line | Tracking | Case | Use |
|---|---|---|---|---|---|
| `caption` | 14 | 20 | +0.06 em | CAPS | category labels, units, footnotes |
| `body` | 18 | 26 | 0 | — | stat values, row labels, settings descriptions |
| `label` | 22 | 30 | +0.02 em | — | digimon names in lists, tab labels, button text |
| `title` | 28 | 36 | 0 | — | screen titles, hero digimon name |
| `display` | 40 | 48 | +0.10 em | CAPS | wordmark, big numerals, roll result |
| `reel` | 56 | 64 | +0.08 em | CAPS | slot-reel stage/type text |

**Minimum-size justification (not arbitrary).** The Vita's 5″ 960×544 panel is ≈220 PPI → 1 px = 0.1155 mm ≈ **1.32 arcmin** at a 30 cm handheld viewing distance. Comfortable sustained reading needs ≥16 arcmin cap height ≈ 12 px cap ≈ **17–18 px em**. Therefore:

- **18 px is the floor for anything read in sentences.**
- **14 px is the absolute floor**, permitted only for glanceable ALL-CAPS labels and units.
- **In the overlay the floors rise to 20 px body / 16 px caption**, because the type sits over moving, arbitrary-luminance gameplay.
- **Never render 12 px.**
- Because of the 220 PPI density, **1 px rules are hairline-invisible.** Use 2 px for any border that must be seen; reserve 1 px for `line-soft` dividers only.

### 7.4 Spacing grid for 960×544

Base unit **8 px**, half-unit 4 px. Outer margin **32 px** (Vita-safe; also inside PS TV's typical 5 % action-safe). Content width **896 px**.

**12 columns of 60 px with 16 px gutters** — chosen because `(896 − 11×16) / 12 = 60` exactly.
Column *n* origin: `x = 32 + 76n`. Span of *k* columns: `w = 76(k−1) + 60`.

| Cols | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Width | 60 | 136 | 212 | 288 | 364 | 440 | 516 | 592 | 668 | 744 | 820 | 896 |

Last column ends at 928; 928 + 32 = 960. ✓

**Vertical bands — fixed on every standalone screen:**

| Band | y range | Height |
|---|---|---|
| Title bar | 0 – 56 | 56 |
| Tab bar | 56 – 104 | 48 |
| Content | 104 – 488 | 384 |
| Button-hint footer | 488 – 544 | 56 |

All internal vertical rhythm on an 8 px baseline.

**Sprite scale ladder** (integer only, from a 16 px source):

| Scale | px | Used for |
|---|---|---|
| 2× | 32 | list icon, HUD strip |
| 3× | 48 | dex grid cell art, evolution strip |
| 4× | 64 | requirement header, card |
| 6× | 96 | overlay full panel |
| 8× | 128 | dex modal hero |
| 10× | 160 | status portrait, reel centre |

### 7.5 Corner radii

| Token | px | Use |
|---|---|---|
| `r-sm` | 4 | chips, bar caps, tags |
| `r-md` | 8 | rows, buttons, inputs |
| `r-lg` | 12 | panels, cards, reels |
| `r-xl` | 20 | modals, LCD plate outer |
| `r-pill` | h/2 | toggles, progress caps |

vita2d has **no** rounded-rect and **no** gradient primitive **[V]** (`vita2d.h` primitives are only `draw_pixel/line/rectangle/fill_circle/array`). Two implementations, use both:

- **Radii ≤ 8, solid fill** → composite: 1 centre `vita2d_draw_rectangle` + 2 side rects + 4 `vita2d_draw_fill_circle`. Cheap, but the circles are un-antialiased triangle fans and look faceted above ~8 px radius.
- **Radii ≥ 12, and anything needing a clean edge** → **9-slice** a single 48×48 alpha-only PNG via `vita2d_draw_texture_tint_part_scale()` (9 draws). One texture serves every panel at every size and every colour, because you tint at draw time.
- **Gradients** → `vita2d_draw_array(SCE_GXM_PRIMITIVE_TRIANGLE_STRIP, verts, 4)` with `vita2d_color_vertex` **[V]**. Per-vertex colour gives a true interpolated gradient at zero texture cost. This draws the background, the title bar, every stat-bar fill, the reel body, and the reel vignettes.

**Elevation:** `e0` flat (no shadow) · `e1` panels/cards — 96×96 alpha-only 9-slice `shadow9.png`, offset y+2, tinted `#000` @ 45 % · `e2` modals/toasts — same texture, offset y+6, `#000` @ 60 %, plus a full-screen `bg-void` @ 72 % scrim · `e3` emphasis glow — 64×64 radial-alpha `glow.png` under `vita2d_set_blend_mode_add(1)` in `accent-hi`, then `(0)`. **Never stack more than two elevation levels.**

### 7.6 Motion rules

The loop is vsync-locked, so **specify everything in FRAMES at 60 Hz, never in milliseconds.** (Halve every count if you ever run `vita2d_set_vblank_wait(0)` at 30 fps **[V]** `vita2d.h:77`.)

Easing, as literal C:

```c
easeOutCubic (t) → 1.0f - powf(1.0f - t, 3.0f)
easeInQuad   (t) → t * t
easeOutQuart (t) → 1.0f - powf(1.0f - t, 4.0f)
easeInOutCubic(t)→ t < 0.5f ? 4.0f*t*t*t : 1.0f - powf(-2.0f*t + 2.0f, 3.0f) * 0.5f
easeOutBack  (t) → 1.0f + 2.70158f*powf(t-1,3) + 1.70158f*powf(t-1,2)
```

| Event | Frames | Easing |
|---|---|---|
| Focus / selection move | 8 | easeOutCubic |
| Button press in / out | 4 / 6 | scale 1.0 → 0.96 |
| Tab switch | 11 | crossfade + 12 px slide, easeOutCubic |
| Panel enter | 14 | easeOutCubic, opacity 0→1, y +16→0 |
| Panel exit | 9 | easeInQuad |
| Modal enter | 12 | scale 0.94→1.0 easeOutBack + scrim fade |
| Toast in / hold / out | 14 / 144 / 12 | slide + fade |
| Stat bar fill on data change | 20 | easeOutQuart |
| Overlay show / hide | 12 / 8 | easeOutCubic / easeInQuad |
| Reel spin (total) | **142** | see §8.2 |

**Nothing except the reel exceeds 24 frames.** Honour a "Reduce motion" setting that clamps every duration to 4 frames and disables the reel spin entirely (jump straight to the reveal).

### 7.7 The header — `ui/ds_theme.h`

This is the literal file. It is `#include`d by **both** targets: the palette works unchanged in the overlay because `SCE_DISPLAY_PIXELFORMAT_A8B8G8R8` and vita2d's `RGBA8()` have identical byte layout **[V]**.

```c
/* ============================================================================
 *  ui/ds_theme.h — DigiSlots Vita design tokens : "DIGIVICE DUSK"
 *
 *  Single source of truth for colour, type, grid, radii and motion.
 *  Shared verbatim by the vita2d app and the software-blitter overlay plugin.
 *
 *  Colour representation
 *  --------------------
 *  Packed 0xAABBGGRR (byte 0 = R, 1 = G, 2 = B, 3 = A).
 *  This is BOTH vita2d's RGBA8() layout  (vita2d.h:15)  AND the layout of
 *  SCE_DISPLAY_PIXELFORMAT_A8B8G8R8     (psp2common/display.h:34),
 *  so one constant serves the GPU renderer and the CPU blitter with no
 *  conversion whatsoever. Do not "fix" this to 0xAARRGGBB.
 * ========================================================================== */
#ifndef DS_THEME_H
#define DS_THEME_H

#include <stdint.h>

/* ---- colour construction ------------------------------------------------ */
#define DS_RGBA(r,g,b,a)  ((uint32_t)(((uint32_t)(a)<<24) | ((uint32_t)(b)<<16) \
                                    | ((uint32_t)(g)<< 8) | ((uint32_t)(r)    )))
/* build from a #RRGGBB literal plus an 0..255 alpha */
#define DS_HEX(h,a)       DS_RGBA(((h)>>16)&0xFF, ((h)>>8)&0xFF, (h)&0xFF, (a))
/* re-alpha an existing token, e.g. DS_ALPHA(DS_OK, 46) for an 18% wash */
#define DS_ALPHA(c,a)     (((c) & 0x00FFFFFFu) | ((uint32_t)(a) << 24))
#define DS_A(pct)         ((uint8_t)((pct) * 255 / 100))

/* ---- base / structure --------------------------------------------------- */
#define DS_BG_VOID        DS_HEX(0x05070D, 255)  /* scrim base, overlay backdrop  */
#define DS_BG_BASE        DS_HEX(0x0A0F1A, 255)  /* app bg, bottom of gradient    */
#define DS_BG_RAISE       DS_HEX(0x111827, 255)  /* app bg, top of gradient       */
#define DS_SURFACE_1      DS_HEX(0x162032, 255)  /* panel fill                    */
#define DS_SURFACE_2      DS_HEX(0x1E2B42, 255)  /* row / reel body               */
#define DS_SURFACE_3      DS_HEX(0x2A3A57, 255)  /* bar + input track             */
#define DS_LINE           DS_HEX(0x33456B, 255)  /* 2px visible border            */
#define DS_LINE_SOFT      DS_HEX(0x24314C, 255)  /* 1px divider                   */

/* ---- text (contrast measured vs DS_SURFACE_1) --------------------------- */
#define DS_TEXT_HI        DS_HEX(0xF2F6FF, 255)  /* 15.08:1  AAA                  */
#define DS_TEXT_MID       DS_HEX(0xA8B7D4, 255)  /*  8.07:1  AAA                  */
#define DS_TEXT_LO        DS_HEX(0x7D8DAD, 255)  /*  4.88:1  AA                   */
/* NOTE: DS_TEXT_LO is licensed for SURFACE_1 and BG_BASE ONLY.
 *       On SURFACE_2 it measures 3.48:1 and FAILS. Do not use it there.
 *       Do NOT revert to #6E7E9E — that measures 4.00:1 and fails everywhere. */

/* ---- accent ------------------------------------------------------------- */
#define DS_ACCENT         DS_HEX(0x2E9BFF, 255)  /*  5.63:1                       */
#define DS_ACCENT_HI      DS_HEX(0x7CC4FF, 255)  /*  8.70:1  focus / glow         */
#define DS_ACCENT_DIM     DS_HEX(0x14416E, 255)  /*          selected-row fill    */

/* ---- semantic state ----------------------------------------------------- */
#define DS_OK             DS_HEX(0x3DD68C, 255)  /*  8.70:1  requirement met      */
#define DS_WARN           DS_HEX(0xFFC24B, 255)  /* 10.16:1  within 10% of target */
#define DS_BAD            DS_HEX(0xFF5C6C, 255)  /*  5.44:1  unmet                */
/* Colour-blind alternate triad (Okabe-Ito). Settings-selectable. */
#define DS_CVD_OK         DS_HEX(0x56B4E9, 255)
#define DS_CVD_WARN       DS_HEX(0xF0E442, 255)
#define DS_CVD_BAD        DS_HEX(0xE69F00, 255)
/* !! MEASURED: greyscale delta OK-vs-BAD = 17 (need >=25); CVD pair = 4.
 * !! NEITHER triad separates without colour. Every state indicator MUST also
 * !! carry a shape-distinct glyph and a numeric delta. See ds_widget_req_row(). */

/* ---- stat categories (1:1 with digislots.fippi.io Bootstrap row classes) - */
#define DS_CAT_IDENT      DS_HEX(0x94A3B8, 255)  /* was table-secondary          */
#define DS_CAT_BATTLE     DS_HEX(0x38BDF8, 255)  /* was table-info               */
#define DS_CAT_CARE       DS_HEX(0x818CF8, 255)  /* was table-primary            */
#define DS_CAT_WEIGHT     DS_HEX(0xFB7185, 255)  /* was table-danger             */
#define DS_CAT_RAISE      DS_HEX(0xFBBF24, 255)  /* was table-warning            */

/* ---- Digivice LCD plate ------------------------------------------------- */
#define DS_LCD_FACE       DS_HEX(0xC9D8EE, 255)  /* darkest sprite px -> 13.48:1 */
#define DS_LCD_FACE_ALT   DS_HEX(0xCBDCC0, 255)  /* classic pale-green Digivice  */
#define DS_LCD_EDGE       DS_HEX(0x8FA8CF, 255)  /* 1px inner bevel, top-left    */
#define DS_LCD_SCAN       DS_HEX(0x05070D, 26)   /* 10% scanline, every 3px      */
#define DS_LCD_SCAN_PITCH 3

/* ---- stage / type chip colours ------------------------------------------ */
#define DS_STAGE_FRESH    DS_HEX(0x94A3B8, 255)
#define DS_STAGE_INTRAIN  DS_HEX(0x38BDF8, 255)
#define DS_STAGE_ROOKIE   DS_HEX(0x3DD68C, 255)
#define DS_STAGE_CHAMPION DS_HEX(0xFBBF24, 255)
#define DS_STAGE_ULTIMATE DS_HEX(0xFB7185, 255)
#define DS_TYPE_DATA      DS_HEX(0x38BDF8, 255)
#define DS_TYPE_VACCINE   DS_HEX(0x3DD68C, 255)
#define DS_TYPE_VIRUS     DS_HEX(0xC084FC, 255)

/* ---- overlay scrims (alphas are MEASURED, not chosen) -------------------- */
/* Composited over a worst-case pure-white game pixel, DS_BG_VOID gives:
 *   a=0.55  text_hi  4.20:1   accent 1.57:1
 *   a=0.72  text_hi  7.92:1   accent 2.96:1
 *   a=0.80  text_hi 10.84:1   accent 4.05:1
 *   a=0.86  text_hi 13.41:1   accent 5.01:1                                   */
#define DS_SCRIM_PANEL    DS_ALPHA(DS_BG_VOID, DS_A(82)) /* full panel         */
#define DS_SCRIM_STRIP    DS_ALPHA(DS_BG_VOID, DS_A(72)) /* HUD strip          */
#define DS_SCRIM_MODAL    DS_ALPHA(DS_BG_VOID, DS_A(72)) /* app modal backdrop */
/* !! At STRIP alpha, DS_ACCENT as TEXT is 2.96:1 and FAILS.
 * !! On the HUD strip, accent may only ever be a chip BACKGROUND.            */

/* ============================================================================
 *  TYPOGRAPHY
 *  Six steps, and six is the hard cap: vita2d's glyph atlas is a fixed
 *  512x512 U8_R111 texture (vita2d_font.c ATLAS_DEFAULT_W/H) that SILENTLY
 *  DROPS glyphs on overflow. Measured occupancy of this scale = 51.3% raw,
 *  ~68% after packing waste. Adding a 7th size risks silent text loss.
 *  "Large text" accessibility is a SWAP of these six, never an addition.
 *  NEVER pass an animated / interpolated value as vita2d_font_draw_text's
 *  `size` argument — it allocates a new atlas row per frame.
 * ========================================================================== */
typedef enum {
    DS_FONT_CAPTION = 0,   /* 14 */
    DS_FONT_BODY,          /* 18 */
    DS_FONT_LABEL,         /* 22 */
    DS_FONT_TITLE,         /* 28 */
    DS_FONT_DISPLAY,       /* 40 */
    DS_FONT_REEL,          /* 56 */
    DS_FONT_COUNT
} ds_font_t;

static const uint8_t  DS_FONT_PX[DS_FONT_COUNT]      = { 14, 18, 22, 28, 40, 56 };
static const uint8_t  DS_FONT_LINE[DS_FONT_COUNT]    = { 20, 26, 30, 36, 48, 64 };
static const float    DS_FONT_TRACK[DS_FONT_COUNT]   = { 0.06f, 0.0f, 0.02f,
                                                         0.0f, 0.10f, 0.08f };
static const uint8_t  DS_FONT_CAPS[DS_FONT_COUNT]    = { 1, 0, 0, 0, 1, 1 };

/* Readability floors. 960x544 on a 5" panel is ~220 PPI => 1 px ~= 1.32 arcmin
 * at 30 cm. Sustained reading needs >=16 arcmin cap height => ~18 px em.      */
#define DS_MIN_PX_PROSE       18
#define DS_MIN_PX_LABEL       14
#define DS_MIN_PX_PROSE_OVL   20   /* overlay sits over moving gameplay */
#define DS_MIN_PX_LABEL_OVL   16

/* PromptFont codepoints (SIL OFL). UTF-8 literals are safe: vita2d_font_draw_text
 * decodes UTF-8 (vita2d_font.c: i += utf8_to_ucs2(...)). Every advance is 1.0em. */
#define DS_G_CROSS    "\u21E3"   /* X      */
#define DS_G_CIRCLE   "\u21E2"   /* O      */
#define DS_G_SQUARE   "\u21E0"   /* []     */
#define DS_G_TRIANGLE "\u21E1"   /* /\     */
#define DS_G_L1       "\u21B0"
#define DS_G_R1       "\u21B1"
#define DS_G_SELECT   "\u21F7"
#define DS_G_START    "\u21F8"
#define DS_G_DPAD     "\u21CE"
#define DS_G_DPAD_LR  "\u21A2"
#define DS_G_DPAD_UD  "\u21A3"
#define DS_G_STICK    "\u21CD"
/* status glyphs live in the UI font subset, not PromptFont */
#define DS_G_MET      "\u2713"   /* check    */
#define DS_G_UNMET    "\u2717"   /* ballot X */
#define DS_G_CLOSE    "\u25B2"   /* triangle */

/* ============================================================================
 *  GRID  (960 x 544)
 * ========================================================================== */
#define DS_SCREEN_W        960
#define DS_SCREEN_H        544
#define DS_UNIT              8
#define DS_HALF              4
#define DS_MARGIN           32
#define DS_CONTENT_W       896                 /* 960 - 2*32                 */
#define DS_COLS             12
#define DS_COL_W            60
#define DS_GUTTER           16                 /* (896 - 11*16) / 12 == 60   */
#define DS_COL_X(n)        (DS_MARGIN + 76 * (n))
#define DS_COL_SPAN(k)     (76 * ((k) - 1) + DS_COL_W)

/* fixed vertical bands */
#define DS_BAND_TITLE_Y      0
#define DS_BAND_TITLE_H     56
#define DS_BAND_TABS_Y      56
#define DS_BAND_TABS_H      48
#define DS_BAND_CONTENT_Y  104
#define DS_BAND_CONTENT_H  384
#define DS_BAND_FOOTER_Y   488
#define DS_BAND_FOOTER_H    56

/* ---- radii -------------------------------------------------------------- */
typedef enum { DS_R_SM = 4, DS_R_MD = 8, DS_R_LG = 12, DS_R_XL = 20,
               DS_R_PILL = 255 /* sentinel: backend uses h/2 */ } ds_radius_t;

/* ---- borders / rules ---------------------------------------------------- */
#define DS_BORDER_PX         2   /* anything that must be SEEN (220 PPI!)     */
#define DS_DIVIDER_PX        1   /* DS_LINE_SOFT only                         */
#define DS_FOCUS_RING_PX     2
#define DS_FOCUS_INSET_PX    2

/* ---- sprite scale ladder (source art is 16x16) -------------------------- */
#define DS_SPRITE_SRC       16
#define DS_SPR_2X           32   /* list icon, HUD strip      */
#define DS_SPR_3X           48   /* dex cell, evolution strip */
#define DS_SPR_4X           64   /* requirement header, card  */
#define DS_SPR_6X           96   /* overlay full panel        */
#define DS_SPR_8X          128   /* dex modal hero            */
#define DS_SPR_10X         160   /* status portrait, reel     */
#define DS_ATLAS_W         128
#define DS_ATLAS_COLS        8   /* 63 sprites fit 8x8 = 64 cells exactly     */
#define DS_ATLAS_U(id)     (((id) % DS_ATLAS_COLS) * DS_SPRITE_SRC)
#define DS_ATLAS_V(id)     (((id) / DS_ATLAS_COLS) * DS_SPRITE_SRC)
/* Sprites drawn WITHOUT an LCD plate need a synthesized rim: 4 tinted copies
 * at +/-2px in x and y in DS_LCD_EDGE @ 35%, then the real sprite on top.
 * 48-63% of each sprite's opaque pixels are near-black (measured); the
 * darkest (#1A0800) is 1.02:1 on DS_BG_BASE, i.e. invisible.                 */
#define DS_RIM_OFFSET_PX     2
#define DS_RIM_COLOR       DS_ALPHA(DS_LCD_EDGE, DS_A(35))

/* ============================================================================
 *  MOTION  — frames @ 60 Hz, never milliseconds (the loop is vsync-locked)
 * ========================================================================== */
#define DS_FR_FOCUS           8   /* easeOutCubic  */
#define DS_FR_PRESS_IN        4
#define DS_FR_PRESS_OUT       6
#define DS_FR_TAB_SWITCH     11   /* + 12 px slide */
#define DS_FR_PANEL_IN       14   /* easeOutCubic, y +16 -> 0 */
#define DS_FR_PANEL_OUT       9   /* easeInQuad    */
#define DS_FR_MODAL_IN       12   /* easeOutBack, scale .94 -> 1.0 */
#define DS_FR_TOAST_IN       14
#define DS_FR_TOAST_HOLD    144
#define DS_FR_TOAST_OUT      12
#define DS_FR_BAR_FILL       20   /* easeOutQuart  */
#define DS_FR_OVERLAY_IN     12
#define DS_FR_OVERLAY_OUT     8
#define DS_FR_TAB_SLIDE_PX   12
#define DS_FR_PANEL_RISE_PX  16
#define DS_PRESS_SCALE     0.96f

/* reel choreography (total 142 frames ~= 2.37 s) */
#define DS_REEL_SPINUP_END   17   /* easeInQuad,  v 0 -> 42 px/fr */
#define DS_REEL_CRUISE_END   65   /* constant 42 px/fr            */
#define DS_REEL_DECEL_END   107   /* easeOutQuart                 */
#define DS_REEL_SETTLE_END  121   /* easeOutBack, +/-10 px        */
#define DS_REEL_REVEAL_END  141
#define DS_REEL_SKIP_TO     108   /* pressing X mid-spin jumps here */
#define DS_REEL_LOCK_1       90   /* STAGE reel locks   */
#define DS_REEL_LOCK_3      102   /* TYPE reel locks    */
#define DS_REEL_LOCK_2      114   /* PORTRAIT locks LAST (payoff) */
#define DS_REEL_V_MAX      42.0f
#define DS_REEL_BLUR_DY      28   /* ghost copies at -28 / 0 / +28 */
#define DS_REEL_INPUT_LOCK  128   /* L1/R1 tab change suppressed until here */

/* Reduce-motion clamp: every DS_FR_* becomes this, reel skips to reveal. */
#define DS_FR_REDUCED         4

/* ============================================================================
 *  WIDGET METRICS
 * ========================================================================== */
#define DS_STATBAR_W       288   /* == DS_COL_SPAN(4) */
#define DS_STATBAR_H        40
#define DS_STATBAR_TRACK_H  10
#define DS_STATBAR_TRACK_Y  26
#define DS_STATBAR_NOTCH_W   2   /* requirement notch */
#define DS_STATBAR_PIP_H     6

#define DS_REQROW_W        896
#define DS_REQROW_H         48
#define DS_REQROW_PITCH     56
#define DS_REQROW_RAIL_W     4
#define DS_REQROW_GLYPH_X   16
#define DS_REQROW_GLYPH_D   24
#define DS_REQROW_LABEL_X   52
#define DS_REQROW_LABEL_W  180
#define DS_REQROW_BAR_X    248
#define DS_REQROW_BAR_W    320
#define DS_REQROW_VAL_X    584
#define DS_REQROW_VAL_W    180
#define DS_REQROW_CHIP_X   780
#define DS_REQROW_CHIP_W   116
#define DS_REQROW_CHIP_H    28

#define DS_CARD_W          116   /* dex grid cell */
#define DS_CARD_H          116
#define DS_CARD_PITCH      130   /* 7 cols: 7*116 + 6*14 == 896 */
#define DS_CARD_COLS         7
#define DS_CARD_ROWS         3   /* 3*130 - 14 == 376 <= 384    */
#define DS_CARD_PLATE_W     84
#define DS_CARD_PLATE_H     60
#define DS_CARD_STAGEBAR_H   4

#define DS_TAB_COUNT         5
#define DS_TAB_W           176   /* 5*176 + 4*4 == 896 */
#define DS_TAB_PITCH       180
#define DS_TAB_UNDERLINE_H   3

#define DS_BUTTON_H         48
#define DS_BUTTON_PAD_X     24
#define DS_TOAST_W         400
#define DS_TOAST_H          64
#define DS_TOAST_X         280
#define DS_TOAST_Y          72
#define DS_MODAL_W         640
#define DS_MODAL_H         384
#define DS_MODAL_X         160
#define DS_MODAL_Y          80
#define DS_SCROLLBAR_X     940   /* in the right margin, outside content */
#define DS_SCROLLBAR_W       4
#define DS_SCROLLBAR_MIN_H  48
#define DS_SCROLLBAR_FADE   60   /* frames of idle before fade-out */

/* ============================================================================
 *  OVERLAY GEOMETRY
 *  PSX (SLUS-01032) is 4:3. Scaled to fill 544 height => 725.33 wide, centred:
 *  the game image occupies x = 117..843, leaving two 117 px pillarbox bars.
 *  [LIKELY — depends on the user's Adrenaline scale setting. Measure at
 *   runtime where possible; expose DS_OVL_ANCHOR_* as a user setting.]
 * ========================================================================== */
#define DS_GAME_43_W       726
#define DS_GAME_43_X0      117
#define DS_GAME_43_X1      843

/* HUD strip, variant A: right pillarbox — occludes ZERO gameplay pixels */
#define DS_OVL_RAIL_X      847
#define DS_OVL_RAIL_Y        8
#define DS_OVL_RAIL_W      105
#define DS_OVL_RAIL_H      528
/* HUD strip, variant B: top edge — for any other PSX scale setting */
#define DS_OVL_STRIP_X       0
#define DS_OVL_STRIP_Y       0
#define DS_OVL_STRIP_W     960
#define DS_OVL_STRIP_H      64
/* Full panel: 640 < 726, so it ALWAYS sits inside the 4:3 image and never
 * spills onto the pillarbox at any scale setting. */
#define DS_OVL_PANEL_X     160
#define DS_OVL_PANEL_Y      48
#define DS_OVL_PANEL_W     640
#define DS_OVL_PANEL_H     448
#define DS_OVL_ROW_W       576
#define DS_OVL_ROW_H        44
#define DS_OVL_ROW_PITCH    52
#define DS_OVL_ROW_X       192
#define DS_OVL_ROW_Y0      192
#define DS_OVL_ROWS_VISIBLE  5

/* ============================================================================
 *  STAT DISPLAY CAPS
 *  FIXED so bars stay comparable roll-to-roll. Requirement maxima observed in
 *  src/models/digimon.c are lower (HP 7000, MP 6000, OFF 500, DEF 600, SPD 400,
 *  BRN 600, CARE 15, WT 65, DISC 100, HAPPY 95, BAT 100, TECH 49); the caps
 *  below are the game-legal ceilings. value > cap => clamp fill, draw a 2px
 *  DS_WARN chevron at the right cap.
 * ========================================================================== */
static const int32_t DS_STAT_CAP[] = {
    /* HP */ 9999, /* MP */ 9999, /* OFF */ 999, /* DEF */ 999,
    /* SPD */ 999, /* BRN */ 999, /* WEIGHT */ 99, /* CARE */ 15,
    /* DISCIPLINE */ 100, /* HAPPINESS */ 100, /* BATTLES */ 100, /* TECHS */ 49
};

/* Weight is a RANGE, not a minimum: the game requires target +/- 5 g.
 * A stored req of 30 means 25..35. Do not render it as ">= 30". */
#define DS_WEIGHT_TOLERANCE   5
/* "Close" threshold for the DS_WARN state on a requirement row. */
#define DS_CLOSE_PERMILLE    100   /* within 10% of target */

#endif /* DS_THEME_H */
```

---

## 8. Screen designs

Ruler convention for every wireframe below: **each ASCII column ≈ 10 px**, and the `y` gutter gives real pixel rows. Fixed bands on every standalone screen: title `y0–56` · tabs `y56–104` · content `y104–488` · footer `y488–544`.

### 8.1 S1 — DIGIMON STATUS

```
x:0        100       200       300       400       500       600       700       800       900  960
   |---------|---------|---------|---------|---------|---------|---------|---------|---------|----|
y0 ┌──────────────────────────────────────────────────────────────────────────────────────────────┐
   │ DIGISLOTS·VITA          SLUS-01032 · SLOT 1 · ● SYNCED          ↰ L1        R1 ↱             │  bg: grad #111827→#0A0F1A
y56├──────────────────────────────────────────────────────────────────────────────────────────────┤  2px #33456B rule
   │ ▎ STATUS  ▎  │    ROLL     │ REQUIREMENTS │  EVOLUTIONS  │   SETTINGS   │                    │  5 tabs, x=32+180n, w176, h48
y104├═════════════┷══════════════════════════════════════════════════════════════════════════════─┤  3px #2E9BFF underline slides
   │ ┌───────────────────────┐   BATTLE  ▸ cat-battle #38BDF8      CONDITION ▸ per-category       │
   │ │ (32,112) 288x256 r-lg │                                                                     │
   │ │  ┌─────────────────┐  │   HP            2140 │   WEIGHT              22 │                   │
   │ │  │ LCD PLATE       │  │   ▓▓▓▓▓▓▓░░░░░┊░░░░░ │   ▓▓▓▓▓▓▓▓▓▓▓░┊░░░░░░░░ │  ┊ = 2px req notch
   │ │  │ (56,136)240x180 │  │                      │                          │
   │ │  │ #C9D8EE r-md    │  │   MP            1980 │   CARE MISTAKES        1 │
   │ │  │  ┌───────────┐  │  │   ▓▓▓▓▓▓░░░┊░░░░░░░░ │   ▓░░░░░░░░░░░░░░░░░░░░ │
   │ │  │  │ SPRITE    │  │  │                      │                          │
   │ │  │  │ 16x16@10x │  │  │   OFFENSE        312 │   DISCIPLINE         58% │
   │ │  │  │ =160x160  │  │  │   ▓▓▓▓▓▓░░░░┊░░░░░░░ │   ▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░ │
   │ │  │  │ (96,146)  │  │  │                      │                          │
   │ │  │  └───────────┘  │  │   DEFENSE        288 │   HAPPINESS          56% │
   │ │  │  ░scanlines░    │  │   ▓▓▓▓▓░░░░░┊░░░░░░░ │   ▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░ │
   │ │  └─────────────────┘  │                      │                          │
   │ │      G R E Y M O N    │   SPEED          204 │   BATTLES             14 │
   │ │   ( CHAMPION )( VACC )│   ▓▓▓▓░░░░┊░░░░░░░░░ │   ▓▓▓░░░░░░░░░░░░░░░░░░ │
   │ └───────────────────────┘                      │                          │
   │ ┌───────────────────────┐   BRAINS         256 │   TECHNIQUES        12/49│
   │ │ TAMER  FIPPI   LV 12  │   ▓▓▓▓▓░░░░┊░░░░░░░░ │   ▓▓▓▓▓░░░░░░░░░░░░░░░░ │
   │ │ AGE 3d 14h  LIVES 3   │                      │                          │
y488├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ ⇣ INFO    ⇡ ROLL    ↕ SELECT    ↰↱ TABS                              ⇷ INFO TEXT   ⇸ ABOUT   │
y544└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

**Exact rects.** Portrait card `(32,112,288,256)` r-lg `surface-1` + e1. LCD plate `(56,136,240,180)` r-md `#C9D8EE`, 1 px `#8FA8CF` top-left bevel, scanlines `#05070D`@10 % every 3 px. Sprite 160×160 at `(96,146)`, POINT-filtered. Name 28 px centred, baseline y=352. Chips 28-tall centred y=372. Meta card `(32,392,288,96)`. Stat sub-column A `x336..624` (288 w), sub-column B `x640..928` (288 w); category label 14 px CAPS at y=112; six 40-tall bars per column at pitch 44 → y = 136/180/224/268/312/356.

**Interaction.** D-pad ↑↓ moves a focus ring (2 px `accent-hi`, inset 2 px) between the 12 stat rows; ←→ crosses columns. **✕** toasts that stat's tooltip, reusing the web app's copy verbatim (e.g. HP → *"Your digimon's health. Increased by HP training at the Gym."* **[V]**). **△** jumps to ROLL. **□** toggles the LCD plate between blue-white and Digivice-green. **○** does nothing (top level). **L1/R1** cycle tabs with wraparound — this is the Vita convention and it means the D-pad is never stolen from list navigation. Left stick mirrors the D-pad (0.4 deadzone, 400 ms delay → 250 ms repeat). Front touch: tap a stat row = tooltip, tap a tab = switch.

> Touch coordinates are in **panel space, not screen space**. Query `sceTouchGetPanelInfo()` for `min/maxAaX/Y` and scale — do not hardcode 1920×1088. This is the classic first bug on Vita.

### 8.2 S2 — EVOLUTION ROLL (the slot machine, signature screen)

```
y104├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │                        (32,112,896,32)  R O L L I N G . . .   /  ▸ RESULT                     │  40px display, centred
y150│    ┌──────────────┐        ┌────────────────────────────┐        ┌──────────────┐            │
   │    │ REEL 1       │        │ REEL 2   ★ PAYOFF          │        │ REEL 3       │            │
   │    │ (96,150)     │        │ (320,150)                  │        │ (672,150)    │            │
   │    │ 192 x 250    │        │ 320 x 250                  │        │ 192 x 250    │            │
   │    │ ░vignette░   │        │ ░vignette░                 │        │ ░vignette░   │            │
y275├════│══ STAGE ═════│════════│════ ▓sprite 16x16@10x▓ ════│════════│══  TYPE  ════│═══════════─┤ PAYLINE y=275
   │    │  CHAMPION    │        │      160x160 @ (400,195)    │        │   VACCINE    │            │  2px #2E9BFF + additive glow
   │    │ ░vignette░   │        │ ░vignette░                 │        │ ░vignette░   │            │
   │    └──────────────┘        └────────────────────────────┘        └──────────────┘            │
y400│                                                                                              │
   │                          ▸  G R E Y M O N                                                     │  40px display, centred y=412
   │                     4 of 6 requirements met   ▓▓▓▓▓▓▓▓░░░░  (296,456,368,12)                  │
y488├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ ⇣ PULL LEVER   ⇡ RE-ROLL   ⇠ FILTERS                              ⇢ BACK   swipe↓ = pull      │
y544└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

Geometry check: 96+192=288, gap 32; 320+320=640, gap 32; 672+192=864, right margin 96 — symmetric, centre reel centred on x=480. ✓

Reel body `surface-2` with a top-and-bottom inner vignette (two 40 px `vita2d_draw_array` gradients from `#05070D`@85 % to transparent) so items appear to emerge from and vanish into shadow. Clip the strip with `vita2d_set_clip_rectangle()`.

**Choreography** (frames @60 Hz — this is the whole reason the app is called DigiSlots):

| Phase | Frames | Behaviour |
|---|---|---|
| P1 SPIN-UP | 0–17 (18) | easeInQuad, velocity 0 → 42 px/fr, all three reels together |
| P2 CRUISE | 18–65 (48) | constant 42 px/fr. Motion blur = draw the cell 3× at y−28/0/+28 with alpha 30/45/30 %. Strip is the full 62-entry list, wrapped |
| P3 DECEL | 66–107 (42) | easeOutQuart into the target cell. **Staggered stops:** reel 1 (STAGE) locks at fr 90, reel 3 (TYPE) at fr 102, **reel 2 (PORTRAIT) LAST at fr 114** |
| P4 SETTLE | 108–121 (14) | easeOutBack, ±10 px overshoot per reel. Each lock flashes the payline `#7CC4FF` for 3 fr + additive `glow.png` 100 %→0 over 10 fr |
| P5 REVEAL | 122–141 (20) | name types on 1 char every 2 fr; progress bar fills easeOutQuart; 6/6 fires the ✓ toast |

Total ≈ 142 fr ≈ 2.37 s. Centre-last is the classic slot tension curve and it also means the payoff *is* the sprite.

The reels are semantically honest: reel 2 is the rolled Digimon; reels 1 and 3 are **its** stage and type, revealed progressively.

**Interaction.** **✕ = pull the lever.** Pressing **✕ again mid-spin skips to frame 108** — never make the user watch 2.4 s twice. **○** back to STATUS. **△** re-roll with the same filters. **□** opens the roll-filter popover (specials on/off, exclude-owned on/off, restrict-to-stage). **L1/R1 are suppressed during frames 0–128** so a tab change can't interrupt the animation. Front-touch anywhere on the reel bank pulls the lever, and a **downward swipe over the reels** also pulls it — the physical-lever gesture, and the single best touch affordance in the app. Reduce-motion skips P1–P4 and cross-fades to the result over 4 fr.

### 8.3 S3 — EVOLUTION REQUIREMENTS CHECKLIST

```
y104├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
   │ │(32,112,896,64) [▓64x64▓] GREYMON  (CHAMPION)(VACCINE)      4 / 6 MET  ▓▓▓▓▓▓▓▓░░░░       │ │
   │ └──────────────────────────────────────────────────────────────────────────────────────────┘ │
y184│ ▌✓ OFFENSE      ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  312 / 300      [ +12 ]                                │  MET   rail #38BDF8
y240│ ▌✓ SPEED        ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  204 / 200      [  +4 ]                                │  MET
y296│ ▌▲ BRAINS       ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░  256 / 280      [ −24 ]                                │  CLOSE rail #38BDF8
y352│ ▌✕ WEIGHT       ▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░   22 /  30      [  −8 ]                                │  UNMET rail #FB7185
y408│ ▌✓ CARE MISTAKE ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓    1 /   3      [  OK ]                                │  MET   rail #818CF8
y464│ ▌✕ TECHNIQUES   ▓▓▓▓▓▓▓░░░░░░░░░░░░░   12 /  30      [ −18 ]                                │  UNMET rail #FBBF24
   │                                                                                    ▏scroll   │  4px bar at x=940
y488├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ ⇣ WHAT IS THIS?   ⇠ HIDE MET   ⇡ RE-ROLL   ↕ SCROLL                              ⇢ BACK      │
y544└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

Row = `(32, y, 896, 48)` r-md, pitch 56 → 5.5 visible; the list scrolls while the focus ring parks in slot 2 of 5 once past the midpoint. Row anatomy, offsets from row x: rail `(+0, w4, h48)` in the category colour, r-sm left only · status disc `(+16,+12, 24×24)` · label 18 px `(+52, w180)` · mini bar `(+248,+19, 320×10)` · `current / required` 18 px right-aligned `(+584, w180)` with `current` in the state colour · delta chip `(+780,+10, 116×28)` r-sm, fill = state colour @16 %, text = state colour.

**Status glyph encoding** — MET = `✓` U+2713 in `ok` on a *filled* 24 px `ok`@18 % disc; UNMET = `✕` U+2717 in `bad` on an *unfilled* disc with a 2 px `bad` ring; CLOSE (within 10 %) = `▲` U+25B2 in `warn`. **Four redundant encodings per row — rail hue, glyph shape, bar fill, signed delta — is what makes this survive CVD and greyscale**, given the measured 17-point greyscale delta between `ok` and `bad`.

**Semantics the current code gets wrong and this screen must fix** (all **[V]** against `src/models/digimon.c`):
- Comparison is `>` today; the game rule is **match or exceed (`>=`)**. Eleven copy-pasted sites, all off by one.
- **Weight is a range**, not a minimum. Stored `30` means **25–35**. Render as `25 - 35 (current 42) ✕`, never `>= 30`.
- `minCare` / `minBattles` are parsed and **never read**. A `Care ≤ 1` requirement currently renders as `≥`. Because care mistakes only rise and reset on evolution, a blown max-care requirement is **permanently lost** — render it struck-through, not merely red.
- `techs` is stored for nearly every champion/ultimate and is **never displayed at all** (`print("Tech: TODO\n\n")`).
- The real rule is **fulfil 3 of 4 requirement GROUPS** (Stats / Weight / Care / Bonus), where Bonus is an OR over {happiness, discipline, battles, techniques, free-digimon-bonus}. The screen must show group headers and a `groups_met / 4` roll-up, not 11 flat rows implying all are required.

A Digimon-bonus row ("Greymon gets a free natural bonus point") renders as a full-width `#FBBF24`@14 % banner above the list.

**Interaction.** ↑↓ scroll one row per press. **✕** toasts the focused requirement's tooltip. **□** toggles "hide already-met" (mirrors the web app's `hideIrrelevant` prop **[V]**). **△** back to ROLL to re-roll. **○** back. Touch drag scrolls with momentum.

### 8.4 S4 — ALL-EVOLUTIONS BROWSER (the "dex")

```
y104├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ (32,108,896,28)  STAGE: ‹ ALL ›     SORT: BY STAGE     SHOWN 62 / 62     OWNED 11            │  14px CAPS filter strip
y144│ ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐                                            │  cell 116x116 r-lg
   │ │▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ │   x = 32 + 130n  (n=0..6)                  │  LCD plate 84x60 at +16,+10
   │ │Agumo││Gabum││Betam││Biyom││Palmo││Patam││Elecm│   7 cols × 116 + 6 × 14 = 896 ✓            │  sprite 3x = 48
   │ │▁▁▁▁▁││▁▁▁▁▁││▁▁▁▁▁││▁▁▁▁▁││▁▁▁▁▁││▁▁▁▁▁││▁▁▁▁▁│   4px stage bar on bottom edge             │
   │ └─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘                                            │
y274│ ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐   y = 144 + 130m  (m=0,1,2)                │
   │ │▓48▓ ││▓48▓ ││ ?   ││▓48▓ ││▓48▓ ││ ?   ││▓48▓ │   "?" = silhouette in --surface-3          │
   │ │Greym││Tyran││ ??? ││Meram││Leomo││ ??? ││Angem│   (spoiler-safe, matches web app ethos)    │
   │ └─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘                                            │
y404│ ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐   3 rows × 130 − 14 = 376 ≤ 384 ✓         │
   │ │▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ ││▓48▓ │   21 per page, 62 entries = 3 pages        │
   │ └─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘                              ● ○ ○ pages   │
y488├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ ⇣ DETAILS   ⇠ STAGE FILTER   ⇡ SORT   ⇎ MOVE                                     ⇢ BACK      │
y544└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

Card states: **locked/unseen** → sprite drawn once in `surface-3` as a silhouette + `?`; **owned** → 2 px `ok` corner tick; **focused** → 2 px `accent-hi` ring + e1 shadow + scale 1.04 over 8 fr.

**✕ opens a 640×384 centred modal**: hero sprite at 8× (128 px) on an LCD plate, name/stage/type chips, that Digimon's own requirement rows, and a horizontal strip of **its** evolution targets. `evolutionPathSize` ranges **1..9** **[V]** (parsed from `digimon.c`), so the strip must lay out up to 9 targets: 9×48 + 8×12 = 528 px, which fits the 576 px modal content width. ✓

Unowned Digimon are silhouettes by default, keeping the browser spoiler-free — matching the web app's positioning. A Settings switch reveals all.

**Interaction.** D-pad moves in 4 directions across the 7-wide grid. **L1/R1 remain tabs**, so paging uses **↑ from row 0 / ↓ from the last row**, or **←** from column 0 to page left. **□** cycles the stage filter (ALL → Fresh → In-Training → Rookie → Champion → Ultimate). **△** cycles sort (by ID / by name / by stage). **○** back. Touch: tap a cell to open, drag to scroll.

### 8.5 S5 — SETTINGS

```
y104├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │  DATA SOURCE                                                                     14px CAPS   │
   │  Memory card slot            The .VMP POPS reads from                    ‹ SLOT 1 ›          │  row 64 tall
   │  Save file path              ux0:/pspemu/PSP/SAVEDATA/SLUS01032/…            [ BROWSE ]      │
   │                                                                                              │
   │  ROLL RULES                                                                                  │
   │  Show special evolutions     Numemon, Sukamon, Kunemon, Vademon              (●━━) ON         │  toggle 64x32 r-pill
   │  Exclude owned digimon       Skip anything already in your tree              (━━○) OFF        │
   │  Remember last roll          Restore on next launch                         (●━━) ON         │
   │                                                                                              │
   │  DISPLAY                                                                                     │
   │  Show info text              Tooltips explaining each stat                  (●━━) ON         │
   │  Colour-blind palette        Okabe-Ito blue / orange                        (━━○) OFF        │
   │  Reduce motion               Disables the reel spin                         (━━○) OFF        │
   │  Large text                  Bumps every size one step                      (━━○) OFF        │
   │  LCD plate tint              Digivice screen colour                     ‹ BLUE-WHITE ›       │
   │                                                                                              │
   │  OVERLAY                                                                                     │
   │  Overlay mode                                                     ‹ HUD STRIP / FULL PANEL › │
   │  Summon gesture              Rear double-tap · L1+R1+SELECT                ‹ REAR TAP ›      │
   │  HUD anchor                  Pillarbox is zero-occlusion at 4:3      ‹ PILLARBOX / TOP ›     │
   │  Overlay status              Last session: 0 frames seen  ▸ CHECK SETUP        [ OPEN ]      │
y488├──────────────────────────────────────────────────────────────────────────────────────────────┤
   │ ⇣ ACTIVATE   ↔ CHANGE   ↕ MOVE                                                   ⇢ BACK      │
y544└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

Rows `(32, y, 896, 64)`; label 18 px `text-hi` at +24, description 14 px `text-lo` below; control right-aligned to x=904. Section headers 14 px CAPS `accent`, 40 tall. Scrolls; ~6 rows visible.

The first three ROLL/DISPLAY toggles are **exactly the web app's three switches, with its own label copy** **[V]**: *"Show special digimon evolutions (eg. Numemon)"*, *"Exclude one or more digimon from the results"*, *"Show info text"*.

The last row, **Overlay status**, is load-bearing: it reads the heartbeat the plugin wrote to `ur0:data/digislots/overlay.log` and opens the **Overlay Setup** modal, which states verbatim: *"Adrenaline → Settings → Graphics Filtering must be set to anything except* **Original***. In Original mode Adrenaline presents no frame, so no overlay can draw."* This is the only place that constraint can be surfaced, because an invisible overlay cannot report its own invisibility.

**Interaction.** ↑↓ move; **←→ change the focused row's value** (toggle, cycle enum, nudge slot); **✕** activates row-typed controls (opens the file browser, opens the exclude-Digimon picker); **○** back, saving on exit to `ux0:data/digislots/settings.bin` (versioned struct with a magic+version prefix).

### 8.6 S6a — OVERLAY: HUD STRIP (compact, read-only, non-input-stealing)

**Variant A — PILLARBOX.** Zero gameplay occlusion. Requires the PSX image scaled to fill height (game at x=117..843) **[L]**.

```
x:0    117                                                                     843          960
   ┌────┬──────────────────────────────────────────────────────────────────────┬─────────────┐
   │▓▓▓▓│                                                                      │ DIGISLOTS   │ y8
   │▓▓▓▓│                                                                      │ ─────────── │
   │▓▓▓▓│                                                                      │  ┌───────┐  │
   │HUD │        4 : 3   G A M E   I M A G E    726 x 544                       │  │ ▓48▓  │  │ sprite 3x
   │RAIL│        (PSX / Digimon World 1 via POPS)                               │  └───────┘  │
   │x847│                                                                      │  GREYMON    │ 18px
   │w105│                                                                      │  ─────────── │
   │    │                                                                      │  TARGET     │ 14px CAPS
   │    │                                                                      │  ┌───────┐  │
   │    │                                                                      │  │ ▓48▓  │  │
   │    │                                                                      │  └───────┘  │
   │    │                                                                      │  ANDROMON   │
   │    │                                                                      │  4/6 MET    │ 20px
   │    │                                                                      │  ▓▓▓▓▓▓░░░  │
   │    │                                                                      │  WT −8      │ 16px, --bad
   │    │                                                                      │  TEC −18    │
   └────┴──────────────────────────────────────────────────────────────────────┴─────────────┘
```

Right rail `(847,8,105,528)` r-lg, fill `#05070D`@72 %, 2 px `#33456B` border, e1. **Nothing is drawn inside x=117..843, so zero pixels of gameplay are occluded.** The left bar `(4,8,105,528)` is the mirror option. Only the **two worst-deficit requirements** are listed — the strip is a glance, not a report.

**Variant B — TOP STRIP.** For any other PSX scale, where the game fills the screen.

```
   ┌──────────────────────────────────────────────────────────────────────────────────────────────┐ y0
   │ ▓32▓ GREYMON (CHA)  │ HP▓▓▓▓░ MP▓▓▓░ OF▓▓▓▓░ DF▓▓▓░ SP▓▓░ BR▓▓▓░ │ ▸ ▓32▓ ANDROMON  4/6  ⇷ │ h64
   ├──────────────────────────────────────────────────────────────────────────────────────────────┤ y64
   │  x32   x76           x300                                          x660      x700       x920 │
   │                          G A M E   I M A G E                                                  │
```

`(0,0,960,64)`, fill `#05070D`@72 % (**measured 7.92:1** for `text-hi` over a worst-case white game pixel), 2 px `#2E9BFF` bottom rule. **No accent-coloured TEXT on this strip** — accent only as a chip background, because accent over a 0.72 scrim measures 2.96:1. Type floors rise to 20 px body / 16 px caption. Micro stat bars are 40×6, label 16 px CAPS above.

**Interaction: none.** This mode **consumes zero inputs and never masks the pad.** It re-reads the save on a timer and animates bar changes over 20 fr. This is the mode the user should live in.

### 8.7 S6b — OVERLAY: FULL PANEL (the "I stopped to think" mode)

```
x:0      117                                                                   843        960
   ┌──────┬───────────────────────────────────────────────────────────────────┬──────────┐ y0
   │ dim  │ ░░░░░ scrim #05070D @ 82% over the whole 960x544 ░░░░░░░░░░░░░░░░ │   dim    │
   │      │  ┌─────────────────────────────────────────────────────────────┐  │          │ y48
   │      │  │ (160,48) 640 x 448  r-xl  surface-1  2px #33456B  e2         │  │          │
   │      │  │  ┌────────────┐   G R E Y M O N        ▸ ANDROMON            │  │          │
   │      │  │  │ LCD 128x96 │   CHAMPION · VACCINE     4 / 6 MET           │  │          │
   │      │  │  │  ▓96▓ 6x   │   ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░                          │  │          │
   │      │  │  └────────────┘                                              │  │          │
   │      │  │  ▌✓ OFFENSE   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  312 / 300   [ +12 ]           │  │          │  row 576x44
   │      │  │  ▌▲ BRAINS    ▓▓▓▓▓▓▓▓▓▓▓▓░░░  256 / 280   [ −24 ]           │  │          │  pitch 52
   │      │  │  ▌✕ WEIGHT    ▓▓▓▓▓▓░░░░░░░░░   22 /  30   [  −8 ]           │  │          │
   │      │  │  ▌✕ TECHNIQUE ▓▓▓▓░░░░░░░░░░░   12 /  30   [ −18 ]           │  │          │
   │      │  │  ▌✓ CARE      ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓    1 /   3   [  OK ]           │  │          │
   │      │  │                                                              │  │          │
   │      │  │  ⇣ RE-ROLL      ↕ SCROLL          ⇢ / L1+R1+⇷  CLOSE         │  │          │  y440,h48
   │      │  └─────────────────────────────────────────────────────────────┘  │          │ y496
   └──────┴───────────────────────────────────────────────────────────────────┴──────────┘ y544
```

**640 < 726, so the panel always sits entirely inside the 4:3 game image and never spills onto the pillarbox** — it reads as intentional at every PSX scale setting. Scrim α=0.82 (measured **10.84:1** for `text-hi`, **4.05:1** for `accent`, over a worst-case white game pixel). Panel `(160,48,640,448)` r-xl. LCD plate `(184,72,128,96)`, sprite at 6× = 96 px. Requirement rows `(192, 192+52n, 576, 44)`, 5 visible. Enter 12 fr easeOutBack scale 0.94→1.0 + scrim fade; exit 8 fr easeInQuad.

**Interaction — the three-tier input ladder.** Under Adrenaline the PSX game owns the pad, so the overlay must assume it has *no* buttons until it takes them.

| Tier | Mechanism | Confidence |
|---|---|---|
| **1 — preferred** | **Rear-panel double-tap** to toggle the strip; **rear swipe-up** to promote to the full panel. PSX hardware has no touch input, so POPS cannot consume it. `SCE_TOUCH_PORT_BACK = 1` **[V]** (`psp2/touch.h`) | **[S]** — whether *Adrenaline* claims the rear panel is unverified. Test on hardware before committing. |
| **2 — fallback toggle** | **L1 + R1 + SELECT held 20 frames.** A chord DW1 cannot produce in normal play. The plugin swallows it via `sceCompatWriteSharedCtrl`. **Never L+SELECT** (Adrenaline POPS fast-forward **[V]**) and **never PS** (Adrenaline menu **[V]**). | **[L]** |
| **3 — while the panel is OPEN** | The plugin masks the pad from the game entirely, and the overlay gets a **deliberately tiny vocabulary: ↑↓ scroll, ✕ re-roll, ○ or the same chord to close.** Nothing else. | **[V]** mechanism |

> **Design rule, not a suggestion: do not build a five-tab UI into the overlay.** The moment the user needs L1/R1 you have taken the shoulder buttons away from a game that uses them. The HUD strip consumes nothing; the full panel is the "I stopped to think" mode and closes fast.

### 8.8 Button-hint footer — literal string constants

```c
/* Footer (0,488,960,56). 2px DS_LINE top rule, fill DS_BG_BASE.
 * Hint group = [24px PromptFont glyph, DS_TEXT_HI] + 8px + [18px label, DS_TEXT_MID].
 * 32px between groups. Positive actions from x=32 LTR; Back/Cancel right-aligned to x=928.
 * Every PromptFont advance is exactly 1.0em, so group width is trivially:
 *     24 + 8 + ui_text_w(DS_FONT_BODY, label)                                   */

#define HINT_ROLL   DS_G_CROSS    " PULL LEVER"
#define HINT_REROLL DS_G_TRIANGLE " RE-ROLL"
#define HINT_FILTER DS_G_SQUARE   " FILTERS"
#define HINT_BACK   DS_G_CIRCLE   " BACK"
#define HINT_TABS   DS_G_L1 DS_G_R1 " TABS"
#define HINT_MOVE   DS_G_DPAD_UD  " SELECT"
#define HINT_GRID   DS_G_DPAD     " MOVE"
#define HINT_INFO   DS_G_SELECT   " INFO TEXT"
#define HINT_ABOUT  DS_G_START    " ABOUT"
```

---

## 9. Component library

Everything below lives in `ui/render/` and is written **once**, against the backend interface, so it compiles into both the vita2d app and the software-blitter plugin. Implement in the order listed — each depends only on what precedes it.

### 9.1 Layer 0 — backend interface (implemented twice)

`ui/render/ui_backend.h`. This is the *only* file with two implementations: `ui_backend_vita2d.c` and `plugin/render/ui_backend_blit.c`.

```c
/* ui/render/ui_backend.h — the sole platform seam of the UI layer. */
#ifndef UI_BACKEND_H
#define UI_BACKEND_H
#include <stdint.h>
#include "ds_theme.h"

typedef struct { int x, y, w, h; } ui_rect_t;

/* --- lifecycle ---------------------------------------------------------- */
int  ui_backend_init  (void);           /* vita2d_init + asset load | memblock alloc  */
void ui_backend_fini  (void);
void ui_begin_frame   (void);           /* vita2d_start_drawing     | clear scratch   */
void ui_end_frame     (void);           /* end_drawing+swap_buffers | mark dirty      */
int  ui_frame_index   (void);           /* monotonic 60 Hz counter, drives all easing */

/* --- primitives --------------------------------------------------------- */
void ui_fill_rect     (int x, int y, int w, int h, uint32_t c);
void ui_fill_rect_v   (int x, int y, int w, int h, uint32_t top, uint32_t bottom);
void ui_fill_rect_h   (int x, int y, int w, int h, uint32_t left, uint32_t right);
void ui_stroke_rect   (int x, int y, int w, int h, int px, uint32_t c);
void ui_hline         (int x, int y, int w, int px, uint32_t c);
void ui_vline         (int x, int y, int h, int px, uint32_t c);

/* --- 9-slice / elevation ------------------------------------------------ */
void ui_panel         (int x, int y, int w, int h, ds_radius_t r, uint32_t fill);
void ui_panel_stroked (int x, int y, int w, int h, ds_radius_t r,
                       uint32_t fill, int border_px, uint32_t border);
void ui_shadow        (int x, int y, int w, int h, ds_radius_t r, int level /*1|2*/);
void ui_glow          (int cx, int cy, int radius, uint32_t c, uint8_t intensity);

/* --- clipping ----------------------------------------------------------- */
void ui_push_clip     (int x, int y, int w, int h);   /* stack depth >= 4 */
void ui_pop_clip      (void);

/* --- sprites ------------------------------------------------------------ */
void ui_sprite        (int sprite_id, int x, int y, int px);          /* px from ladder */
void ui_sprite_tint   (int sprite_id, int x, int y, int px, uint32_t tint);
void ui_sprite_rimmed (int sprite_id, int x, int y, int px);          /* 4-copy halo    */
void ui_sprite_silhouette(int sprite_id, int x, int y, int px, uint32_t c);

/* --- text --------------------------------------------------------------- */
/* NOTE: `f` is an ENUM, never a runtime pixel size. The vita2d backend maps it
 * to one of six preloaded sizes. Passing an interpolated size would allocate a
 * new glyph-atlas row every frame and silently exhaust the 512x512 atlas.     */
void ui_text          (ds_font_t f, int x, int y, uint32_t c, const char *s);
void ui_textf         (ds_font_t f, int x, int y, uint32_t c, const char *fmt, ...);
void ui_text_center   (ds_font_t f, int cx, int y, uint32_t c, const char *s);
void ui_text_right    (ds_font_t f, int rx, int y, uint32_t c, const char *s);
/* truncates with U+2026 to fit max_w; returns 1 if it truncated */
int  ui_text_ellipsis (ds_font_t f, int x, int y, int max_w, uint32_t c, const char *s);
int  ui_text_w        (ds_font_t f, const char *s);
int  ui_text_h        (ds_font_t f);
/* PromptFont handle; same enum sizes, separate atlas (the two TTFs cannot be
 * merged: fontTools Merger raises AssertionError on the GDEF/GPOS/GSUB mismatch) */
void ui_glyph         (ds_font_t f, int x, int y, uint32_t c, const char *utf8);
int  ui_glyph_w       (ds_font_t f);   /* always == DS_FONT_PX[f]; advance is 1.0em */

#endif /* UI_BACKEND_H */
```

### 9.2 Layer 1 — animation & focus helpers

```c
/* ui/render/ui_anim.h */
typedef enum { DS_EASE_LINEAR, DS_EASE_OUT_CUBIC, DS_EASE_IN_QUAD,
               DS_EASE_OUT_QUART, DS_EASE_IN_OUT_CUBIC, DS_EASE_OUT_BACK } ds_ease_t;

float ds_ease        (ds_ease_t e, float t);          /* t in [0,1] */

typedef struct { int start_frame, duration; float from, to; ds_ease_t ease; } ds_tween_t;
void  ds_tween_start (ds_tween_t *tw, float from, float to, int frames, ds_ease_t e);
float ds_tween_value (const ds_tween_t *tw);          /* clamped, safe after end */
int   ds_tween_done  (const ds_tween_t *tw);
int   ds_frames      (int nominal);                   /* -> DS_FR_REDUCED if reduce-motion */

uint32_t ds_lerp_color(uint32_t a, uint32_t b, float t);

/* Focus ring that animates between rects. One per screen. */
typedef struct { ui_rect_t cur, from, to; ds_tween_t tw; int visible; } ds_focus_t;
void ds_focus_move   (ds_focus_t *f, ui_rect_t target);
void ds_focus_draw   (const ds_focus_t *f, uint32_t c);
```

### 9.3 Layer 2 — atoms

```c
/* ui/render/widgets.h */

/* --- text/label atoms --------------------------------------------------- */
void ds_widget_section_header(int x, int y, int w, const char *caps_label);
void ds_widget_divider       (int x, int y, int w);

/* --- chip / pill -------------------------------------------------------- */
/* h=28, r-pill, 14px CAPS, 12px h-padding. fill = c@18%, border 1px c@55%,
 * text = c. Returns the drawn width so callers can chain chips.            */
int  ds_widget_chip          (int x, int y, const char *caps_label, uint32_t c);
int  ds_widget_stage_chip    (int x, int y, ds_stage_t stage);
int  ds_widget_type_chip     (int x, int y, ds_type_t type);

/* --- button ------------------------------------------------------------- */
typedef enum { DS_BTN_PRIMARY, DS_BTN_SECONDARY, DS_BTN_GHOST } ds_btn_style_t;
void ds_widget_button        (int x, int y, int w, const char *label,
                              ds_btn_style_t style, int focused, int pressed_frames);

/* --- toggle / stepper --------------------------------------------------- */
void ds_widget_toggle        (int x, int y, int on, int focused);   /* 64x32 r-pill */
void ds_widget_stepper       (int x, int y, int w, const char *value, int focused);

/* --- progress ----------------------------------------------------------- */
/* Generic track+fill, r-pill. `notch_permille` < 0 to omit the requirement pip. */
void ds_widget_progress      (int x, int y, int w, int h, int permille,
                              uint32_t track, uint32_t fill_a, uint32_t fill_b,
                              int notch_permille);

/* --- LCD plate ---------------------------------------------------------- */
/* Pale back-lit Digivice plate: fill + 1px top-left bevel + scanlines every 3px.
 * MANDATORY behind any sprite >= 4x; the darkest sprite pixel is 1.02:1 on
 * DS_BG_BASE, i.e. invisible. Use ui_sprite_rimmed() where a plate won't fit. */
void ds_widget_lcd_plate     (int x, int y, int w, int h, uint32_t face);

/* --- scrollbar ---------------------------------------------------------- */
void ds_widget_scrollbar     (int y, int h, int first_visible, int visible,
                              int total, int idle_frames);
```

### 9.4 Layer 3 — molecules

```c
/* --- stat bar (288x40) --------------------------------------------------- */
/* Label 14px CAPS in `cat` at (x, y+2); value 18px right-aligned at (x+288, y+2);
 * track 288x10 at (x, y+26) r-pill in DS_SURFACE_3; fill is a horizontal
 * gradient cat -> cat lightened 18%, width = 288 * clamp(value/cap,0,1);
 * 2px DS_TEXT_HI notch at 288*req/cap with a 6px triangular pip above.
 * value > cap => clamp the fill and draw a 2px DS_WARN chevron at the right cap. */
void ds_widget_stat_bar      (int x, int y, ds_stat_t stat, int32_t value,
                              int32_t requirement, uint32_t cat, int focused,
                              const ds_tween_t *fill_tw);

/* --- requirement row (896x48) ------------------------------------------- */
/* Renders ALL FOUR redundant encodings. Do not simplify this — the measured
 * greyscale delta between DS_OK and DS_BAD is 17 (need >=25), and 4 for the
 * Okabe-Ito pair, so colour alone is never sufficient:
 *   1. category rail       (4px, left edge, r-sm)
 *   2. SHAPE-distinct glyph (check / triangle / ballot-X on filled/ringed disc)
 *   3. bar fill length
 *   4. signed numeric delta chip
 * `lost` (care mistakes already exceeded, unrecoverable) strikes the label
 * through and forces the DS_BAD state regardless of the numbers.             */
void ds_widget_req_row       (int x, int y, const ds_check_t *chk,
                              int focused, int lost);
int  ds_widget_req_row_h     (void);   /* == DS_REQROW_H */

/* --- group header (the 3-of-4 rule) ------------------------------------- */
void ds_widget_group_header  (int x, int y, int w, ds_group_t g,
                              int met, int used);
void ds_widget_group_rollup  (int x, int y, int w, int groups_met /*of 4*/);

/* --- digimon card (116x116) --------------------------------------------- */
typedef enum { DS_CARD_NORMAL, DS_CARD_OWNED, DS_CARD_LOCKED } ds_card_state_t;
void ds_widget_card          (int x, int y, ds_species_id_t id,
                              ds_card_state_t st, int focused, float scale);

/* --- header strip used by S3 and the dex modal -------------------------- */
void ds_widget_target_header (int x, int y, int w, ds_species_id_t id,
                              int met, int total);

/* --- tab bar ------------------------------------------------------------ */
/* The 3px underline SLIDES between tabs over DS_FR_TAB_SWITCH frames.
 * Do NOT cross-fade it — the sliding rule is the single most "Vita" motion
 * in the entire application. */
void ds_widget_tab_bar       (const char *const *labels, int count,
                              int active, const ds_tween_t *slide_tw);

/* --- button-hint footer ------------------------------------------------- */
typedef struct { const char *glyph; const char *label; } ds_hint_t;
void ds_widget_footer        (const ds_hint_t *left, int n_left,
                              const ds_hint_t *right, int n_right);

/* --- toast (400x64, max one visible; queue the rest) -------------------- */
typedef enum { DS_TOAST_INFO, DS_TOAST_OK, DS_TOAST_WARN, DS_TOAST_BAD } ds_toast_kind_t;
void ds_toast_push           (ds_toast_kind_t k, const char *msg);
void ds_toast_tick_and_draw  (void);

/* --- modal (max 640x384, e2 + full-screen scrim) ------------------------ */
void ds_widget_modal_begin   (int w, int h, const char *title,
                              const ds_tween_t *enter_tw);
void ds_widget_modal_end     (void);
```

### 9.5 Layer 4 — the reel (S2 only)

```c
/* ui/render/ds_reel.h
 * Three reels, staggered stops, centre-last. Total 142 frames.
 * The state machine is the only genuinely fiddly component in the app; keep
 * it entirely frame-indexed so it is deterministic and unit-testable on host. */
typedef enum { DS_REEL_IDLE, DS_REEL_SPINUP, DS_REEL_CRUISE,
               DS_REEL_DECEL, DS_REEL_SETTLE, DS_REEL_REVEAL, DS_REEL_DONE } ds_reel_phase_t;

typedef struct {
    ds_reel_phase_t phase;
    int             frame;            /* 0..DS_REEL_REVEAL_END           */
    float           offset[3];        /* px scroll per reel              */
    int             locked[3];        /* per-reel lock flags             */
    ds_species_id_t result;
    int             reveal_chars;     /* typewriter cursor               */
} ds_reel_t;

void ds_reel_start   (ds_reel_t *r, ds_species_id_t result);
void ds_reel_skip    (ds_reel_t *r);              /* jumps to DS_REEL_SKIP_TO */
void ds_reel_tick    (ds_reel_t *r);              /* advance one vsync         */
void ds_reel_draw    (const ds_reel_t *r, const ds_species_id_t *strip, int strip_len);
int  ds_reel_busy    (const ds_reel_t *r);        /* suppress L1/R1 while true */

/* internals worth exposing for the strip vignettes and blur */
void ds_reel_draw_vignette(int x, int y, int w, int h);   /* 40px top+bottom grads */
void ds_reel_draw_payline (int y, uint8_t glow_intensity);
```

### 9.6 Layer 5 — overlay-only composites

```c
/* plugin/render/ovl.h — these exist ONLY in the plugin target. */

typedef enum { DS_OVL_HIDDEN, DS_OVL_STRIP, DS_OVL_PANEL } ds_ovl_mode_t;
typedef enum { DS_OVL_ANCHOR_PILLAR_R, DS_OVL_ANCHOR_PILLAR_L,
               DS_OVL_ANCHOR_TOP } ds_ovl_anchor_t;

void ds_ovl_strip_draw  (const ds_save_t *s, const ds_eval_t *target,
                         ds_ovl_anchor_t anchor);
void ds_ovl_panel_draw  (const ds_save_t *s, const ds_eval_t *target,
                         int scroll_row, const ds_tween_t *enter_tw);

/* The blit itself. Compose into `scratch` off the display hook (worker thread),
 * then copy H scanlines inside the hook. Never per-pixel alpha-blend a
 * full-screen panel in the display hook — that steals frametime from DW1.
 * Dirty-rect: only `dirty` is copied.                                        */
typedef struct {
    uint32_t *pixels; int w, h, pitch;   /* pitch in PIXELS, matching SceDisplayFrameBuf */
    ui_rect_t dirty; int dirty_valid;
} ds_canvas_t;

int  ds_canvas_init     (ds_canvas_t *c, int w, int h);
void ds_canvas_mark     (ds_canvas_t *c, int x, int y, int w, int h);
void ds_canvas_blit     (const ds_canvas_t *c, void *fb_base, int fb_pitch,
                         int dst_x, int dst_y);

/* Heartbeat the app reads back to diagnose "Graphics Filtering = Original".
 * The overlay cannot report its own invisibility; the app must. */
void ds_ovl_heartbeat   (uint32_t frames_seen);   /* -> ur0:data/digislots/overlay.log */
```

### 9.7 Build order for §9

`ui_backend` (vita2d impl) → `ui_anim` → atoms (§9.3) → `ds_widget_stat_bar` → `ds_widget_req_row` → `ds_widget_tab_bar` + `ds_widget_footer` → `ds_widget_card` → `ds_toast` + `ds_widget_modal` → `ds_reel` → `ui_backend` (blit impl) → overlay composites.

The first six items alone are enough to ship S1 and S3 — which is exactly the M3 milestone below.

---

## 10. UI implementation plan

Sequenced so the app **is never broken mid-way**. Each milestone ends at a state you could tag and ship. The `Overlay?` column marks which milestones are independent of all plugin work — **M0 through M7 land entirely without touching taiHEN**, which matters because the overlay's feasibility has a genuine unresolved risk (§10.2).

| # | Milestone | Ends at | Overlay-coupled? |
|---|---|---|---|
| **M0** | **Unbreak & de-risk** | Text UI still works, byte-identical | **No** |
| **M1** | **Const-table + core extraction** | Text UI still works, now on `digislots_core` | **No** |
| **M2** | **Backend seam behind the text UI** | Text UI still works, drawn by vita2d | **No** |
| **M3** | **Status + Requirements on the real design** | Two real screens, text UI gone | **No** |
| **M4** | **Chrome: tabs, footer, toasts, settings** | Navigable five-tab app | **No** |
| **M5** | **The reel** | Signature screen lands | **No** |
| **M6** | **Dex browser + modal** | Feature-complete standalone | **No** |
| **M7** | **Accessibility + polish pass** | v1.0 standalone | **No** |
| **M8** | **Overlay spike (red square)** | Go / no-go decision | **Yes** |
| **M9** | **HUD strip** | Read-only in-game glance | **Yes** |
| **M10** | **Full panel** | Goal #1 complete | **Yes** |

### M0 — Unbreak & de-risk *(½ day, no visible change)*

Everything here is a prerequisite for compiling at all on a current toolchain, and none of it touches the UI.

1. `cmake_minimum_required(VERSION 3.16)` — `2.8` is a **hard error** under CMake ≥ 4.0 **[V]**.
2. CI: `ubuntu-20.04` → `ubuntu-24.04`, `actions/checkout@v3` → `@v4`, `upload-artifact@v3` → `@v4`, `softprops/action-gh-release@v1` → `@v2`. All three v3 actions and the 20.04 runner are retired **[L]**.
3. `git rm --cached` the six committed build artifacts; extend `.gitignore` **[V]**.
4. Add a `LICENSE` to this repo. This is not bureaucracy — §6.3 shows the plugin's reference implementations are either unlicensed (`PSVshell`, `VGi`: `gh api .../license` → 404 **[V]**) or GPL-3.0 (`reVita`, `VitaGrafix`, `PSV-VSH-Menu` **[V]**). Decide now whether DigiSlots is GPLv3 (and may derive from reVita) or permissive (and must write the blitter clean-room). **Recommendation: permissive + clean-room blitter.** It is ~250 LOC against a documented struct.
5. Delete `src/resources/image.png` (63,164 B, unreferenced **[V]**). Remove the duplicated dead `file(COPY ...)` block.
6. Add `-Wall -Wextra -Wconversion -Wvla -Wformat=2` to the app target. `-Wvla` and `-Wformat=2` each catch a live bug today.

**Exit test:** `cmake -B build . && make -C build` produces a VPK that behaves exactly as before.

### M1 — Const table + core extraction *(2 days, no visible change)*

The single highest-priority fix in the whole project ships here, and it is a memory-safety bug, not a UI one.

1. **Kill the dangling evolution pointers.** `initialiseDigimon()` assigns the addresses of *function-local* arrays (`Digimon* botamonEvolutions[2] = {...}`) into `digimon->evolutionPath` **[V]** (`src/models/digimon.c:427-669`). All 59 arrays die when that function returns; every later `getRandomEvolution()` / `getEvolutionNameStrings()` reads freed stack. **Fix:** make the whole table `static const` file-scope data with `const ds_species_id_t*` arrays, and delete `initialiseDigimon()` entirely. The table becomes `.rodata`, costs zero init, is const-correct, and is safe to share with a `-nostdlib` plugin.
2. Move `models/`, `offsets.c`, `save_mappings/` into `core/` behind `ds_types.h` / `ds_save.h` / `ds_eval.h`. Sever `#include "../screen.h"` from `digimon.c` — that one line is the entire core/UI coupling edge **[V]**.
3. Delete `printDigimonStats()`, `printEvolutionRequirements()`, `getEvolutionNameStrings()` from the core; replace with `ds_evaluate()` returning `ds_check_t[]` + `ds_eval_t`. Core becomes allocation-free (today `getEvolutionNameStrings` mallocs and leaks on every roll, and `getRandomEvolution` declares a **zero-length VLA** for Etemon/Vademon **[V]**).
4. Add the **host** CI job: plain gcc, `-fsanitize=address,undefined`, `ctest`, a checked-in sample `.VMP` fixture, and a `tests/data/requirements.csv` extracted from SydMontague's guide that `test_table.c` asserts against — because the requirement rows for **Agumon, Airdramon, Angemon, Andromon and Bakemon are currently rotated among each other** **[V]** (the Vita port was forked from web v0.1.2 and never rebased past the v0.1.3 fix). Fixing them without a regression test guarantees they drift again.
5. Add `int spriteIndex` to the species struct; generate `ds_sprite_ids.h`.

**Exit test:** the text UI is untouched and still prints; the host job is green; ASan is clean.

> M1 is where the §8.3 semantics bugs get fixed in the *core*: `>` → `>=`, weight as `±5` range, `minCare`/`minBattles` honoured, `techs` populated from `popcount(mastered_moves)`, happiness decoded as **signed int16** (it is currently a single unsigned byte, so −50 reads as 206 and satisfies every happiness requirement **[V]**), and the 4-group / 3-of-4 eligibility rule implemented. Do this *before* any UI work — otherwise you build the UI against the wrong data model (11 flat rows instead of 4 groups) and rebuild it.

### M2 — Backend seam behind the text UI *(2 days, near-invisible change)*

This is the milestone that makes "never broken" possible.

1. Run `tools/mkassets.py`; produce the atlas, font subsets and baked glyph headers.
2. Implement `ui_backend_vita2d.c` (§9.1) and `ui_anim` (§9.2).
3. **Reimplement `screen.c`'s `print()` / `clearScreen()` on top of the backend** — a monospaced scrolling text buffer drawn with `ui_text(DS_FONT_BODY, ...)` on the `bg-raise → bg-base` gradient. Delete `common/debugScreen.c` and its static 960×544×4 (~2 MB) framebuffer.
4. Verify the six-size atlas actually fits at runtime (instrument `atlas_add_glyph` failure by rendering the full charset at startup in a debug build — remember it fails **silently** **[V]**).

**Exit test:** the app looks ~identical to today but is running entirely on vita2d, at 60 fps, with the real fonts. This de-risks every subsequent milestone: if the atlas or the fonts are wrong, you find out here, with nothing else in flight.

### M3 — Status + Requirements on the real design *(3 days — first real visual payoff)*

1. Build atoms (§9.3) → `ds_widget_stat_bar` → `ds_widget_req_row` → `ds_widget_lcd_plate` → `ds_widget_card`.
2. Ship **S1 STATUS** (§8.1) and **S3 REQUIREMENTS** (§8.3). These two are direct re-skins of the deleted `printDigimonStats()` / `printEvolutionRequirements()`, so they prove the whole kit against known-good data.
3. Sprites: `POINT` filtering + integer scales + LCD plate. Verify the near-black-silhouette fix on real hardware — this is the one design decision that is impossible to evaluate from a desktop screenshot.
4. Wire the ✕-toasts using the web app's tooltip copy verbatim.

**Exit test:** `print()` still exists and still works for any screen not yet ported; S1 and S3 are the real design. **The app is shippable here.**

### M4 — Chrome *(2 days)*

Tab bar with the sliding underline, button-hint footer, toast queue, modal, **S5 SETTINGS** (§8.5) incl. persistence to `ux0:data/digislots/settings.bin` (versioned struct, magic + version prefix), and the Overlay Setup modal copy. Add the L1/R1 tab cycling and the touch layer (`sceTouchGetPanelInfo()`-scaled).

**Exit test:** a fully navigable five-tab app, two tabs of which are placeholders.

### M5 — The reel *(1 day, highest risk-per-line in the standalone half)*

`ds_reel` (§9.5) + **S2 ROLL** (§8.2). The risk is entirely in the frame-timing state machine, not in anything vita2d can't do — so keep `ds_reel_tick()` pure and frame-indexed and unit-test it on the host job. Implement `ds_reel_skip()` in the same sitting as `ds_reel_start()`; a 2.4 s animation you can't skip is worse than no animation.

Reel motion blur is the one place where 3× overdraw is justified. If it costs frames on hardware, drop to 2 ghosts before you shorten the animation.

**Exit test:** the app finally does the thing it is named after.

### M6 — Dex browser *(2 days)*

**S4 EVOLUTIONS** (§8.4) + the detail modal, with the 9-target evolution strip and the silhouette/spoiler system. This is the biggest new surface and the one with the least existing logic behind it, which is why it is last.

**Exit test:** feature parity with digislots.fippi.io, plus everything the web app structurally cannot do (real save data, ranked progress, prediction).

### M7 — Accessibility & polish *(2 days)* → **v1.0 standalone**

Colour-blind palette swap, Reduce-motion clamp, Large-text **swap** (not addition — atlas budget), high-contrast mode, error states (no Adrenaline install / no DW1 save / bad checksum / unknown species byte) replacing today's silent-garbage path **[V]** (`readFileToBuffer`'s return is discarded and an uninitialised 8 KB stack buffer gets parsed), a real quit affordance replacing `sceKernelDelayThread(~0)`, and the LiveArea re-skin (`icon0.png` 128×128, `bg.png` 840×500, `startup.png` 280×158 — all already at correct spec **[V]**, so it is a pure art swap with no `template.xml` change).

**→ Everything above ships without a single line of taiHEN code. If the overlay never happens, the user still gets goals #2 and #3 in full.**

### M8 — Overlay spike *(1 day — GO/NO-GO GATE)*

**Do not write any overlay UI before this passes.** A ~50-line `.suprx` under `*PSPEMUCFW`:

1. `taiHookFunctionImport(&ref, "ScePspemu", 0x4FAACD11, 0x8C36B628, hook)` **[V]**.
2. In the hook, if `pParam && pParam->base`, paint a 100×100 red square at (100,100) and `TAI_CONTINUE`.
3. Log frame count + `pParam->width/height/pitch/pixelformat` to `ur0:data/digislots/overlay.log` on a timer.
4. Run **twice**: once with Adrenaline Graphics Filtering = *Original*, once with anything else.

**Decision matrix:**

| Result | Action |
|---|---|
| Square visible with filtering ≠ Original | **GO.** Proceed to M9. Ship the Setup modal that requires the setting. |
| Square visible in *both* modes | **GO+.** The Original-mode caveat is empirically wrong for POPS; drop it from onboarding. (Note the mechanism was only ever **[L]**-graded, so this is possible.) |
| Square never visible | **Try Plan B:** the kernel `.skprx` hooking `ksceDisplaySetFrameBufInternal` **[V]**. If that also fails, **stop** — ship M0–M7 and mark goal #1 as blocked with the evidence. Do **not** burn weeks on `SceSharedFb` (**[S]**, zero precedent, DolceSDK) or a PSP-side POPS PRX (a *modal pause screen*, not an overlay **[V]**). |

Also test in this sitting: is `SCE_TOUCH_PORT_BACK` sampled inside `ScePspemu`? That single boolean decides Tier 1 vs Tier 2 of §8.7 and it is currently **[S]**.

### M9 — HUD strip *(3 days, only if M8 is GO)*

1. Write `ui_backend_blit.c` clean-room (§6.3 licensing). Compose in a worker thread; copy scanlines in the hook; dirty-rect only. Steal the *architecture* from PSVshell's `psvs_gui_cpy()` and reVita's `renderer.c`; copy no code.
2. Bake the sprite blob + glyph atlases into `.rodata` via `ld -r -b binary`.
3. Ship **S6a** (§8.6), both anchor variants.
4. Ship the heartbeat + the app-side status readback (§8.5).

**Consumes zero inputs. Never masks the pad.** That is why it is first: it is 90 % of the everyday value at ~30 % of the risk, and it cannot break Digimon World even if it misbehaves.

### M10 — Full panel *(3 days)*

**S6b** (§8.7) + the input ladder + `sceCompatWriteSharedCtrl` masking **[V]**. Keep the vocabulary to ↑↓ / ✕ / ○ — no tabs, ever.

### 10.1 Dependency graph

```
M0 ──► M1 ──► M2 ──► M3 ──► M4 ──► M5 ──► M6 ──► M7  ═══► v1.0 STANDALONE
                       │                                    (goals #2, #3 complete)
                       └──────────────────► M8 ──► M9 ──► M10 ═══► goal #1
                          (needs the widget layer, which is
                           backend-agnostic from M3 onward)
```

M8 can be spiked **in parallel** with M3–M7 by anyone who wants to de-risk early; it shares no code with the app.

### 10.2 Standing risks

| Risk | Grade | Mitigation |
|---|---|---|
| Overlay invisible under Graphics Filtering = Original | **[V]** outcome, **[L]** mechanism | Onboarding modal + heartbeat diagnostics in the app (M9). Non-negotiable, documented, user-fixable. |
| Overlay never visible under POPS at all | **[L]** it works — reVita draws under Adrenaline; but *nobody has shipped a dynamic game-state HUD over a POPS title* | M8 gate. Standalone app is fully shippable without it. |
| Rear touchpad claimed by Adrenaline | **[S]** | Three-tier input ladder (§8.7); Tier 2 chord is the designed fallback. |
| Glyph atlas silently drops text | **[V]** | Six-size hard cap; enum-typed `ds_font_t`; startup charset probe in debug builds; Large-text is a swap not an addition. |
| Blitter licensing contamination | **[V]** | Clean-room + explicit repo LICENSE at M0. |
| Adrenaline version drift (FloW v7 vs isage v8) | **[V]** — v8.0.2 shipped 2026-07-12; plugin config format, file layout and the PNG-overlay filter all differ | Detect at runtime; target isage v8 primarily; never hardcode ScePspemu text offsets without a `module_nid` guard (Adrenaline itself branches per-firmware **[V]**). |
| CDRAM write cost | **[V]** vita2d display buffers are `USER_CDRAM_RW`; CPU writes there are uncached | Dirty-rect blit only. Never a 2 MB full-screen copy per frame. |

### 10.3 Free win available at any time, independent of everything

Adrenaline v8 already alpha-blends `<memstick>/overlays/<TITLEID>.png` — 960×544 RGBA — over the running PS1 image **[V]** (`isage user/main.c:386-428`, `user/menu.c:719-726`; documented at `???0:/pspemu/overlays/<TITLE_ID>.png`). Writing `ux0:pspemu/overlays/SLUS01032.png` is a **zero-injection, zero-code** way to prove the visual language on hardware — a static reference card in the Digivice Dusk identity. It is loaded on `APP_STARTED` and is not dynamic (and forcing a mid-game reload races a use-after-free on `overlay_texture` **[V]**, so don't), but as a design-validation step during M3 it costs an afternoon and derisks the entire overlay aesthetic before M8.
