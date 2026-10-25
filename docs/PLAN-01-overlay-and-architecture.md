# DigiSlots Vita — Implementation Plan (Technical Half)

> Scope note: this document covers feasibility, unknowns, architecture, phasing and risk. Product/UX copy, visual design tokens and screen wireframes are in the design half. Every technical claim below is tagged **[V]** verified against a named artifact, **[L]** likely (strong indirect evidence), or **[S]** speculation. Where an adversarial verifier refuted the original research I have taken the verifier's position and said so inline.

---

## 1. Feasibility verdict — the overlay

### Answer: **YES-BUT**. Confidence: **MEDIUM**.

A taiHEN plugin *can* draw a toggleable panel over Digimon World 1 running under Adrenaline/POPS, and it *can* read the live PSX RAM so the numbers are current rather than last-saved. But it is gated on a user-set Adrenaline option, no one has ever shipped this exact thing, and two of the load-bearing constants are unverified on hardware. **The go/no-go decision must come from the spikes in §2, not from this document.**

#### What is proven

| Claim | Grade | Evidence |
|---|---|---|
| Overlays are drawn by hooking the display-flip and CPU-blitting into the caller's framebuffer; no library does this for you | **[V]** | `Electry/PSVshell` src/main.c:307 `taiHookFunctionExportForKernel(KERNEL_PID,…,"SceDisplay",0x9FED47AC,0x16466675,…)`; `MERLev/reVita` src/main.c:735 same; `Rinnegatamante/vitaRescale` renderer.c (user-mode variant) |
| User-mode seam: `sceDisplaySetFrameBuf` NID `0x7A410B64` in library **SceDisplayUser** libnid `0x4FAACD11` | **[V]** | vita-headers `db/360/SceDriverUser.yml:140-151`. **Correction:** original research wrote libnid `0x5ED8F994` — that is the *other* `SceDisplay` library whose export is `_sceDisplaySetFrameBuf 0xF51523CB`. Both verifiers independently caught this; the hook would silently fail to resolve. |
| The pspemu process runs under titleid `PSPEMUCFW`, and taiHEN loads config.txt sections by exactly that titleid | **[V]** | `adrenaline_vita.h:10 / adrenaline_compat.h:31 ADRENALINE_TITLEID "PSPEMUCFW"`; taiHEN `hen.c:375-378` `ksceKernelGetProcessTitleId(pid,titleid,32); taiLoadPluginsForTitleForKernel(pid,titleid,0)` with no pspemu exclusion |
| The whole 64 MB emulated-PSP RAM is a flat, dereferenceable user pointer inside ScePspemu: `CONVERT_ADDRESS(a) = 0x68000000 + (a & 0x0FFFFFFF)` | **[V]** | `adrenaline_compat.h:188`, republished in `pspdev/psp-cfw-sdk include/systemctrl_adrenaline.h`. Decisive corroboration: Adrenaline's own `SaveState()` (user/main.c:145) LZ4-compresses straight through `ScePspemuConvertAddress(0x88000000,…,64MB)` |
| A user plugin inside `PSPEMUCFW` has unrestricted ux0/ur0 file IO | **[V]** | `adrenaline_user.suprx` does exactly this: flashfs.c:219, utils.c:51, all of msfs.c. **Correction:** the original "user plugins need ioPlus/kuio" claim was refuted — `Electry/VitaGrafix` src/io.c:100 opens `ux0:data/VitaGrafix/config.txt` with plain `sceIoOpen`; ioPlus is only an error hint |
| Adrenaline can freeze POPS on demand: `ScePspemuPausePops(1/0)` | **[V]** | Adrenaline user/menu.c:179/193; per-firmware offsets in isage user/main.c:131-152 |
| PSX main RAM is at emulated-PSP `0x09800000` → Vita VA `0x71800000` | **[L]** | `raing3/tempar` src/objects/config.c:90 `cfg.address_start = 0x09800000` for `PSP_INIT_KEYCONFIG_POPS`. Adrenaline runs the same `pops` PRX (`cef/popcorn/main.c:145`). Neighbours corroborated by `ARK-4 core/compat/vitapops/popsdisplay.c:16-17` (VRAM 0x490C0000, cfg 0x49FE0000). **Not confirmed on Vita POPS specifically — spike S3.** |
| DW1 partner struct at PSX RAM `+0x1557A8`; repo's own `resources/DMW_New.CT` gives `PSXBase+0x138460` (== save 0x03E0) and `+0x1557E0` (== save 0x0470) | **[V]** | `SydMontague/DigimonWorldAPI`: 0xBAD148 − 0xA579A0 = 0x1557A8; GameShark `801557F0` infinite HP for SLUS-01032 |

#### What is *not* proven, and is the reason confidence is only MEDIUM

| Blocker | Grade | Evidence |
|---|---|---|
| **The overlay is invisible when Adrenaline "Graphics Filtering" = "Original".** With that setting Adrenaline's vita2d present loop early-`continue`s and never issues a frame; separately, POPS calls `sceDisplaySetFrameBufForCompat` with `pParam == NULL` and Adrenaline only substitutes a real A8B8G8R8 buffer when `graphics_filtering != 0` — so there is no user-visible framebuffer to write in Original mode. | **[V]** outcome, **[L]** mechanism | isage `user/menu.c:657` (byte-identical to FloW `menu.c:549`): `if (sceCommonDialogIsRunning() \|\| (config.graphics_filtering == 0 && menu_open == 0 && draw_native == 0)) { sceDisplayWaitVblankStart(); continue; }`. `user/pops.c:172-186` substitution gate. Independently documented by `MERLev/reVita README.md:74` and repeated by the *live* fork `JSilva6/reVita-extended` (pushed 2026-07-22). The original research asserted "no `sceDisplaySetFrameBuf` ever happens" as VERIFIED-by-mechanism; a verifier showed the compat setter still fires. **I take the verifier's position: outcome verified, mechanism only empirical.** |
| **Nobody has shipped a dynamic game-state HUD over a POPS title.** reVita/reVita-extended draw a remap UI over Adrenaline with the documented caveat; `Electry/PSVshell` explicitly refuses (`src/main.c:68 if (g_is_in_pspemu) goto DISPLAY_HOOK_RET;`, commit `bcd0885c "Blacklist Adrenaline"`); `psonescrot` runs inside pspemu but draws nothing. | **[V]** | as cited |
| **`*PSPEMUCFW` has zero observed uses in the wild.** GitHub code search returns none. The shipped precedent for a pspemu user plugin is `*ALL` + a `ScePspemu`-presence guard (`LiEnby/NoPspEmuDrm` installs as `*ALL ur0:/tai/NoPspEmuDrm_user.suprx`). ABM-generated bubbles report a *different* titleid (reVita models this as `APP_PSPEMU_ABM`). | **[V]** | as cited |
| **A second `vita2d`/GXM context in the pspemu process is impossible** — `sceGxmInitialize` is once-per-process and `adrenaline_user` already owns it. All drawing must be CPU blitting. | **[V]** | `psp2/gxm.h:21 SCE_GXM_ERROR_ALREADY_INITIALIZED 0x805B0001`; Adrenaline user/menu.c:389 `vita2d_init()`. *Nuance from a verifier:* GXM objects *can* be created in a foreign process by borrowing the host context (`SonicMastr/Sonic-Overlay` patches.c:250-300), and `SceSharedFb` (`psp2/sharedfb.h`, SceAppMgr.yml:260-275) is a real third path — but neither has any shipping precedent for overlaying a game and both are out of scope. |
| **CPU blitting into vita2d's buffers is expensive.** vita2d allocates its display buffers `SCE_KERNEL_MEMBLOCK_TYPE_USER_CDRAM_RW` (libvita2d vita2d.c:326-335); CDRAM is uncached for the CPU. A 960×544×4 ≈ 2 MB per-frame copy is a framerate hazard. | **[V]** | as cited |
| **Kernel-side SceCtrl hooks cannot fully gate input.** Adrenaline reads the pad through its own `kuCtrlPeekBufferPositive` syscall (kernel/exports.yml `AdrenalineKubridge`), which bypasses the eight user `SceCtrl` exports PSVshell hooks. The real guest-input gate is `sceCompatWriteSharedCtrl` (ScePspemu import, SceCompat libnid `0x0F35909D`, NID `0x2306FFED`) — reachable **only from a user plugin inside ScePspemu**. | **[V]** | Adrenaline user/main.c:403-424 zeroes `pad_data->Buttons/Lx/Ly/Rx/Ry` there when `menu_open` |
| **Adrenaline is a moving target and the wrong repo was researched.** `TheOfficialFloW/Adrenaline` is dead (last release v7, 2020-09-22). The live one is `isage/Adrenaline` v8.0.2 (2026-07-12, HEAD 2026-07-29). Core conclusions survive re-verification against isage, but PSP-side plugin config format, file layout and line numbers do not. | **[V]** | GitHub releases API |

#### Ranking the "seamless" tiers

Score = user value ÷ (risk × effort), all 1–5, higher value is better, lower risk/effort is better.

| # | Tier | What the user gets | Value | Risk | Effort | V/(R×E) | Verdict |
|---|---|---|---|---|---|---|---|
| T0 | Static PNG overlay `ux0:pspemu/overlays/SLUS01032.png` (isage v8 only) | A fixed reference card blended over gameplay. Zero code. Reloads only on app start; forcing a mid-game reload is a **use-after-free** (isage frees `overlay_texture` from the Kermit thread while AdrenalineDraw dereferences it, no mutex) **[V]** | 1 | 1 | 1 | 1.00 | **Prototype only** — use it in week 1 to validate the visual language and prove the compositing path exists |
| T1 | Standalone VPK, reads `SCEVMC0.VMP` (today, polished) | Correct data, good UI, but requires leaving the game *and* having saved in-game | 2 | 1 | 2 | 1.00 | **Fallback / always-shipped baseline** |
| T2 | User `.suprx` overlay reading the `.VMP` | In-game panel, but data is only as fresh as the last in-game save | 3 | 3 | 3 | 0.33 | Stepping stone to T3, same binary |
| **T3** | **User `.suprx` overlay reading live PSX RAM** | **In-game panel with current, unsaved stats — the actual goal** | **5** | **4** | **4** | **0.31** | **PRIMARY** (T2 → T3 is an incremental data-source swap, not a rewrite) |
| T4 | PSP-side `.prx` in the POPS runlevel | Filter-agnostic, but it is a *pause-the-game modal*, not an overlay: visibility comes from suspending the `popsmain` thread → `PAUSE_POPS` → 480×272 PSP framebuffer **[V]** (isage `cef/core/pentazemin/src/modpatches.c:50-62`). Second MIPS toolchain (pspsdk). `raing3/tempar` docs/TODO.md:6 says Vita compat is *unverified by its own author* | 3 | 4 | 5 | 0.15 | **Rejected** |
| T5 | WiFi companion (kernel plugin + TCP → digislots.fippi.io) | Second-screen stats; needs a phone/PC and network up | 2 | 2 | 2 | 0.50 | **Optional bonus**, ~60 LOC of net code (`Electry/VitaPresence` plugin/src/main.c:140-186 is a complete template) |
| T6 | Suspend-and-resume via Adrenaline savestates | Community reports name **Digimon World specifically** for post-savestate audio corruption, and in-game saves then land inside the savestate's card image rather than `SCEVMC0.VMP` — this can silently diverge the user's real save | 2 | 5 | 2 | 0.20 | **Rejected as a product path.** Useful *offline on PC* for offset discovery (spike S3) |
| T7 | SceShell `ScePaf` plugin / `SceSharedFb` | Genuinely first-party-looking, GPU-composited | 4 | 5 | 5 | 0.16 | **Rejected.** `GrapheneCt/PSVshellPlus` needs DolceSDK + MSVC + a hand-authored RCO; `SceSharedFb` needs a system-budget process and has no precedent for overlaying a game |

**Primary path: T3, delivered as T2 → T3.** One user-mode `digislots.suprx` injected into the pspemu process, hooking `AdrenalineUser`'s import of `sceDisplaySetFrameBuf` (`0x4FAACD11`/`0x7A410B64`), CPU dirty-rect blitting a small HUD panel, gating guest input via `sceCompatWriteSharedCtrl`, reading the `.VMP` first (T2) and swapping to `CONVERT_ADDRESS(0x09800000 + off)` once spike S3 lands (T3).

**Why user-mode and not a kernel `.skprx`:**
1. `ScePspemuConvertAddress` and `ScePspemuPausePops` are *in-process user functions resolved by hardcoded text offsets* (isage user/main.c:123, 131-152). A kernel module cannot call them.
2. `sceCompatWriteSharedCtrl` — the only real guest-input gate — is a ScePspemu import, hookable only in-process.
3. File IO in `PSPEMUCFW` is unrestricted (refutes the original "go kernel to avoid ioPlus" argument).
4. `pParam->base` is a plain pointer in-process — no `ksceKernelMemcpyKernelToUser` per scanline, ~3× cheaper.
5. User-space hooks are auto-cleaned on process exit; kernel patches are not. Lower brick risk (§5).

**Fallback: T1.** If spike S1 or S2 fails, ship the polished standalone app plus a T0 static evolution-chart PNG for v8 users. Phases 0–4 of §4 are written so that this fallback costs *nothing* — they are the same work either way.

**Non-negotiable user-facing constraint either way:** the overlay README must say *"set Adrenaline Settings → Graphics Filtering to anything except Original"*, and the plugin must detect "no frames seen in N seconds while pspemu is foreground" and surface that instruction rather than failing silently. This is exactly what reVita does.

---

## 2. Unknowns to resolve first — the RESEARCH plan

Ordered so the riskiest assumption dies first. Do not write UI code until S1–S3 are green. Total ≈ 3–5 days of on-device work.

Standing setup for all spikes: a `taiHookFunctionImport` log line written with `sceIoOpen("ux0:data/digislots/spike.log", SCE_O_WRONLY|SCE_O_CREAT|SCE_O_APPEND, 0777)` + `sceIoWrite` — there is no debugger, so file logging is the only instrument. **Escape hatch: hold L (or L1) while launching the app; taiHEN `hen.c:368-373` skips all plugin loading.** Document that in the README before the first spike.

---

**S1 — Does a user plugin load into the pspemu process, and can it paint a pixel? (RISKIEST — kill first)**
- **Question:** Does taiHEN inject a `.suprx` into the Adrenaline process, and does a hook on `AdrenalineUser`'s `sceDisplaySetFrameBuf` import resolve and fire while Digimon World 1 is running?
- **Experiment:** ~80-line `.suprx`. `module_start` → log "loaded", then `taiGetModuleInfo(TAI_MAIN_MODULE, &info)` and log `info.name` (expect `ScePspemu`). Then `taiHookFunctionImport(&g_ref, "AdrenalineUser", 0x4FAACD11, 0x7A410B64, hook)`; **if the return is < 0, retry with the module name `"adrenaline_user"` and log which one worked** — the two verifiers disagree (`user/exports.yml` top-level key is `AdrenalineUser`; `user/CMakeLists.txt:94` names the *file* `adrenaline_user.suprx`, and taiHEN matches the runtime SceModuleInfo name). In the hook: log `pParam->width/height/pitch/pixelformat` once, memset a 100×100 red block into `pParam->base`, `TAI_CONTINUE`. Install to `ur0:tai/digislots.suprx`; register **both** ways in separate runs: (a) `*PSPEMUCFW`, (b) `*ALL` plus an in-plugin `ScePspemu` guard (`LiEnby/NoPspEmuDrm` pattern). Set Graphics Filtering to **Bilinear** (not Original). Boot Adrenaline → Digimon World 1.
- **Pass:** a solid red square appears over the running PS1 game, and the log records a sane `960×544 / pitch 960 / pixelformat 0`.
- **Fail:** no log → injection failed (try (b), then fall back to a kernel loader using the `psv-plugins-archive/psonescrot` un-inhibit trick: hook `SceKernelModulemgr` `0xEAED1616`/`0x6CED1F63` `sceKernelInhibitLoadingModule`, return 0 when the process contains a `ScePspemu` module, then `ksceKernelLoadStartModuleForPid`). Log but no square → wrong seam, go to S2.
- **Time:** 1 day. **Blocks:** everything.

---

**S2 — How hard is the "Graphics Filtering = Original" wall, and is there any way through it?**
- **Question:** (a) Confirm the square from S1 disappears in Original mode. (b) Does the *kernel* seam `ksceDisplaySetFrameBufInternal` still fire in Original mode? The mechanism is unproven either way — a verifier showed `sceDisplaySetFrameBufForCompat` *is* still invoked in Original mode (it just carries `pParam == NULL`), so the "no frames at all" story is not established at the kernel layer.
- **Experiment:** (a) toggle Adrenaline Settings → Graphics Filtering to Original, relaunch, observe. (b) Build a throwaway `digislots_spike.skprx` under `*KERNEL`: `taiHookFunctionExportForKernel(KERNEL_PID,&ref,"SceDisplay",0x9FED47AC,0x16466675,hook)`; in the hook, if `head == ksceDisplayGetPrimaryHead()` and `pParam && pParam->base`, log `index`, `width`, `pParam->base` once per second and write a red block via `ksceKernelMemcpyKernelToUser`. Copy PSVshell's guards verbatim (`if (index && ksceAppMgrIsExclusiveProcessRunning()) return;`). Note `ksceAppMgrIsExclusiveProcessRunning` is **not declared in any psp2kern header** — it exists only as NID `0xA888C2DC` in `db/360/SceAppMgr.yml:232`; forward-declare it yourself as PSVshell does (`src/main.c:15`) and link `SceAppMgrForDriver_stub`.
- **Pass (a):** square vanishes → constraint confirmed, document it and move on. **Pass (b):** if the kernel hook *does* fire in Original mode with a writable buffer, that is a significant upside — record it, but do **not** re-plan around it (PSVshell blacklisted pspemu for undocumented reasons and this is exactly the code path it refused to run).
- **Fail-safe:** if the kernel spike crashes or bootloops, hold L at launch and delete the config line.
- **Time:** 0.5 day. **Kernel risk: this is the one spike that touches kernel — do it on a Vita you can recover, with a working `ur0:tai/` backup.**

---

**S3 — Where is PSX main RAM, and is the DW1 save block mirrored contiguously?**
- **Question:** Is `CONVERT_ADDRESS(0x09800000) == 0x71800000` the base of the PS1's 2 MB, and does `RAM_addr = 0x71800000 + 0x138460 + (save_off − 0x03E0)` hold across the whole `0x03E0..0x0690` range the parser uses?
- **Experiment, in two halves.**
  - **Offline first (no device time, no plugin):** in Adrenaline, savestate DW1, change a known stat in-game (feed until weight moves), savestate again. Pull `ux0:pspemu/PSP/SAVESTATE/STATE00.BIN` / `STATE01.BIN` to a PC, parse `AdrenalineStateHeader` (`ram_part1_offset/size`, isage user/main.c:142-192), LZ4-decompress part 1, and diff. PSX RAM should be at image offset `0x09800000 − 0x08000000 = 0x01800000`. Locate the partner block by searching for the known species byte + stat values. **This answers the question with a laptop and zero crash risk.** *Do not then reload those savestates on the device* — see §5.
  - **On device:** extend the S1 plugin: on combo, `memcpy` 64 bytes from `(void*)(0x71800000 + 0x138460)` into a log line as hex. Compare against the `.VMP` you just saved.
- **Pass:** the RAM bytes match the parsed save block field-for-field for the whole range.
- **Partial pass:** base correct but layout diverges → the core needs a *second* offset table (`ds_ram_offsets.h`), which `resources/DMW_New.CT` already supplies (`PSXBase+0x1384B6` = age-in-hours-for-evolve, `+0x1557E0` = Offense, `+0x154F9A` = *Evolve Target ID*).
- **Fail:** ship T2 (`.VMP` only) and treat T3 as a stretch goal.
- **Time:** 1 day (mostly the offline diff). **Blocks:** T3 only, not T2.

---

**S4 — Can the overlay take input without breaking the game, and which combo is safe?**
- **Question:** Which chord can DW1 never produce, and does `sceCompatWriteSharedCtrl` actually gate the guest?
- **Experiment:** extend the plugin — `sceCtrlPeekBufferPositive(0,&pad,1)` in the display hook (peek is non-destructive) to detect a chord; `taiHookFunctionImport(&ref,"ScePspemu",0x0F35909D,0x2306FFED,hook)` and, while `g_open`, zero `pad_data->Buttons/Lx/Ly/Rx/Ry` exactly as Adrenaline does. Test candidate chords: `SELECT+R`, `SELECT+START`, `SELECT+R+□`. **Explicitly avoid `L+SELECT` (POPS fast-forward, isage menu.c:640-653) and PS double-click (returns to LiveArea, user/menu.c:519-523).**
- **Pass:** chord opens the panel; while open, D-pad/× move nothing in DW1; on close, input resumes immediately.
- **Also test:** rear touch (`sceTouchSetSamplingState(SCE_TOUCH_PORT_BACK,…)` + `sceTouchPeek`) — PS1 hardware has no touch, so it *should* be a free channel **[S]**, but it is unknown whether Adrenaline claims it. If free, a rear double-tap is a strictly better toggle than any chord.
- **Time:** 0.5 day.

---

**S5 — What does the blit cost?**
- **Question:** Does a per-frame CPU blit into CDRAM visibly drop DW1's framerate?
- **Experiment:** parameterise the S1 red block: 100×100, 320×200, 640×400, 960×544. Draw a frame counter in the corner (or log `sceKernelGetProcessTimeWide()` deltas between hook invocations to a file). Compare against baseline with the plugin loaded but drawing disabled.
- **Pass:** target ≤ 0.5 ms per composited frame at the chosen panel size. Expect a full-screen 2 MB copy to be far worse than that — vita2d display buffers are `USER_CDRAM_RW` and uncached for the CPU.
- **Design implication:** budget a **panel**, not a screen (PSVshell uses 308×344 ≈ 424 KB), compose off-hook in a worker thread, and copy only dirty scanlines in the hook.
- **Time:** 0.5 day.

---

**S6 — ABM bubbles and Adrenaline version matrix**
- **Question:** What titleid does the process report when DW1 is launched from an Adrenaline Bubble Manager bubble rather than the `PSPEMUCFW` bubble? Is the user on FloW v7 or isage v8?
- **Experiment:** log `taiGetModuleInfo`/`sceAppMgrAppParamGetInt`-derived identity in S1 across both bubble types. In parallel, check `ux0:app/PSPEMUCFW/` version and log the `ScePspemu` `module_nid` (0x2714F07D = 3.60, 0x3F75D4D3 = 3.65–3.70, 0xEA8C1AE2 = 3.71, 0x5459B715 = 3.72–3.74 per isage user/main.c:131-152).
- **Pass:** either `*PSPEMUCFW` covers both, or `*ALL` + a `ScePspemu` main-module guard does. Ship whichever works for both.
- **Time:** 0.5 day.

---

**S7 — Checksum and block location on real cards**
- **Question:** Does the checksum at save `0x06FC` (sum of `0x0200..0x10FF` with the checksum zeroed, byte-swapped — algorithm already written in `resources/digimon-save.hexpat:136-157`) verify on a real `SCEVMC0.VMP`, and does the title-signature scan (`hexpat:97`, `std::mem::find_sequence(0, 53 43 13 01 82 63 82 89)`) find the block regardless of which card slot the save occupies?
- **Experiment:** pure host work. Copy a real `.VMP` off the device, run it through the `dsdump` host CLI from Phase 1.
- **Pass:** checksum matches; scan finds the block at a non-`0x2080` offset in a card where DW1 is not in slot 1.
- **Time:** 2 hours. **No device needed** — do this alongside S3's offline half.

---

## 3. Target architecture

### 3.1 Repo layout

```
CMakeLists.txt              # top level; host-buildable, Vita targets guarded
cmake/
  digislots_core.cmake      # helper to instantiate the core lib under 3 flag sets
core/                       # freestanding C99. NO malloc, NO stdio, NO FILE*, NO float printf.
  include/digislots/        # ds_types.h ds_data.h ds_io.h ds_save.h ds_eval.h
                            # ds_roll.h ds_time.h ds_fmt.h ds_text.h ds_err.h
  src/                      # ds_data.c ds_data_special.c ds_data_items.c ds_data_techs.c
                            # ds_save.c ds_save_locate.c ds_eval.c ds_roll.c
                            # ds_time.c ds_fmt.c ds_text.c
app/                        # VitaSDK VPK (vita2d) — the standalone title
  main.c app.c app.h
  io_vita.c                 # sceIoOpen/sceIoPread -> ds_source_t ; settings persistence
  input_vita.c              # edge-detected sceCtrl + sceTouch
  render/                   # gfx.c theme.h scr_*.c   (design half owns these)
  assets/                   # atlas.png, 9-slices, subsetted TTFs
plugin/                     # user-mode .suprx injected into ScePspemu
  main.c                    # module_start/stop, hooks, worker thread
  pspemu.c pspemu.h         # ScePspemu resolution: text_addr, ConvertAddress, PausePops
  blit.c blit.h             # ARGB compositor + alpha blend + dirty-rect scanline copy
  font_atlas.h              # bin2c'd 8bpp glyph atlas (built by tools/mkatlas)
  sprites_bin.h             # bin2c'd raw RGBA sprite atlas
  digislots.yml             # module_start/module_stop declaration
tools/
  dsdump.c                  # host CLI: dsdump card.vmp [--json]  — test oracle
  mkatlas.c                 # host: PNGs -> raw RGBA blob + C header (build-time)
tests/
  fixtures/*.vmp            # checked-in sample cards (tamer name redacted)
  data/requirements.csv     # extracted from SydMontague's evolution guide
  test_locate.c test_parse.c test_eval.c test_roll.c test_table.c
docs/
  offsets.md                # ex resources/*.txt — promoted out of the shipped VPK
```

Deleted: `common/debugScreen.c` (once vita2d owns the screen), `src/other/snippets.c`, `src/resources/image.png` (63,164 B, 460×460, referenced nowhere — 61% of the resource dir), and the committed build artifacts (`CMakeCache.txt`, `Makefile`, `cmake_install.cmake`, `digislots_vita`, `digislots_vita.vpk`, `hello.txt`).

### 3.2 The pure-C core API

Design rules, all forced by real constraints:
- **No `malloc`, no `stdio`, no `FILE*`** — the plugin builds `-nostdlib` against `SceLibc_stub`, matching `Electry/VitaGrafix`'s CMakeLists. Permitted libc surface: `memcpy/memset/memcmp/strlen/strcmp/strncpy/snprintf`, all present in SceLibc.
- **All tables `static const`** in `.rodata`. This is not a style preference: `initialiseDigimon()` (digimon.c:427-669) currently assigns the addresses of *function-local* arrays into `digimon->evolutionPath`, so every evolution list in the program is a dangling pointer into a dead stack frame. Making the table file-scope `const` deletes both the bug and the function.
- **All output into caller-provided storage.** No returned heap strings (`getEvolutionNameStrings()` leaks on every roll today).
- **One platform seam:** `ds_source_t`.

```c
/* ===== ds_err.h ===== */
typedef enum {
    DS_OK = 0,
    DS_ERR_IO             = -1,  DS_ERR_TOO_SMALL      = -2,
    DS_ERR_NO_BLOCK       = -3,  DS_ERR_CHECKSUM       = -4,
    DS_ERR_UNKNOWN_SPECIES= -5,  DS_ERR_NO_CANDIDATES  = -6,
    DS_ERR_RANGE          = -7
} ds_err_t;
const char *ds_strerror(ds_err_t e);

/* ===== ds_types.h ===== */
typedef enum { DS_STAGE_FRESH, DS_STAGE_IN_TRAINING, DS_STAGE_ROOKIE,
               DS_STAGE_CHAMPION, DS_STAGE_ULTIMATE } ds_stage_t;
typedef enum { DS_TYPE_DATA, DS_TYPE_VACCINE, DS_TYPE_VIRUS } ds_type_t;

/* value == the byte at save 0x03B8, so species lookup is O(1) indexing,
   not the linear scan over digimonMap[61] that main.c does today */
typedef uint8_t ds_species_id_t;
#define DS_SPECIES_NONE 0x00
#define DS_SPECIES_MAX  0x80

typedef enum { DS_STAT_HP, DS_STAT_MP, DS_STAT_OFFENSE, DS_STAT_DEFENSE,
               DS_STAT_SPEED, DS_STAT_BRAINS,               /* group STATS  */
               DS_STAT_WEIGHT,                              /* group WEIGHT */
               DS_STAT_CARE,                                /* group CARE   */
               DS_STAT_DISCIPLINE, DS_STAT_HAPPINESS,
               DS_STAT_BATTLES, DS_STAT_TECHS,              /* group BONUS  */
               DS_STAT_COUNT } ds_stat_t;

typedef enum { DS_GROUP_STATS, DS_GROUP_WEIGHT, DS_GROUP_CARE,
               DS_GROUP_BONUS, DS_GROUP_COUNT } ds_group_t;

/* DS_CMP_RANGE5 encodes DW1's real weight rule: within +/-5g of a target.
   Today the app compares weight with '>' and treats it as a minimum, so an
   overweight Digimon passes every weight requirement. */
typedef enum { DS_CMP_UNUSED = 0, DS_CMP_GE, DS_CMP_LE, DS_CMP_RANGE5 } ds_cmp_t;

typedef struct { int32_t v[DS_STAT_COUNT]; } ds_stats_t;

typedef struct {
    int32_t         v  [DS_STAT_COUNT];
    uint8_t         cmp[DS_STAT_COUNT];   /* ds_cmp_t; UNUSED == not part of this evolution */
    ds_species_id_t bonus_from;           /* free BONUS group if current species matches */
} ds_req_t;

enum { DS_F_SPECIAL = 1u << 0, DS_F_NO_STATGAIN = 1u << 1 };

typedef struct {
    ds_species_id_t        id;
    const char            *name;
    ds_stage_t             stage;
    ds_type_t              type;
    ds_req_t               req;        /* requirements to BECOME this species */
    const ds_species_id_t *evo;        /* static const array; never a stack array */
    uint8_t                evo_count;  /* 1..9 observed in the current table */
    uint8_t                flags;
    uint8_t                sprite_index; /* index into the 128x128 sprite atlas */
} ds_species_t;

/* ===== ds_data.h ===== */
const ds_species_t *ds_species(ds_species_id_t id);          /* NULL if unknown */
const ds_species_t *ds_species_at(int index);
int                 ds_species_count(void);
const char         *ds_stage_name(ds_stage_t);
const char         *ds_type_name(ds_type_t);
const char         *ds_stat_label(ds_stat_t);                /* "Offense" */
const char         *ds_stat_tooltip(ds_stat_t);              /* web-app copy, verbatim */
ds_group_t          ds_stat_group(ds_stat_t);
const char         *ds_special_hint(ds_species_id_t);        /* ex getSpecialDigimonInfo */
const char         *ds_item_name(uint8_t id);
const char         *ds_tech_name(uint8_t bit);               /* 0..63, from digimon-save.hexpat */

/* ===== ds_io.h : THE ONLY PLATFORM SEAM ===== */
typedef struct ds_source {
    int      (*read)(void *ctx, uint64_t off, uint8_t *dst, uint32_t len); /* 0 == ok */
    uint64_t   size;      /* 0 == unknown */
    void      *ctx;
} ds_source_t;
/* app     -> sceIoPread on the .VMP
   plugin  -> sceIoPread (T2) OR memcpy from CONVERT_ADDRESS(0x09800000+off) (T3)
   host    -> fread
   tests   -> memcpy from a fixture buffer */

/* ===== ds_save.h ===== */
#define DS_BLOCK_SIZE  8192      /* 0x2000 */
#define DS_CARD_SIZE   (128*1024)
#define DS_MAX_RAISED  40
#define DS_INV_SLOTS   30

typedef struct {
    ds_species_id_t species;
    ds_stats_t stats;                    /* incl. techs = popcount(mastered_moves) */
    uint16_t cur_hp, cur_mp;
    uint16_t age_days;                   /* 0x03E0+0x4A */
    uint16_t evolve_hours;               /* 0x03E0+0x56 -- "age in hours (for evolve)" */
    uint16_t lifetime_hours_left;        /* 0x03E0+0x48 */
    uint16_t virus_bar, poop_level, tiredness, saturation, sickness_count;
    uint64_t mastered_moves;             /* 0x03BC, low 64 bits */
    uint8_t  techs_equipped[4], tech_priority[4];
    uint8_t  lives_left, tamer_level;
    char     tamer_name[11], digimon_name[11];
    uint8_t  inv_id[DS_INV_SLOTS], inv_qty[DS_INV_SLOTS], inv_count;
    ds_species_id_t raised[DS_MAX_RAISED];   /* 0x0700 + n*0x40 + 0x1C */
    uint8_t  raised_count;
    uint32_t checksum_stored, checksum_calc;
    uint64_t block_offset;
} ds_save_t;

ds_err_t ds_find_block (const ds_source_t *src, uint64_t *out_offset);
ds_err_t ds_read_block (const ds_source_t *src, uint64_t off, uint8_t blk[DS_BLOCK_SIZE]);
ds_err_t ds_parse_block(const uint8_t blk[DS_BLOCK_SIZE], ds_save_t *out);
uint32_t ds_checksum   (const uint8_t blk[DS_BLOCK_SIZE]);
ds_err_t ds_load       (const ds_source_t *src, ds_save_t *out); /* find+read+parse+verify */

uint16_t ds_rd_u16(const uint8_t *b, uint32_t off);
int16_t  ds_rd_i16(const uint8_t *b, uint32_t off);   /* happiness is signed -100..100;
                                                         it is read as a u8 today */
uint32_t ds_rd_u32(const uint8_t *b, uint32_t off);

/* ===== ds_eval.h ===== */
typedef struct {
    ds_stat_t  stat;  ds_group_t group;  ds_cmp_t cmp;
    int32_t    required, lo, hi, current, delta;
    uint8_t    met;    /* 0/1 */
    uint8_t    lost;   /* 1 == unreachable forever (care mistakes only ever rise) */
} ds_check_t;

typedef struct {
    ds_species_id_t target;
    ds_check_t checks[DS_STAT_COUNT];
    int        check_count;
    uint8_t    group_used[DS_GROUP_COUNT], group_met[DS_GROUP_COUNT];
    int        groups_met;         /* DW1 rule: eligible when >= 3 of 4 */
    uint8_t    bonus_is_free, eligible, already_raised;
    int32_t    priority_score;     /* SydMontague's algorithm, integer */
    int32_t    progress_permille;  /* 0..1000, for sorting/bars */
} ds_eval_t;

ds_err_t ds_evaluate    (const ds_save_t *, ds_species_id_t target, ds_eval_t *out);
int      ds_evaluate_all(const ds_save_t *, ds_eval_t *out, int max_out);
int      ds_rank_by_progress(ds_eval_t *e, int n);   /* in place */
int      ds_predict     (const ds_save_t *, ds_eval_t *e, int n); /* index the game WOULD pick */

/* ===== ds_time.h ===== */
uint16_t ds_evolve_threshold_hours(ds_stage_t);  /* 6 / 24 / 72 / 144 */
int32_t  ds_hours_to_evolve (const ds_save_t *); /* negative == overdue */

/* ===== ds_roll.h ===== */
typedef struct { uint32_t s[4]; } ds_rng_t;      /* xoshiro128** -- deterministic, testable.
                                                    Today: srand(time(NULL)) INSIDE the roll
                                                    loop, so two rolls in one second match. */
void     ds_rng_seed (ds_rng_t *, uint64_t seed);
uint32_t ds_rng_next (ds_rng_t *);
uint32_t ds_rng_below(ds_rng_t *, uint32_t n);   /* unbiased, rejection-sampled */

typedef struct {
    unsigned allow_special : 1, exclude_raised : 1, avoid_repeat : 1, only_eligible : 1;
    uint8_t  excluded[DS_SPECIES_MAX / 8];
    ds_species_id_t last_roll;
} ds_roll_opts_t;

int      ds_candidates(const ds_save_t *, const ds_roll_opts_t *,
                       ds_species_id_t *out, int max);
ds_err_t ds_roll      (const ds_save_t *, const ds_roll_opts_t *, ds_rng_t *,
                       ds_species_id_t *out);

/* ===== ds_fmt.h : pure, writes into caller buffers ===== */
int ds_fmt_requirement(char *buf, uint32_t n, const ds_check_t *c); /* "25 - 35" / ">= 100" */
int ds_fmt_evolve_eta (char *buf, uint32_t n, const ds_save_t *s);
int ds_fmt_names      (char *buf, uint32_t n, const ds_species_id_t *ids, int count);
```

**Nothing in `core/` includes `screen.h`, `psp2/*`, `stdio.h` or `stdlib.h`.** That is enforced by a CI grep, not by discipline.

### 3.3 What links where

| Target | Type | Core flavour | Links |
|---|---|---|---|
| `digislots_core_host` | STATIC (host gcc) | default | — (plus `-fsanitize=address,undefined` in CI) |
| `digislots_core_app` | STATIC (arm-vita-eabi) | newlib | — |
| `digislots_core_plugin` | STATIC (arm-vita-eabi) | `-nostdlib`, `-ffreestanding` | — |
| `dsdump`, `ds_tests` | host EXE | — | `digislots_core_host` |
| `digislots_vita` (VPK) | Vita EXE | — | `digislots_core_app`, `vita2d`, `png`, `z`, `m`, `SceGxm_stub`, `SceDisplay_stub`, `SceCtrl_stub`, `SceTouch_stub`, `SceIofilemgr_stub`, `SceLibKernel_stub`, `SceAppMgr_stub`, `SceCommonDialog_stub`, `ScePower_stub` |
| `digislots` (`.suprx`) | Vita MODULE | — | `digislots_core_plugin`, `taihen_stub`, `SceLibc_stub`, `SceDisplay_stub`, `SceCtrl_stub`, `SceIofilemgr_stub`, `SceLibKernel_stub`, `SceKernelModulemgr_stub`, `SceSysmem_stub`, `SceKernelThreadMgr_stub`, `SceAppMgr_stub`, `gcc` — all verified present in `$VITASDK/arm-vita-eabi/lib/` |

**Stub libraries the current CMakeLists links and never uses — delete them:** `SceHttp_stub SceNet_stub SceNetCtl_stub SceSsl_stub SceMtpIfDriver_stub SceMusicExport_stub ScePhotoExport_stub SceUdcd_stub SceUsbstorVStorDriver_stub SceVshBridge_stub ScePromoterUtil_stub SceShellSvc_stub SceAudiodec_stub SceAudio_stub SceIme_stub SceAppUtil_stub SceSysmodule_stub`.

### 3.4 CMake changes — the macro names, correctly

**Verified against `$VITASDK/share/vita.cmake` in the installed toolchain (build 2023-12-25, vita-toolchain `fb328e20`). Exactly three macros exist:**

- `vita_create_self(<output> <source> [UNCOMPRESSED] [UNSAFE] [STRIPPED] [NOASLR] [REL_OPTIMIZE] [CONFIG <yml>] [GEN_EXPORTS <yml>] [ATTRIBUTE n] [MEMSIZE kib] [MODULE_ENTRY "start,stop,exit"])` — vita.cmake:65
- `vita_create_stubs(<target-dir> <source> <config.yml> [KERNEL] [GEN_EXPORTS])` — vita.cmake:263
- `vita_create_vpk(<output.vpk> <titleid> <eboot.self> …)` — vita.cmake:365

**There is no `vita_elf_create` CMake macro.** `vita-elf-create` is a *binary* (`$VITASDK/bin/vita-elf-create`) that `vita_create_self` invokes internally via the `VITA_ELF_CREATE` toolchain variable; you influence it with `VITA_ELF_CREATE_FLAGS` or with the `CONFIG` / `MODULE_ENTRY` / `STRIPPED` options. Do not try to call it as a macro — it does not exist and the original research's phrasing would send you looking for it.

`vita_create_stubs(... KERNEL)` is only needed if a **kernel** module exports syscalls to userland (the `reVita/export.yml` + `syscall: true` pattern). **The primary architecture does not need it** — the plugin is user-mode, in-process, and shares data with the app through a file, not a syscall. Keep it in the back pocket for the kernel fallback.

```cmake
cmake_minimum_required(VERSION 3.16)   # 2.8 is a HARD ERROR on CMake >= 4.0
option(DIGISLOTS_HOST "Build core + tools + tests for the host" OFF)

if(NOT DIGISLOTS_HOST)
  if(NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    if(DEFINED ENV{VITASDK})
      set(CMAKE_TOOLCHAIN_FILE "$ENV{VITASDK}/share/vita.toolchain.cmake" CACHE PATH "toolchain")
    else()
      message(FATAL_ERROR "Define VITASDK, or configure with -DDIGISLOTS_HOST=ON")
    endif()
  endif()
endif()

project(digislots C)
if(CMAKE_BINARY_DIR STREQUAL CMAKE_SOURCE_DIR)
  message(FATAL_ERROR "In-source builds are forbidden. Use: cmake -B build")
endif()

file(GLOB DS_CORE_SRC ${CMAKE_SOURCE_DIR}/core/src/*.c)

# One source set, three flag sets.
function(ds_add_core name)
  add_library(${name} STATIC ${DS_CORE_SRC})
  target_include_directories(${name} PUBLIC ${CMAKE_SOURCE_DIR}/core/include)
  set_target_properties(${name} PROPERTIES C_STANDARD 99 C_STANDARD_REQUIRED ON)
  # -Wvla and -Wformat=2 each catch a real bug that ships today.
  target_compile_options(${name} PRIVATE
    -Wall -Wextra -Wconversion -Wshadow -Wformat=2 -Wvla
    -Werror=return-type -Werror=return-local-addr -Wno-unused-parameter)
  target_compile_options(${name} PRIVATE ${ARGN})
endfunction()

if(DIGISLOTS_HOST)
  ##### HOST: core + oracle CLI + unit tests. No VitaSDK required. #####
  enable_testing()
  ds_add_core(digislots_core_host)
  add_executable(dsdump   tools/dsdump.c)
  add_executable(ds_tests tests/test_locate.c tests/test_parse.c
                          tests/test_eval.c   tests/test_roll.c tests/test_table.c)
  target_link_libraries(dsdump   PRIVATE digislots_core_host)
  target_link_libraries(ds_tests PRIVATE digislots_core_host)
  add_test(NAME core COMMAND ds_tests ${CMAKE_SOURCE_DIR}/tests/fixtures)
  option(DIGISLOTS_SAN "sanitizers" ON)
  if(DIGISLOTS_SAN)
    foreach(t digislots_core_host dsdump ds_tests)
      target_compile_options(${t} PRIVATE -fsanitize=address,undefined -fno-omit-frame-pointer)
      target_link_options(${t}    PRIVATE -fsanitize=address,undefined)
    endforeach()
  endif()
else()
  ##### VITA #####
  include("${VITASDK}/share/vita.cmake" REQUIRED)
  set(VITA_APP_NAME "DigiSlots Vita")
  set(VITA_TITLEID  "VSDK00007")
  set(VITA_VERSION  "01.00")

  ds_add_core(digislots_core_app)
  ds_add_core(digislots_core_plugin -nostdlib -ffreestanding)

  # ---- the standalone app -------------------------------------------------
  add_executable(digislots_vita
    app/main.c app/app.c app/io_vita.c app/input_vita.c
    app/render/gfx.c app/render/scr_status.c app/render/scr_roll.c
    app/render/scr_requirements.c app/render/scr_settings.c app/render/scr_dex.c)
  target_link_libraries(digislots_vita PRIVATE
    digislots_core_app vita2d png z m
    SceGxm_stub SceDisplay_stub SceCtrl_stub SceTouch_stub
    SceIofilemgr_stub SceLibKernel_stub SceAppMgr_stub
    SceCommonDialog_stub ScePower_stub)
  vita_create_self(digislots_vita.self digislots_vita UNSAFE)
  vita_create_vpk(digislots_vita.vpk ${VITA_TITLEID} digislots_vita.self
    VERSION ${VITA_VERSION} NAME ${VITA_APP_NAME}
    FILE sce_sys/icon0.png                          sce_sys/icon0.png
    FILE sce_sys/livearea/contents/bg.png           sce_sys/livearea/contents/bg.png
    FILE sce_sys/livearea/contents/startup.png      sce_sys/livearea/contents/startup.png
    FILE sce_sys/livearea/contents/template.xml     sce_sys/livearea/contents/template.xml
    FILE app/assets/atlas.png                       assets/atlas.png
    FILE app/assets/panel9.png                      assets/panel9.png
    FILE app/assets/ui.ttf                          assets/ui.ttf
    FILE app/assets/glyphs.ttf                      assets/glyphs.ttf)
  # NOTE: the two duplicated file(COPY resources/ ...) blocks in the current
  # CMakeLists are deleted. They shipped the 27k-line DMW_New.CT inside the VPK,
  # and the VPK FILE directive only resolved because builds were in-source.

  # ---- the overlay plugin (Phase 6+, gated on the spikes) -----------------
  option(DIGISLOTS_PLUGIN "Build the Adrenaline overlay .suprx" OFF)
  if(DIGISLOTS_PLUGIN)
    add_executable(digislots plugin/main.c plugin/pspemu.c plugin/blit.c)
    target_link_libraries(digislots PRIVATE
      digislots_core_plugin
      taihen_stub SceLibc_stub
      SceDisplay_stub SceCtrl_stub SceIofilemgr_stub
      SceLibKernel_stub SceKernelModulemgr_stub SceSysmem_stub
      SceKernelThreadMgr_stub SceAppMgr_stub gcc)
    set_target_properties(digislots PROPERTIES LINK_FLAGS "-nostdlib")
    # CONFIG takes plugin/digislots.yml, which declares only:
    #   digislots: { attributes: 0, version: {major: 1, minor: 0},
    #                main: { start: module_start, stop: module_stop } }
    vita_create_self(digislots.suprx digislots CONFIG ${CMAKE_SOURCE_DIR}/plugin/digislots.yml UNSAFE)
    # If a KERNEL fallback becomes necessary, that module additionally uses:
    #   vita_create_self(digislots_k.skprx digislots_k CONFIG plugin_k/export.yml UNSAFE)
    #   vita_create_stubs(stubs digislots_k ${CMAKE_SOURCE_DIR}/plugin_k/export.yml KERNEL)
    # which emits libDigislotsKernel_stub.a for the user side. Not needed for the primary path.
  endif()
endif()
```

### 3.5 IPC / data path

There is deliberately **no IPC** in the primary design. The plugin runs in-process and owns its whole pipeline; the app runs at a different time. They share only two files under `ux0:data/digislots/`:

| File | Writer | Reader | Contents |
|---|---|---|---|
| `settings.bin` | app (and plugin, on close) | both | versioned packed struct: `{u32 magic 'DGSL', u16 version, u16 size, ds_roll_opts_t, ui_opts, last_roll, card_path[128], card_slot}` |
| `snapshot.bin` | plugin | app | last parsed `ds_save_t` + timestamp, so the app can show "as of your last session" without the game running |
| `digislots.log` | plugin | human | `sceIoWrite` diagnostics; there is no debugger |

If S1 forces the kernel fallback, the data path becomes a `vita_create_stubs(... KERNEL)` syscall library (`digislots_getState(ds_save_t *out)` wrapped in `ENTER_SYSCALL`/`EXIT_SYSCALL`, marshalled with `ksceKernelCopyToUser`, mind the 4096-byte syscall stack limit documented in taiHEN `USAGE.md`) — but that is a contingency, not the plan.

### 3.6 Diagram

```
 ┌────────────────────────────── HOST (CI, Linux, gcc) ───────────────────────────────┐
 │  tests/*.c  ──►  ds_tests  ──►  ctest        tools/dsdump.c ──► dsdump card.vmp     │
 │        │                                            │                              │
 │        └────────────┬───────────────────────────────┘                              │
 │              digislots_core_host.a      ds_source_t.read = fread / memcpy(fixture)  │
 └──────────────────────────────────┬─────────────────────────────────────────────────┘
                                    │  same .c files, three flag sets
                ┌───────────────────┴────────────────────┐
                ▼                                        ▼
 ┌──── digislots_core_app.a ────┐          ┌──── digislots_core_plugin.a ────┐
 │  newlib, VitaSDK             │          │  -nostdlib -ffreestanding       │
 └──────────────┬───────────────┘          └────────────────┬────────────────┘
                │                                           │
 ══════════════ VITA ═══════════════════════   ═════════════════════════════════════════
                │                                           │
 ┌──────────────▼────────────────────┐   ┌──────────────────▼──────────────────────────┐
 │  app/  digislots_vita.vpk         │   │  plugin/  ur0:tai/digislots.suprx           │
 │  LiveArea bubble, own process     │   │  ur0:tai/config.txt   *PSPEMUCFW  (or *ALL   │
 │                                   │   │                        + ScePspemu guard)    │
 │  vita2d + SceGxm (owns GXM)       │   │                                              │
 │  io_vita.c: sceIoPread(.VMP)      │   │  injected into the ScePspemu process         │
 └──────────────┬────────────────────┘   └──────────────────┬──────────────────────────┘
                │                                           │
                │  ux0:data/digislots/{settings,snapshot}.bin│
                └───────────────────────◄───────────────────┘

 ══════════ inside the ScePspemu process (titleid PSPEMUCFW) ═════════════════════════

   ┌─────────────────────────────────────────────────────────────────────────────┐
   │ ScePspemu (Sony, closed)                                                    │
   │   └─ emulated PSP 6.61 + Adrenaline CFW ──► `pops` PRX ──► Digimon World 1  │
   │                                                                             │
   │   emulated-PSP RAM 0x88000000..0x8C000000                                   │
   │        │  CONVERT_ADDRESS(a) = 0x68000000 + (a & 0x0FFFFFFF)   [V]          │
   │        ▼                                                                    │
   │   Vita VA   0x70000000..0x74000000   (flat, dereferenceable)                │
   │        PSX main RAM  @ 0x71800000        (== PSP 0x09800000)  [L, spike S3] │
   │        DW1 partner blk @ +0x138460  (== save offset 0x03E0)   [V, DMW_New.CT]│
   └───────────────┬─────────────────────────────────────────────────────────────┘
                   │  read-only memcpy                    ┌─────────────────────┐
   ┌───────────────▼──────────────────┐                   │ AdrenalineUser      │
   │ digislots.suprx                  │                   │ (adrenaline_user)   │
   │                                  │                   │  owns vita2d/GXM    │
   │ [1] ds_source_t.read             │                   │  presents the frame │
   │       = memcpy from 0x71800000   │◄── T3             └──────────┬──────────┘
   │       = sceIoPread(.VMP)         │◄── T2                        │ imports
   │ [2] ds_load/ds_evaluate (core)   │                              │
   │ [3] worker thread composes an    │        HOOK ── sceDisplaySetFrameBuf
   │     ARGB panel (dirty-flagged)   │        SceDisplayUser libnid 0x4FAACD11
   │ [4] display hook: scanline copy  │────────► NID     0x7A410B64
   │     panel ──► pParam->base       │        (NOT 0x5ED8F994 — that library's
   │                                  │         export is _sceDisplaySetFrameBuf
   │ [5] combo detect (peek, non-     │         0xF51523CB and will not resolve)
   │     destructive) + guest gate    │
   └───────────────┬──────────────────┘        HOOK ── sceCompatWriteSharedCtrl
                   └────────────────────────────► ScePspemu import,
                                                  SceCompat libnid 0x0F35909D
                                                  NID 0x2306FFED
                                                  (zero pad_data->Buttons while open)

   HARD GATE, both hooks: Adrenaline Settings -> Graphics Filtering must NOT be
   "Original". In Original mode AdrenalineUser never presents (menu.c:657) and the
   compat setter carries pParam == NULL (pops.c:172-186). Detect and instruct.
```

---

## 4. Implementation plan

Every phase is individually shippable. Phases 0–4 are worth doing **whether or not the overlay ever exists** — they are the fallback and the foundation simultaneously. Difficulty 1–5.

### Phase 0 — Unbreak the build (difficulty 1)

| | |
|---|---|
| **Deliverable** | The repo builds from a clean clone on a 2026 toolchain and a modern CI runner |
| **Files** | `CMakeLists.txt`, `.github/workflows/build.yml`, `.gitignore` |
| **Depends on** | — |
| **Verify** | `git clone && cmake -B build && cmake --build build` produces a `.vpk`; CI green on a fresh runner |

- `cmake_minimum_required(VERSION 2.8)` → `3.16`. **This is a hard error under CMake ≥ 4.0** ("Compatibility with CMake < 3.5 has been removed"). Nothing else works until this is fixed.
- CI: `ubuntu-20.04` → `ubuntu-24.04`; `actions/checkout@v3` → `@v4`; `actions/upload-artifact@v3` → `@v4`; `softprops/action-gh-release@v1` → `@v2`. The v3 artifact backend was shut down in Jan 2025 and the 20.04 runners were removed in 2025 — CI is currently dead, not merely stale.
- `git rm --cached CMakeCache.txt Makefile cmake_install.cmake digislots_vita digislots_vita.vpk hello.txt`; extend `.gitignore`. `CMakeCache.txt` hardcodes the original author's absolute paths and will fight every fresh clone.
- Delete both duplicated `file(COPY ${CMAKE_SOURCE_DIR}/resources/ ...)` blocks and forbid in-source builds. The VPK's `FILE ${CMAKE_BINARY_DIR}/src/resources resources` only resolves today because `CMAKE_BINARY_DIR == CMAKE_SOURCE_DIR` — out-of-source builds silently ship a VPK with no sprites.
- Delete `src/resources/image.png` (63,164 B, 460×460, referenced by nothing — 61% of the resource directory).

### Phase 1 — Core extraction and the memory-safety fix (difficulty 3)

| | |
|---|---|
| **Deliverable** | `core/` compiles standalone under host gcc with `-fsanitize=address,undefined`; `dsdump card.vmp --json` prints a full parse on a laptop; the dangling-pointer bug is gone |
| **Files** | new `core/**`, `tools/dsdump.c`, `tests/**`; delete `src/models/digimon.c`'s print functions, `src/screen.c/h` coupling |
| **Depends on** | Phase 0 |
| **Verify** | `ctest` green under ASan/UBSan; `dsdump` output matches the on-device screen for a known save |

- **The critical fix:** replace `initialiseDigimon()` (digimon.c:427-669) with a file-scope `static const ds_species_t ds_table[]`. Every one of the 59 evolution arrays is currently a *function-local automatic* whose address is stored into `digimon->evolutionPath`; every later read (`getRandomEvolution`, `getEvolutionNameStrings`, `printDigimonStats`) reads a dead stack frame. The app only appears to work because nothing deterministically clobbers that region.
- Split type-level from instance-level state: `const ds_species_t` (table, `.rodata`) vs `ds_save_t` (parsed instance). Today `setDigimonStatsFromBuffer()` writes the *player's* stats into the *global species struct*, which blocks `const`-ing the table and blocks evaluating several candidates at once.
- Sever the UI: delete `printDigimonStats` / `printEvolutionRequirements` / `getEvolutionNameStrings` and `#include "../screen.h"` from `models/digimon.c`. `printEvolutionRequirements` is 130 lines of copy-pasted `if (req.X > 0) print(...)` with inline ANSI escapes (`\e[16G`) hardcoding a monospace grid.
- Kill allocation: `getEvolutionNameStrings()` mallocs and is never freed (leaks per roll); `getRandomEvolution` declares `Digimon* nonSpecial[digi->evolutionPathSize]` which is a **zero-length VLA** (UB in C99) for Etemon and Vademon, whose `evolutionPathSize` is 0. `-Wvla` catches the second at compile time.
- Deterministic RNG: `ds_rng_t` (xoshiro128\*\*), seeded once. Today `srand(time(NULL))` is called *inside* the roll loop (main.c:119-128), so two rolls within one wall-clock second are identical.
- Bounds fix: `ascii_m.c:36` returns `hexAsciiMapping[0x8148].asciiChar` — element 33,096 of a **79-element** array. Latent only because name decoding isn't wired up yet; it will fire the moment names are displayed. Intent was clearly `return '?';`.
- Test oracle: `tests/data/requirements.csv` extracted from SydMontague's evolution guide; `test_table.c` asserts the C table against it row by row.

### Phase 2 — Correctness: the data and the rules (difficulty 3)

| | |
|---|---|
| **Deliverable** | The app prints numbers that are actually right, including for the default starter |
| **Files** | `core/src/ds_data.c`, `ds_save.c`, `ds_save_locate.c`, `ds_eval.c`, `app/io_vita.c` |
| **Depends on** | Phase 1 |
| **Verify** | `test_table` passes against the CSV; `test_eval` covers the 3-of-4 rule, `>=`, weight ±5 and `lost`; `test_locate` finds the block in a fixture where DW1 is not in card slot 1 |

Ordered by user-visible severity:

1. **Five rotated requirement rows.** Agumon, Airdramon, Angemon, Andromon and Bakemon each hold another's requirements. These are exactly the five the web app's v0.1.3 changelog says it fixed — the Vita port was forked from web v0.1.2 and never rebased. **Agumon is the default starter**, so most users see wrong numbers immediately.
2. **`>` should be `>=`.** All 11 comparison sites use `>`; the game's rule is "match *or exceed*". A Digimon at exactly 100 Offense against a 100 requirement currently shows unmet.
3. **Weight is a ±5 range, not a minimum.** Greymon's stored `30` means 25–35. Today an overweight Digimon satisfies every weight requirement — false positives on the single most actively-managed stat in DW1. → `DS_CMP_RANGE5`.
4. **`minCare` / `minBattles` are parsed and never read.** "Care Mistakes ≤ 1" renders as "≥". Care mistakes only rise and reset on evolution, so a blown max-care requirement is *permanently lost* — the exact opposite of what the UI implies. → `ds_check_t.lost`, rendered struck-through.
5. **The real eligibility rule is 3-of-4 groups** (Stats / Weight / Care / Bonus), where Bonus is an OR over {happiness, discipline, battles, techniques, free-digimon-bonus}. The app shows 11 flat rows implying all are required. This is the single highest-value change and the web app doesn't have it either.
6. **`techs` is stored for nearly every champion/ultimate (28/30/35/40/45/49) and never displayed** — `printDigimonStats` literally prints `"Tech: TODO"`. The data is already in the repo: `techs_learned = popcount(mastered_moves)` over the 16-byte bitfield at save `0x03BC`, fully named in `resources/digimon-save.hexpat`.
7. **Happiness is signed −100..100 as int16, decoded as u8.** A Digimon at −50 reads as 206 and satisfies every happiness requirement. `digimon-save.hexpat:108` declares `u16 happiness`; every other partner value already goes through the 16-bit path.
8. **Block location is hardcoded `0x2080`.** Port the hexpat's signature scan (`find_sequence(0, 53 43 13 01 82 63 82 89)`) plus the checksum at `0x06FC` into `ds_find_block` / `ds_checksum`. Also resolves the memory-stick-root bug: Adrenaline's root is user-configurable across `ux0/ur0/imc0/xmc0/uma0` (`user/utils.c:316-329 getPspemuMemoryStickLocation()`), and the app hardcodes `ux0:`.
9. **IO error handling.** `readFileToBuffer`'s return is discarded (main.c:95), so an open/read failure parses an *uninitialised* 8192-byte stack buffer and shows plausible-looking random stats; the fd leaks when `sceIoPread` fails; a short read is treated as success because only `< 0` is checked.
10. **Format-string defect.** `print(path)` passes a non-literal to a `__attribute__((format(printf,1,2)))` function — any `%` in a user-supplied path is UB. Must be fixed *before* paths become configurable. `-Wformat=2` catches it.

### Phase 3 — App UI on vita2d (difficulty 3)

| | |
|---|---|
| **Deliverable** | The standalone app looks like a Vita title; `psvDebugScreen` is deleted |
| **Files** | new `app/render/**`, `app/assets/**`, `tools/mkatlas.c`; delete `common/debugScreen.c` |
| **Depends on** | Phase 2 (**do not start before Phase 2 lands** — the UI must be built against the 4-group data model, not 11 flat rows) |
| **Verify** | On-device screenshots of all five screens; no glyph is silently missing at any size |

vita2d and libpng are already in the link line, so this adds zero dependencies. Design half owns the visuals; the technical constraints are:

- **`vita2d_font` uses a fixed 512×512 atlas and silently drops glyphs on overflow** (`vita2d_font.c`: `ATLAS_DEFAULT_W/H 512`; `atlas_add_glyph` returns 0 with no error). Cap the type scale at 6 sizes and **never animate the `size` argument** — every distinct size allocates new atlas rows, and ramping it per frame exhausts the atlas in seconds. Animate position/alpha, or draw to a render target.
- **`vita2d_font` rescales cached glyphs across sizes** (`draw_scale = size / (float)data.glyph_size`) — the *first* size a glyph is drawn at is the only size it is ever rasterised at. Load **one `vita2d_font*` per point size**, not one handle used at several sizes. Same trap in `vita2d_pvf.c` (`font->vsize = 10.125f` hardcoded).
- **`vita2d_texture_set_filters(tex, SCE_GXM_TEXTURE_FILTER_POINT, SCE_GXM_TEXTURE_FILTER_POINT)`** on all sprites, drawn at integer scales. 16×16 art at 4–10× under the default bilinear filter is mush.
- **No rounded-rect / gradient / shadow primitives exist.** Gradients: `vita2d_draw_array(SCE_GXM_PRIMITIVE_TRIANGLE_STRIP, verts, 4)` with `vita2d_color_vertex` (per-vertex colour, zero texture cost). Rounded panels: one 48×48 alpha-only 9-slice PNG drawn with `vita2d_draw_texture_tint_part_scale` (tint at draw time so one texture serves every colour and size).
- Asset hygiene: pack all 63 sprites into a single 128×128 atlas (8×8 grid of 16×16 cells — exactly fits) to replace 63 `vita2d_load_PNG_file` calls and 63 libpng decodes. Normalise `Machinedramon.png` and `Player.png`, which are genuinely 32×32 (bboxes 31×28 / 30×28) and will render double-size at a shared scale factor. **Always derive scale as `TARGET_PX / vita2d_texture_get_width(tex)`, never a constant.**

### Phase 4 — QoL + settings persistence (difficulty 2)

| | |
|---|---|
| **Deliverable** | The README TODO list is closed; the app remembers you |
| **Files** | `core/src/ds_time.c`, `ds_roll.c`, `ds_eval.c`; `app/io_vita.c`, `app/render/scr_settings.c` |
| **Depends on** | Phases 2, 3 |
| **Verify** | Host unit tests for time/roll/exclusion logic; on-device round-trip of `settings.bin` across a reboot |

README TODOs mapped to the work:

| README TODO | Concretely | Data status | Diff |
|---|---|---|---|
| Age & time-to-digivolve | `0x03E0+0x4A` (days), `+0x56` (evolve-hours), `+0x48` (lifetime); thresholds 6/24/72/144 h by stage, counter resets on evolution | **All present.** `+0x56` is labelled "Unknown" in `resources/DMW Memory Card Offsets.txt` but is identified as *"Age in hours (for evolve)"* by `resources/DMW_New.CT` at `PSXBase+0x1384B6` (0x1384B6 − 0x138460 = 0x56) | 2 |
| Read learned techs | `popcount` of `0x03BC..0x03CB`; names already written in the hexpat | Present; **exact valid mask uncertain** — the hexpat names 64 bits and flags the "filth" group as "seem wrong" | 2 |
| Exclude already-owned | Registered Digimon array `0x0700..0x10FF`, 40 × 0x40, species id at `+0x1C` | Present. **Not just a filter preference** — the game *itself* disables already-owned candidates once an unowned one is in the running, so this belongs in the prediction logic | 2 |
| min/max battles & care | Phase 2 item 4 | Present | 1 |
| Configurable save path / slot | `ds_find_block` + cycle SCEVMC0/1 + probe all five memory-stick roots | Present | 2 (3 with a file browser) |
| Web-app-parity options | `showSpecialDigimon`, exclusion list, `hideIrrelevant`, `showSpoilers`, `showInfoText` → `ds_roll_opts_t` + UI flags | Present | 2 |
| Nice GUI | Phase 3 | — | 3–4 |
| Persistent settings + last roll | `ux0:data/digislots/settings.bin`, versioned struct with magic+version prefix (`Electry/PSVshell src/profile.c` pattern) | — | 2 |

Plus three features the web app *cannot* have, because it has no save data:
- **Pinned target with a live checklist** — pin a roll, then see only that target's 4 groups with met / unmet / *permanently lost* state and exact deltas.
- **"What am I on track for"** — `ds_evaluate_all` + `ds_rank_by_progress` across the current evolution path.
- **True outcome prediction** — `ds_predict` implementing SydMontague's documented priority algorithm (score = Σ required stats, HP and MP ÷ 10, plus carry-over, ÷ count; evaluated in chart order). The web app's own README lists this as an unimplemented TODO.

### Phase 5 — Overlay spikes (difficulty 4, mostly risk not code)

| | |
|---|---|
| **Deliverable** | A go/no-go decision with named evidence, plus ~200 lines of throwaway plugin |
| **Files** | `spikes/` (never merged to `main`) |
| **Depends on** | Phase 0 only — **can run in parallel with Phases 1–4** |
| **Verify** | The pass/fail signals in §2 |

Run S1 → S2 → S3 → S4 → S5 → S6 → S7. If S1 fails after both injection routes and the psonescrot kernel-loader fallback, stop: ship T1 + T0 and close the overlay track.

### Phase 6 — Overlay MVP: HUD strip on `.VMP` data (T2) (difficulty 4)

| | |
|---|---|
| **Deliverable** | `digislots.suprx` + install instructions; a toggleable read-only panel over the running game showing last-saved stats and the pinned target's checklist |
| **Files** | `plugin/main.c`, `plugin/pspemu.c`, `plugin/blit.c`, `plugin/digislots.yml`, `tools/mkatlas.c` (RGBA blob + `bin2c`), CMake `DIGISLOTS_PLUGIN` branch |
| **Depends on** | Phases 1, 2, 5 |
| **Verify** | Panel appears over DW1; DW1's framerate is unchanged (S5 methodology); pressing the combo does not move the Digimon; unloading the plugin leaves the system clean |

- Hooks: `sceDisplaySetFrameBuf` on `AdrenalineUser` (libnid `0x4FAACD11`, NID `0x7A410B64`); `sceCompatWriteSharedCtrl` on `ScePspemu` (libnid `0x0F35909D`, NID `0x2306FFED`).
- Guards, copied verbatim from PSVshell/reVita: bail if `!pParam || !pParam->base`; bail if `pParam->pixelformat != SCE_DISPLAY_PIXELFORMAT_A8B8G8R8` (`= 0`); handle `width` 480/640/720/960 (pitch is in **pixels**, not bytes — `psp2common/display.h:59`).
- Renderer: a worker thread (`sceKernelCreateThread`, ~50 ms tick) composes an ARGB panel into a `sceKernelAllocMemBlock(SCE_KERNEL_MEMBLOCK_TYPE_USER_RW)` buffer with a dirty flag; the display hook does **only** scanline `memcpy`s of dirty rows. Budget a panel (≈ 640×400 max), not a screen — CDRAM writes are uncached.
- Text and sprites: **no libpng in the plugin.** `tools/mkatlas` converts the 63 PNGs to one raw RGBA blob and the TTF to an 8bpp glyph atlas at build time; both are embedded with the `ld -r -b binary` resource trick (`Adrenaline user/CMakeLists.txt` `ADD_RESOURCES` demonstrates it). 63 × 16 × 16 × 4 = 64 KB — comfortably small (reVita's whole `.skprx` is 86,609 B).
- Alpha blending: neither PSVshell's nor vitaRescale's blitter has any — both write opaque boxes. Take the integer blend from `Adrenaline cef/satelite/blit.c` (`adjust_alpha` + `(col & 0xFFFFFF) + c1 + c2`), ~20 lines.
- **Licensing (blocker — resolve before writing a line):** `Electry/PSVshell` and `Electry/VGi` have **no LICENSE file** (`gh api repos/Electry/PSVshell/license` → 404) — all rights reserved, *not* copyable, and the original research's "copy PSVshell's gui.c (MIT-ish)" recommendation is the one legally unusable option. `MERLev/reVita`, `JSilva6/reVita-extended`, `Electry/VitaGrafix` and `joel16/PSV-VSH-Menu` are **GPL-3.0** — lifting their renderers forces DigiSlots to GPL-3.0. `SonicMastr/Sonic-Overlay` and `GrapheneCt/PSVshellPlus` are MIT. `digislots-vita` currently has no LICENSE at all. **Decide: write the blitter fresh (~200 lines against a documented struct) or consciously adopt GPL-3.0 and derive from reVita-extended. Add a LICENSE file either way.**
- Lifecycle: `void _start() __attribute__((weak, alias("module_start")))`; release **every** hook in `module_stop`; beware taiHEN issue #12 (hooking a function while it is executing can crash).
- README: the Graphics Filtering instruction, the hold-L escape hatch, and the exact `ur0:tai/config.txt` lines.

### Phase 7 — Live PSX RAM (T3) (difficulty 5)

| | |
|---|---|
| **Deliverable** | The overlay shows *current, unsaved* stats |
| **Files** | `plugin/pspemu.c` (address resolution), new `core/include/digislots/ds_ram_offsets.h`, `plugin/main.c` (source swap) |
| **Depends on** | Phase 6 + spike S3 green |
| **Verify** | Feed the Digimon in-game without saving; the overlay's weight changes within one refresh tick and matches what the game shows |

- Swap `ds_source_t.read` from `sceIoPread` to `memcpy` from `CONVERT_ADDRESS(0x09800000 + off)` = `0x71800000 + off`. **This is the entire change if S3's contiguity test passed** — the parser, evaluator and renderer are untouched. That is the payoff of the core split.
- If contiguity failed, add `ds_ram_offsets.h` derived from `resources/DMW_New.CT`: `PSXBase+0x138460` (partner block, == save `0x03E0`), `+0x1557E0` (Offense, == save `0x0470`), `+0x1384B6` (evolve hours), `+0x1384A8` (remaining lifetime), `+0x154F9A` (**"Evolve Target ID"** — the game's own decided outcome, the ultimate version of this tool, though it is unknown when in the frame that value is valid).
- Portable variant of the address translation: resolve `ScePspemuConvertAddress` at `text_addr + 0x6364 + 1` via `taiGetModuleInfo("ScePspemu", &info)` + `module_get_offset` (firmware-independent per isage `user/main.c:123`), and call it with `0x89800000`. Guard on `module_nid` and **no-op on unknown NIDs** rather than guessing.
- **Cache coherency, unaddressed anywhere in the research:** Adrenaline calls `ScePspemuWritebackCache()` after every write to shared ePSP RAM (isage `user/main.c:136, 234, 262, 269, 320, 329`). A reader of emulator-authored memory may observe stale cache lines, and there is no exposed invalidate primitive. Treat live reads as best-effort; validate against a known-changing value before trusting them; keep the `.VMP` path as a fallback the user can select.
- Do **not** hold `ScePspemuPausePops(1)` while the panel is open in v1 — it is available (isage `user/main.c:131-152`) and it would guarantee a consistent snapshot, but pausing POPS is a behaviour change with unknown interaction with Adrenaline's own menu. Add it behind a setting only after Phase 7 is stable.

### Phase 8 — CI and release for two artifacts (difficulty 2)

| | |
|---|---|
| **Deliverable** | Every push runs host tests; every tag ships `digislots_vita.vpk` + `digislots.suprx` + a README with install instructions |
| **Files** | `.github/workflows/{host,vita,release}.yml` |
| **Depends on** | Phases 1, 6 |
| **Verify** | A tag produces both artifacts; the host job fails on a deliberately corrupted `requirements.csv` row |

Three jobs:
1. **`host`** — `ubuntu-24.04`, plain gcc, `cmake -B build -DDIGISLOTS_HOST=ON && ctest`, ASan+UBSan on. Fast (< 60 s) and it is the job that catches data-table regressions. Add a grep gate: `! grep -rE '#include <(stdio|stdlib)\.h>|\bmalloc\(|psp2/' core/src core/include`.
2. **`vita`** — vdpm bootstrap + `install-all`, `cmake -B build && cmake --build build`, uploads the `.vpk` and (when `DIGISLOTS_PLUGIN=ON`) the `.suprx`.
3. **`release`** — on tag, `softprops/action-gh-release@v2`.

Pin the vdpm bootstrap or record the resolved SDK build in the job summary — the local SDK here is a 2023-12-25 snapshot while CI installs current, and the two can diverge.

### Explicitly NOT in scope

| Excluded | Why |
|---|---|
| **Any write to the save file or to PSX RAM** | See §5. The tool is read-only, permanently, unless the repo owner explicitly says otherwise. |
| A kernel `.skprx` for the overlay | The user-mode path is strictly better for this use case and carries far less brick risk. Kernel is a *contingency* if spike S1 fails, and even then only as a loader. |
| A PSP-side POPS `.prx` (T4) | Second MIPS toolchain, 480×272 canvas, and the visibility mechanism *pauses the game* rather than overlaying it. `raing3/tempar`'s own docs/TODO.md says Vita compat is unverified by its author. |
| `SceSharedFb` / `ScePaf` SceShell plugin (T7) | Needs DolceSDK/MSVC or a system-budget process; no precedent for overlaying a game. |
| Forking or patching Adrenaline | Breaks on every Adrenaline update; isage v8 shipped three releases in July 2026 alone. |
| Adrenaline savestate parsing as a *product* feature (T6) | Real corruption risk on DW1 specifically. Used only offline on a PC for offset discovery in S3. |
| Dynamically reloading the v8 PNG overlay | Verified use-after-free: `overlay_texture` is freed from the Kermit handler thread while AdrenalineDraw dereferences it, unsynchronised. |
| ePSXe / Duckstation desktop build | Deferred. `dsdump` already gives 80% of the value on a `.mcr` file with zero extra work, and `SydMontague/DigimonWorldAPI` only supports ePSXe 1.9.0 anyway. |
| Multi-language / CJK | The subsetted font and the 512×512 atlas budget assume Latin. Revisit only if tamer names decode to non-ASCII. |
| PSVita 3.60-only or PSTV-only shortcuts | Target the whole 3.60–3.74 range; kernel/user NID hooks are firmware-stable (NIDs are name hashes), so this costs nothing. |

---

## 5. Risks and mitigations

### 5.1 The read-only rule — non-negotiable

**DigiSlots must never write to `SCEVMC0.VMP`, to `SCEVMC1.VMP`, or to PSX RAM, unless the repo owner explicitly changes this decision.**

A corrupted PSX memory card destroys an irreplaceable, hours-long playthrough, and DW1 has no cloud save and no in-game backup. The blast radius is unbounded and the upside of writing is zero for this tool's purpose.

Enforcement, mechanical rather than cultural:

| Layer | Control |
|---|---|
| `ds_source_t` | Has **only** a `read` callback. There is no `write` member and none may be added. |
| App / plugin IO | `sceIoOpen(..., SCE_O_RDONLY, 0)` at every call site. Never `SCE_O_WRONLY`, `SCE_O_CREAT`, `SCE_O_TRUNC`, `SCE_O_APPEND` on any path under `PSP/SAVEDATA/`. |
| CI gate | `! grep -rnE 'SCE_O_(WRONLY\|RDWR\|CREAT\|TRUNC\|APPEND)' app/io_vita.c plugin/` except for `ux0:data/digislots/*`. Fail the build on a hit. |
| Plugin memory | The live-RAM path is `memcpy(dst, src, n)` only. No pointer into `0x70000000..0x74000000` is ever written. No `ScePspemuWritebackCache` call — that API only matters when you *have* written. |
| Doc | README states "DigiSlots never modifies your save" prominently, next to the install instructions. |

Secondary corruption vector to warn users about: **do not use Adrenaline savestates with Digimon World while using DigiSlots.** Community reports name DW1 specifically for post-savestate audio corruption, and once a savestate is loaded, subsequent in-game saves land in the memory-card image *inside the savestate* rather than in `ux0:pspemu/PSP/SAVEDATA/SLUS01032/SCEVMC0.VMP` — which would make DigiSlots silently read a stale card while the player believes they saved.

### 5.2 Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | **Brick / bootloop from a kernel plugin.** Kernel patches are *not* auto-cleaned on failure; a bad `*KERNEL` line can make the console unbootable. | Low (the primary path is user-mode) | Severe | The overlay is **user-mode**; the only kernel code is throwaway spike S2. Document the escape hatch *before* the first kernel test: **hold L / L1 while launching an app and taiHEN skips all plugin loading** (`hen.c:368-373`). Keep a `ur0:tai/config.txt` backup and a known-good `ux0:app` on the card. Never test kernel code on a console you can't recover. User-space patches are cleaned when the process exits — that alone removes most of this class. |
| R2 | **Crash inside the display hook takes down the game (and the player's unsaved progress).** | Medium | High | Validate `pParam`, `pParam->base`, `pixelformat`, `width`, `pitch` on *every* invocation before touching a byte. Never write past `pitch * height * 4`. Never allocate or take a lock in the hook — the worker thread composes, the hook only copies. Respect taiHEN issue #12 (hooking a function while it executes can crash): install hooks in `module_start`, never dynamically. Ship a "panic" path: on any inconsistency, set a latch that disables all drawing for the rest of the session rather than retrying. |
| R3 | **Adrenaline version drift.** `TheOfficialFloW/Adrenaline` is dead (v7, 2020); `isage/Adrenaline` v8.0.x shipped three releases in July 2026 and changed the PSP plugin loader format, file layout and line numbers. `ScePspemuConvertAddress` and `ScePspemuPausePops` are resolved by **hardcoded text offsets**, keyed to `module_nid`. | High | Medium | Prefer NID-based hooks (NIDs are name hashes and are firmware-stable) over offset-based resolution wherever possible. For the two offset-resolved functions, guard on `module_nid` exactly as isage does, and **no-op gracefully on an unknown NID** rather than jumping into arbitrary code. Read `SceAdrenaline` at the *fixed* VA `0x73CDE000` (= `CONVERT_ADDRESS(0xABCDE000)`) for `pops_mode` and `titleid` — two shipping third-party plugins hardcode that address (`Electry/VitaPresence` main.c:100, `MERLev/reVita` main.c:181), and it needs no offset resolution. Log the detected Adrenaline version and refuse to arm on untested ones. |
| R4 | **Vita firmware spread 3.60–3.74.** | Medium | Low–Medium | Good news, verified: taiHEN is frozen at v0.11 (last commit 2018-02-20) so there is no API churn, and NID-based hooks are firmware-independent. The exposure is entirely via Adrenaline's own offset tables (R3). Note `ksceAppMgrIsExclusiveProcessRunning` exists only as NID `0xA888C2DC` in `db/360/SceAppMgr.yml:232` with **no psp2kern header declaration** — forward-declare it yourself (PSVshell `src/main.c:15` does). Expect more NID-only symbols; the vita-headers NID DB is a strict superset of the C headers. |
| R5 | **The Original-filtering wall makes the overlay invisible for some users, silently.** | High | Medium | Detect it: if pspemu is foreground and the display hook has not fired for N seconds, the overlay is disabled — surface the instruction ("Adrenaline Settings → Graphics Filtering → anything but Original") rather than doing nothing. This is exactly what reVita's README does. Put it in the install docs above the fold. |
| R6 | **The PSX RAM base (`0x09800000`) is wrong on Vita's POPS build.** | Medium | Medium (T3 only) | Killed offline by spike S3's savestate diff before any device code is written. Falls back cleanly to T2 (`.VMP`), which is a supported product state, not a failure. |
| R7 | **Blit cost drops DW1's framerate.** vita2d display buffers are `USER_CDRAM_RW`; CPU writes to CDRAM are uncached. | Medium | Medium | Measured in spike S5 *before* the UI is designed. Panel-sized (not fullscreen) composite buffer; dirty-rect scanline copies; lazy re-composition (PSVshell's `g_gui_lazydraw_*` pattern). Ship a "HUD strip" default that consumes nothing and never masks the pad, with the full panel as an explicit "I stopped to think" mode. |
| R8 | **Input conflicts.** `L+SELECT` is POPS fast-forward; PS double-click exits to LiveArea; kernel `SceCtrl` hooks cannot gate Adrenaline's own `kuCtrlPeekBufferPositive` syscall path. | Medium | Medium | Gate the *guest* at `sceCompatWriteSharedCtrl` (in-process, verified as Adrenaline's own mechanism), not at the SceCtrl user library. Choose the combo from spike S4's results, avoiding both known collisions. Prefer rear touch if S4 shows it is free. Keep the open-panel input vocabulary tiny (↑↓ / ✕ / ○) — the moment you need L1/R1 you have stolen the shoulder buttons from a game that uses them. |
| R9 | **Licensing.** The most-recommended reference (`Electry/PSVshell`) is unlicensed = all rights reserved; the usable alternatives are GPL-3.0; `digislots-vita` has no LICENSE. | Certain | High (legal, not technical) | Decide before Phase 6 line one: write the blitter fresh, or adopt GPL-3.0 and derive from `JSilva6/reVita-extended`. Add a LICENSE file to this repo regardless. Also check the redistributability of the 63 Digimon sprites, which appear to be ripped/traced game assets. |
| R10 | **Maintenance burden.** Two front-ends, three build flavours of the core, a plugin coupled to a third-party CFW that ships monthly, and a data table that must stay in sync with a separate web app. | Certain | Medium | (a) The core/UI split means the *only* per-front-end code is the `ds_source_t` implementation and the renderer. (b) `test_table.c` asserts the C table against a checked-in CSV extracted from SydMontague's guide, so the five-rotated-rows class of bug can never recur silently — that drift is exactly how it happened. (c) The plugin fails **soft** in every unknown condition (unknown module_nid, no frames seen, RAM read looks implausible) rather than fighting to work. (d) The plugin is optional: the VPK is always the shipped baseline, and a broken plugin costs the user nothing but a deleted config line. |
| R11 | **Testing without hardware.** There is no debugger on device; iteration is minutes per cycle. | Certain | Medium | (a) `core/` is ~80% of the logic and is 100% host-testable under ASan/UBSan against checked-in `.vmp` fixtures — that's where every parsing and rule bug lives. (b) `dsdump card.vmp --json` answers save-format questions in seconds on a laptop and doubles as the test oracle. (c) The savestate route (S3) turns the hardest unknown — the PSX RAM map — into a PC-side binary diff. (d) On device, `sceIoWrite` logging to `ux0:data/digislots/digislots.log` is the only instrument; build it in from spike S1 and never remove it. (e) Vita3K can validate the standalone app's UI, but **not** Adrenaline/POPS — the overlay is device-only, which is why the spikes come first and are budgeted separately. |
| R12 | **Injection doesn't happen** — `*PSPEMUCFW` has zero observed uses in the wild, and ABM bubbles report a different titleid. | Medium | High (blocks the whole overlay) | Spike S1 tests both `*PSPEMUCFW` and `*ALL` + a `ScePspemu` main-module guard (the pattern with actual shipped precedent, `LiEnby/NoPspEmuDrm`). Third fallback: a small `*KERNEL` loader using the `psonescrot` un-inhibit trick — note that `ScePspemu` actively inhibits module loading, so any *load-on-demand* injection needs `sceKernelInhibitLoadingModule` hooked and returning 0. Config.txt-time injection (`start_preloaded_modules`) runs *before* the inhibit and is unaffected. |
| R13 | **Load ordering** — no guarantee `digislots.suprx` starts after `adrenaline_user.suprx`, and hooking an import of a module that has not loaded yet fails. | Low–Medium | Medium | Verifiers disagree on whether taiHEN's config-time load provably runs after Adrenaline's injection. Do not depend on it: resolve `ScePspemu` yourself via `taiGetModuleInfo("ScePspemu", &info)` + `module_nid` and derive `text_addr` (the `ARK-4 ps1cfw_enabler.c:45-53` pattern), and if `taiHookFunctionImport` on `AdrenalineUser` returns < 0 at `module_start`, retry from the worker thread for a few seconds before giving up. Also log which module-name spelling resolved (`AdrenalineUser` per `user/exports.yml`, vs `adrenaline_user` the filename) — the two verifiers disagree and this is cheap to settle at runtime. |
