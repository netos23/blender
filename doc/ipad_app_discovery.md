# iPad App Discovery: Opportunities and Blockers

Status: discovery / pre-planning. Based on a read-through of this repository at
`180be4ab` ("Fix: EEVEE: Vector Transform functions name mismatch").

## TL;DR

- **No iOS/iPadOS target exists today.** No CMake platform file, no UIKit code, no
  `TARGET_OS_IPHONE` guards. Everything Apple-specific assumes macOS/AppKit.
- **The hardest part is already done: rendering.** Blender has a mature Metal backend for the
  viewport (`source/blender/gpu/metal`, ~24k lines) and Cycles has a Metal device that only
  targets Apple Silicon with unified memory, the same GPU family as M-series iPads.
- **The main work is the platform layer.** GHOST (the windowing/input layer) has a Cocoa
  implementation (~5.3k lines) that needs a UIKit counterpart. Touch and Apple Pencil input
  have to be mapped onto an input model built for mouse and keyboard.
- **App Store rules are the biggest non-technical risk.** Embedded Python running
  user-supplied scripts and add-ons, runtime downloads of extensions, `fork()/execv()` and the
  JIT used by OSL all conflict with iOS sandbox or App Review rules.
- **Recommended path:** a staged prototype. (1) Headless/background Blender on iPadOS,
  (2) a UIKit GHOST backend with Metal viewport and Pencil input, (3) a touch-first UI layer.
  Distribute outside the App Store (TestFlight / EU alternative marketplaces) until the
  scripting policy question is settled.

---

## 0. Decisions (round 2)

| Topic | Decision | Rationale |
|---|---|---|
| Licensing / channel | **TestFlight first.** Dual licensing is not possible for the fork. | See §0.1. |
| Python | **Keep CPython, embedded and statically linked.** Do not replace it. | CPython already is an interpreter. Blender's UI is written in Python. See §0.2. |
| `fork`/`exec` | **Threads plus in-process replacements.** No subprocesses. | The C code barely depends on subprocesses on Apple. See §0.3. |
| Input | **Keyboard and trackpad work day one. Apple Pencil gets first-class support.** | See §0.4. |
| Hardware / OS | **M-series iPads only (M1+). Deployment target iPadOS 18, built with the latest SDK, tested on 18 and 26.** | See §0.5. |
| Scope | **Full Blender** (all editors and modes), not a cut-down companion app. | See §0.6. |

### 0.1 Dual license? No. Plan for TestFlight, with caveats
- Only **copyright holders** can offer a second license. Blender is `GPL-2.0-or-later` with
  thousands of contributors and no copyright assignment, so a fork cannot relicense their code.
  You *can* dual-license code you write yourself (for example a new `GHOST_SystemUIKit`), but
  the combined app is still GPL.
- **TestFlight is not a licensing loophole.** Testers still install through Apple's terms, and
  external TestFlight builds go through Beta App Review. Guideline 2.2 expects them to be
  "intended for public distribution" and to follow the App Review Guidelines, so the Python and
  code-download rules (§0.2) still apply. The GPL risk is a copyright holder objecting, which is
  what happened to VLC. In practice this risk is lower for a free, source-published beta, but it
  is not zero.
- **Operational limits:** 10,000 external testers, builds expire after 90 days (so you need a
  steady release cadence), and a paid Apple Developer account.
- **Ways to reduce risk:** publish full source for every build (GPL §6), add no extra
  restrictions, ask the Blender Foundation for a statement of no objection, and keep an **EU
  alternative marketplace** (for example AltStore PAL, which already hosts GPL apps) and **Ad
  Hoc builds** (up to 100 iPads for internal testers) as fallbacks.
- *Not legal advice. Get a real legal opinion before any public build.*

### 0.2 Python: keep CPython (it is an interpreter). Control what code gets in
- The App Store problem is not interpreters. Pythonista, Pyto, Carnets and a-Shell ship CPython.
  The problems are **(a) JIT / writable+executable memory** and **(b) downloading code that adds
  features** (Guideline 2.5.2). CPython has no JIT unless you enable 3.13+'s experimental one, so
  keep it off. **CPython 3.13+ officially supports iOS (PEP 730).**
- **Python cannot be dropped or swapped for a different language.** The whole UI layout (menus,
  panels, headers) is Python: `scripts/startup/bl_ui/` has 81 modules. So do all operators in
  `scripts/startup/bl_operators/`, all bundled add-ons, and the `bpy` API that drivers and
  `.blend` scripts use. A replacement interpreter would mean rewriting every one of them.
- **What has to change:**
  - Link CPython statically. Ship native modules (`numpy`, `_ssl`, `_ctypes`, ...) as signed
    `.framework`s inside the app bundle. BeeWare's `Python-Apple-support` is the reference.
  - No `pip` and no runtime wheels. Remove or disable the online extension repository in
    `scripts/addons_core/bl_pkg` and the downloader in `_bpy_internal/http/downloader.py`.
  - Allow **pure-Python** add-ons and user scripts that the user brings in through the Files app,
    editable in the Text Editor. This is the Pythonista-style "user-authored code" model. Keep
    `WITH_PYTHON_SECURITY` on, so scripts embedded in `.blend` files need explicit consent.

### 0.3 Replacing `fork`/`exec`: threads, not background processes
iPadOS has no API to launch a child process, and "background tasks" only give *your own process*
extra run time. The audit shows the dependency is small:

| Use | Location | iOS replacement |
|---|---|---|
| GL shader compile subprocess (`BLI_subprocess.cc`) | `gpu/opengl/*` | Not built. iOS uses Metal, which already compiles shaders on worker threads (`mtl_shader.mm`, `GPU_max_parallel_compilations()`). |
| Move to trash via `gio`/`kioclient` (`fileops_c.cc`) | Linux-only path | Not built. Use `NSFileManager trashItemAtURL` or delete directly. |
| Open file/URL externally (`wm.py`, `file.py`, `image.py`) | Python `subprocess` → `open` | `UIApplication.open` / `UIDocumentInteractionController` / share sheet, called from a new `bpy` or GHOST hook. |
| Play rendered animation (`screen_play_rendered_anim.py`) | Launches a second Blender | Play inside the Image or Sequencer editor, or use AVPlayer. |
| External text editor, system info, i18n tools | Python `subprocess` | Disable on iOS. |
| Heavy compute (render, bake, simulation, remesh) | Already threads (TBB / `BLI_task`) | Nothing to change. Add memory-pressure handling. |
| Long render while the app is in the background | n/a | iPadOS 26: `BGContinuedProcessingTask` keeps *this* process running with user-visible progress. iPadOS 18: `beginBackgroundTask` (about 30 s) to finish the current tile, then autosave and pause. Resume when the app returns. |

### 0.4 First-class Apple Pencil (keyboard and trackpad also work)
Keyboard and trackpad reuse the existing Cocoa-style paths (`UIKey` → `GHOST_kEventKey*`,
`UIPointerInteraction` + indirect pointer touches → cursor and buttons, trackpad pinch/rotate/scroll
→ `GHOST_kTrackpadEvent*`). Pencil work, by feature:

| Pencil feature | UIKit API | Blender mapping | Notes |
|---|---|---|---|
| Pressure | `UITouch.force / maximumPossibleForce` | `GHOST_TabletData.Pressure` | Expose a pressure curve. Blender already has one in Preferences > Input > Tablet. |
| Tilt | `altitudeAngle`, `azimuthAngle(in:)` | `Xtilt`, `Ytilt` (projected vector, −1..1) | Same convention as the comment in `GHOST_Types.hh`. Watch the Y sign (Cocoa already flips it). |
| 240 Hz samples | `coalescedTouches(for:)` | Emit each one as a cursor event. The WM already turns earlier queued moves into `INBETWEEN_MOUSEMOVE`, which paint and sculpt strokes consume. | This is the key to smooth strokes. |
| Low latency | `predictedTouches(for:)` | Optional preview-only stroke extension | Never commit predicted points. |
| Late force / azimuth | `estimatedPropertiesExpectingUpdates` + `touchesEstimatedPropertiesUpdated` | Either delay stroke samples slightly, or accept the first estimate. | Start by accepting the estimate. |
| Hover (M2+ Pro, M2+ Air) | `UIHoverGestureRecognizer`, `zOffset` | Cursor move with `Active = Stylus`, `Pressure = 0` | Gives brush-cursor preview and tooltips, which fixes the biggest "no hover" UX gap. |
| Barrel roll (Pencil Pro) | `UITouch.rollAngle` | **New** `GHOST_TabletData` field (for example `Twist`) → `wmTabletData` | Brush and texture rotation, Grease Pencil nib angle. Wacom Art Pens could later feed it on desktop too. |
| Squeeze (Pencil Pro) | `UIPencilInteraction` squeeze | Configurable: open a pie menu or brush picker at the pen tip | Make it a keymap item so users can rebind it. |
| Double-tap | `UIPencilInteraction` tap | Toggle eraser / last tool / undo | Respect the system `preferredTapAction`. |
| Eraser semantics | n/a (Pencil has no eraser end) | Double-tap or squeeze toggles `GHOST_kTabletModeEraser` | Grease Pencil and texture paint already honor eraser mode. |
| Palm rejection | `UITouch.type` `.pencil` vs `.direct` | While the pen is down, drop `.direct` touches. Fingers navigate, pen acts. | Add a preference: "Only Pencil draws". |
| Haptics (Pencil Pro) | `UICanvasFeedbackGenerator` | Snap and increment feedback | Nice to have. |

Priority workspaces: **Sculpt**, **Grease Pencil draw/animate**, **Texture paint**. These already
use tablet pressure and tilt, so the gain is highest there.

### 0.5 M-series only
- **Devices:** iPad Pro (M1 2021 and later), iPad Air (M1 2022 and later). 8 GB+ RAM on all of them,
  16 GB on 1 TB+ Pro models.
- **Minimum OS: iPadOS 18** (`IPHONEOS_DEPLOYMENT_TARGET=18.0`), built against the latest SDK.
  Every M-series iPad runs 18 and 26. Everything the plan relies on already exists in 18:

  | API | Available since | On iPadOS 18 |
  |---|---|---|
  | Metal 3, `MTLGPUFamilyApple7+`, ray-tracing intersection | 16 | ✅ |
  | Pencil hover (`UIHoverGestureRecognizer`, `zOffset`) | 16.1 | ✅ |
  | Pencil Pro squeeze, `rollAngle`, `UICanvasFeedbackGenerator` | 17.5 | ✅ |
  | CPython official iOS support (PEP 730, 3.13+) | 3.13 | ✅ (iOS 13+) |
  | `BGContinuedProcessingTask` (long renders in the background) | **26** | ❌ Use the `@available` fallback from §0.3 |
  | New iPadOS 26 windowing (free-form windows, menu bar) | **26** | ❌ 18 uses Stage Manager and Split View. Both work through `UIScene` if the GHOST window follows the scene size and does not assume fixed sizes. Add `UIMenuBuilder` menus on 26. |

  Guard 26-only calls with `if (@available(iOS 26.0, *))`. CI needs both an 18.x and a 26.x
  device or simulator run.
- **Why this simplifies things:**
  - One GPU family (`MTLGPUFamilyApple7`+), always unified memory. This fits Cycles' existing
    Apple-Silicon-only filter.
  - G1 (managed storage) is solved by always using `MTLStorageModeShared`.
  - G2 becomes "treat iPad like an Apple Silicon Mac".
  - MetalRT is available on Apple9 (M3/M4 iPads) and falls back to Cycles' BVH elsewhere.
- **Tiering:** Pencil Pro features (squeeze, barrel roll, haptics) need M2+ Air or M4+ Pro. Hover
  needs M2+. Detect these at runtime and do not gate the app on them.
- Request `com.apple.developer.kernel.increased-memory-limit` and test on 8 GB M1 running
  **iPadOS 18** as the floor device. That is the worst case for both memory and APIs.

### 0.6 Full Blender scope
Goal: the same editors, modes and file compatibility as desktop. The only differences should be
features the platform makes impossible. Keyboard and trackpad give desktop parity. Touch and
Pencil are added on top, not a replacement UI.

**Dependency plan for full scope** (`build_files/build_environment/cmake`):

| Status | Libraries | Notes |
|---|---|---|
| ✅ Port as-is (arm64-iOS cross-compile) | zlib, zstd, png, jpeg, webp, tiff, openjpeg, openjph, freetype, harfbuzz, fribidi, expat, pugixml, xml2, fmt, tbb, imath, openexr, OpenColorIO, OpenImageIO, OpenSubdiv, Embree (NEON via `sse2neon`), OpenImageDenoise (arm64 uses BNNS/Accelerate, which exists on iOS), OpenPGL, Alembic, Draco, fftw, gmp, manifold, potrace, sndfile, ogg/vorbis/flac/opus, sqlite, ssl, yamlcpp, thorvg, haru, meshoptimizer, libheif, brotli, deflate, blosc, lzma | Mostly autotools/CMake with an iOS toolchain file. Build **static** or as embedded frameworks. |
| 🟠 Port with effort | **Python + numpy** (static CPython, numpy built for iOS), **USD** (large; plugin system uses `dlopen`, so link plugins statically or embed them as frameworks), **OpenVDB / NanoVDB**, **MaterialX**, **FFmpeg** (+ x264/x265/vpx/aom/theora/lame; GPL is fine because the app is GPL, but codec patents are a risk, so prefer VideoToolbox hardware encode/decode through FFmpeg), **OpenAL** (use Apple's built-in OpenAL or switch the Audaspace backend to CoreAudio), Rubberband, Ceres | Budget most of Phase 1 here. |
| ❌ Not possible / not applicable | **LLVM + OSL** (JIT is forbidden. Cycles SVM still runs every built-in node; only OSL Script nodes and OSL-only features are lost), **OpenXR / VR** (no runtime on iPad), **Vulkan, epoxy/OpenGL, shaderc** (Metal only), **SDL, Wayland, X11, dbus, JACK, spnav/3Dconnexion**, **CUDA/HIP/oneAPI/level-zero** (Metal only) | Turn off with the existing `WITH_*` options. The UI already hides these features when they are off. |

**What "full" costs:**
- **Size:** expect 500 MB – 1 GB installed. Ship essential assets in the bundle. Put optional
  content (the full asset library, extra fonts, locale files) in On-Demand Resources or the
  Background Assets framework.
- **Memory:** full scenes on an 8 GB iPad are the real limit, not the feature list. Hook
  `didReceiveMemoryWarning` into Blender to reduce undo steps, free GPU texture caches and trim
  image buffers.
- **UX coverage:** every editor must be usable, so the touch work is systemic:
  - a global touch keymap
  - a larger UI scale preset
  - long-press opens the context menu
  - an on-screen modifier bar for Ctrl/Shift/Alt
  - two-finger navigation in every 2D and 3D editor
  - Pencil-specific behavior in paint, sculpt and Grease Pencil
- **Testing:** run Blender's existing `tests/` (Python and GTest) on device in headless mode.
  This keeps parity measurable.

### 0.7 Prior art: `Shlok-Bhakta/blender-ios-build`
Reviewed at `ad79172a` (2026-09-14). It is a Blender **5.2.0** fork. This repo is `main` (5.3 alpha).
Its docs say it adapts an earlier **`ios` branch of Blender** ("donor" `a1de44dd`, a 213-file
delta against v5.1.2) one subsystem at a time instead of merging it. We have not yet verified
where that branch lives or how active it is.

**It has already made the same decisions we did:** full-Blender profile (`blender_full` minus
platform limits), iOS 18.0 deployment target, static CPython 3.13 with native modules packaged as
frameworks, OSL off, Metal only, threads instead of subprocesses.

| Area | Their state | vs. our plan |
|---|---|---|
| Build | `platform_ios.cmake`, `blender_ios_{device,sim}*.cmake`, iOS dependency recipes and patches (Embree, OCIO, USD, ...), content-addressed dependency cache, GitHub Actions on `macos-15` | Covers Phase 1a/1b. |
| GHOST | `GHOST_SystemIOS` / `WindowIOS` / `ContextIOS` (~7k lines), a single `UIWindowScene`, MTKView drawable as the pixel authority, hardware keyboard, trackpad, software keyboard bridge, `UIDocumentPicker` with security-scoped files | Covers most of Phase 2. |
| Touch | 1 finger = left mouse, 2 fingers = orbit, pinch = zoom, 3 fingers = pan, double tap = right click | A reasonable baseline for §0.4. |
| Pencil | Pressure (`force/maximumPossibleForce`), tilt (azimuth/altitude), hover, Pencil-vs-finger separation, double-tap = right click | **Basic only.** Missing: `coalescedTouches` (240 Hz), `predictedTouches`, estimated-property updates, `rollAngle`, squeeze, eraser toggle, pressure curve, "only Pencil draws". |
| GPU | Metal viewport with iOS capability flags (`mtl_platform_ios.hh`), EEVEE and Workbench on the simulator. Cycles CPU proven. Cycles Metal compiled but only enabled on tier-2 GPUs (M-series qualifies) | Matches §0.5. |
| Python | CPython 3.13.13, NumPy 2.3.4, zstandard. The extensions downloader was moved to a worker thread | Matches §0.2/§0.3. It keeps online extensions working, which carries App Review risk under 2.5.2. |
| Missing | Memory-warning handling, background-task handling (none found), USD Hydra (off), OSL | Gaps from §0.3/§0.6. |
| **Validation** | **Simulator only.** Their device IPA is deliberately **unsigned** and **has never been launched on real hardware.** Pencil, Cycles Metal, memory/jetsam and thermal behavior are all open "owner-signed" gates. | **Real-device validation is the biggest open gap, and we are well placed to close it** (paid developer account plus TestFlight). |

**What it means for us:**
1. **Do not start from scratch.** Our Phase 1a/1b and most of Phase 2 already exist.
2. **Pick a base:**
   - **(A) Fork theirs.** Fastest route to a device build. We inherit 5.2.0 and their process
     tooling, which is tied to one specific build Mac.
   - **(B) Forward-port their shims onto this repo's `main`.** Their design keeps iOS logic in
     separate `*IOS*` files with small upstream hooks (see `PORT_MAP.tsv`, 214 files), so this is
     feasible but costs weeks.
   - **(C) Contribute upstream to them** (or to the official `ios` branch).
3. **Our value-add, in order:** signed device builds and TestFlight → real-device gates
   (Pencil, Metal Cycles, memory) → first-class Pencil from §0.4 → memory and background handling →
   a policy-safe extensions mode.

---

## 1. Current state in the codebase

| Area | Where | iPad relevance |
|---|---|---|
| Build config | `build_files/cmake/platform/platform_apple.cmake` | macOS only. Links `AppKit`, `Cocoa`, `Carbon`, `IOKit`, `CoreServices`, `OpenGL`. A `platform_ios.cmake` would be new. |
| Windowing and input | `intern/ghost/intern/GHOST_SystemCocoa.mm`, `GHOST_WindowCocoa.mm`, `GHOST_WindowViewCocoa.hh` | All `NSApplication` / `NSWindow` / `NSView` / `NSEvent`. Needs a UIKit rewrite. |
| Backend selection | `intern/ghost/intern/GHOST_ISystem.cc` | Selects Cocoa / SDL / Wayland / X11 / Headless at compile time. A new `GHOST_SystemUIKit` fits in here cleanly. |
| GPU context | `intern/ghost/intern/GHOST_ContextMTL.mm` | Builds on `CAMetalLayer` (available on iOS) plus `NSView` (not available). A small change. |
| Viewport GPU backend | `source/blender/gpu/metal/*` | Mostly portable Metal. Includes `<Cocoa/Cocoa.h>`, asserts `GPU_OS_MAC`, checks `MTLGPUFamilyMac2`, and uses `MTLStorageModeManaged`, which does not exist on iOS. |
| Cycles | `intern/cycles/device/metal/*` | Already Apple Silicon only (`hasUnifiedMemory`, Apple GPU families, MetalRT on Apple9+). No managed-storage use. Gated by `@available(macos 13.0, *)`. |
| Pen input | `GHOST_SystemCocoa.mm::handleTabletEvent` | Pressure, tilt and eraser are already modeled in `GHOST_TabletData`. Pencil maps onto this directly. |
| Gestures | `GHOST_Types.hh` (`GHOST_kTrackpadEventScroll/Rotate/Magnify/SmartMagnify`) and WM `MOUSEPAN/MOUSEZOOM/MOUSEROTATE` | Gesture events already reach the window manager and keymaps. They can be reused for touch pinch, rotate and pan. |
| Headless | `GHOST_SystemHeadless.hh`, `WITH_HEADLESS` | Allows a first port with no UI. |
| Subprocess | `source/blender/blenlib/intern/BLI_subprocess.cc` (`fork()` + `execv()`), GL shader compile subprocess | Not allowed on iOS. Must be compiled out. |
| Paths | `GHOST_SystemPathsCocoa.mm` | `NSSearchPathForDirectoriesInDomains` + `NSBundle` work on iOS, but the directory layout differs and there is no `/Library/...`. |
| Other `.mm` files | `fsmenu_system_macos.mm`, `messages_apple.mm`, `storage_apple.mm`, `fileops_apple.mm`, `blendthumb` | Small, mostly Foundation-based. Swap the `Cocoa.h` includes for `Foundation.h` and guard the AppKit-only calls. |
| Precompiled libs | `lib/macos_arm64` (git submodule) | No iOS slice. About 80 dependencies in `build_files/build_environment/cmake` would need arm64-iOS builds. |

---

## 2. Opportunities

### 2.1 Product
1. **Apple Pencil as a first-class input.** Sculpting, texture painting, Grease Pencil drawing
   and 2D animation are the workflows that benefit most. Pressure and tilt already exist in
   `GHOST_TabletData`. Pencil also adds hover (M2+ iPads), barrel roll (Pencil Pro), squeeze and
   double-tap, which can map to brush size, pie menus or tool switching.
2. **A "companion" scope beats full parity.** Useful first slices:
   - Grease Pencil storyboarding and 2D animation.
   - Sculpt mode (a single-mode, gesture-friendly workspace).
   - Viewing and reviewing `.blend` files (orbit, scrub the timeline, EEVEE/Cycles preview) for
     directors and clients.
   - Asset library browsing and scene layout.
3. **Round-trip with desktop.** `.blend` is platform-independent, so iCloud Drive / Files app
   integration lets people start on iPad and finish on desktop with no conversion step.
4. **Market gap.** Nomad Sculpt, Procreate Dream and Forger show there is demand for pro 3D and
   animation on iPad. None of them offers a full DCC with Blender's open file format and
   ecosystem.

### 2.2 Technical leverage
1. **The Metal viewport exists.** EEVEE and Workbench already run on Apple Silicon GPUs. iPad
   M-series chips are the same GPU architecture family as M-series Macs.
2. **Cycles Metal is Apple Silicon only.** The device filter already requires Apple GPUs with
   unified memory, which is exactly the iPad hardware profile. MetalRT hardware ray tracing
   is enabled on Apple9+ (M3/M4-class) GPUs, which recent iPad Pros have.
3. **GHOST is a clean seam.** Blender already maintains 5+ windowing backends behind one
   interface (`GHOST_ISystem`, `GHOST_IWindow`, `GHOST_IContext`). A UIKit backend is additive.
   It does not require a fork.
4. **The gesture and tablet event plumbing exists.** Trackpad magnify/rotate/scroll and tablet
   pressure/tilt are already understood by the window manager and keymaps.
5. **Stage Manager and external displays.** iPadOS supports resizable windows and external
   monitors with keyboard and trackpad. With a Magic Keyboard, a large share of the existing
   desktop keymap and UI "just works".
6. **Headless as a stepping stone.** `WITH_HEADLESS` + `-b` (background) lets you validate
   the core (DNA/RNA, depsgraph, modifiers, geometry nodes, Cycles) on-device before any UI
   work.
7. **Build options for trimming.** Most heavy or at-risk features already have `WITH_*`
   switches (`WITH_CYCLES_OSL`, `WITH_LLVM`, `WITH_OPENVDB`, `WITH_USD`, `WITH_XR_OPENXR`,
   `WITH_JACK`, `WITH_CYCLES_EMBREE`, the FFmpeg codecs, and more). This keeps binary size and
   dependency work under control.

---

## 3. Blockers and risks

Severity: 🔴 must solve before any shippable build · 🟠 significant effort or risk · 🟡 manageable.

### 3.1 Platform and build
| # | Blocker | Severity | Notes |
|---|---|---|---|
| B1 | **No iOS toolchain or platform CMake.** | 🔴 | Needs `platform_ios.cmake` (iphoneos SDK, deployment target, code signing, `.app` bundle layout, Info.plist, entitlements) and Xcode generator support (`platform_apple_xcode.cmake` is macOS-oriented). |
| B2 | **Dependencies not built for iOS.** | 🔴 | About 80 libs in `build_files/build_environment`. Some are easy (zlib, png, jpeg, freetype, harfbuzz, tbb, OpenImageIO, OpenEXR, OpenColorIO, OpenSubdiv, Embree). Some are hard or should be dropped at first: Python (needs static embedding, see B9), LLVM/OSL, FFmpeg (licensing plus size), USD, OpenVDB, MaterialX, OpenXR, JACK, SDL, spnav. |
| B3 | **AppKit is linked and included everywhere Apple code lives.** | 🟠 | `-framework AppKit -framework Cocoa -framework Carbon` in `platform_apple.cmake`. `#include <Cocoa/Cocoa.h>` in `mtl_backend.mm`, `mtl_context.hh`, `mtl_memory.hh`, `messages_apple.mm`. Mostly mechanical: switch to `Foundation`/`Metal`/`QuartzCore` and guard with `TARGET_OS_IOS`. |
| B4 | **App size.** | 🟡 | The desktop bundle is several hundred MB. The App Store cellular download limit is soft, but on-demand resources or stripped features are advisable. |

### 3.2 Graphics
| # | Blocker | Severity | Notes |
|---|---|---|---|
| G1 | **`MTLStorageModeManaged` is macOS-only.** | 🟡 (M-series: always Shared) | Used in `mtl_vertex_buffer.mm` and `mtl_storage_buffer.mm`, with `didModifyRange` sync paths (~23 references). On iOS, use `Shared` (unified memory) and remove the manual sync. Apple Silicon Macs could share this path too. |
| G2 | **The backend is hard-gated to macOS.** | 🟡 | `BLI_assert_msg(os == GPU_OS_MAC, ...)` and `supportsFamily:MTLGPUFamilyMac2` checks in `mtl_backend.mm` (supported-GPU check, barycentrics whitelist). Needs `MTLGPUFamilyApple7+` equivalents for iPad and a new `GPU_OS_IOS`. |
| G3 | **Runtime shader compilation.** | 🟡 | `newLibraryWithSource` is used for the viewport and Cycles kernels. This is allowed on iOS (it is not a CPU JIT), but first-launch compile times on iPad will be long. Consider shipping precompiled `.metallib` archives / binary archives. |
| G4 | **Memory limits.** | 🟠 | iPadOS terminates apps well below physical RAM (jetsam). The `com.apple.developer.kernel.increased-memory-limit` entitlement helps. Large scenes, Cycles BVH builds and undo stacks need memory pressure handling. |
| G5 | **Thermal and battery.** | 🟡 | Sustained Cycles rendering throttles. Default to EEVEE/Workbench and lower viewport sample counts. |

### 3.3 Windowing and input
| # | Blocker | Severity | Notes |
|---|---|---|---|
| W1 | **No UIKit GHOST backend.** | 🔴 | Needs `GHOST_SystemUIKit`, `GHOST_WindowUIKit` and a `UIView`/`CAMetalLayer` host covering the app lifecycle (`UIApplicationDelegate`/`UISceneDelegate`), multi-window via scenes, safe areas, DPI, clipboard (`UIPasteboard`), drag and drop, IME (`UITextInput`), cursors (`UIPointerInteraction`). Reference: `GHOST_SystemCocoa.mm` (2.2k lines), `GHOST_WindowCocoa.mm` (1.3k lines). |
| W2 | **The event loop inverts control.** | 🟠 | Blender owns its main loop (`WM_main` → `processEvents`). UIKit owns the run loop. Options: drive Blender from `CADisplayLink`, or run the WM on a secondary thread and marshal UI events to it. The macOS backend pumps `NSApp` manually. UIKit is less forgiving. |
| W3 | **The UI is built for a mouse.** | 🔴 (UX) | No hover without Pencil, small hit targets, right-click menus, modifier-key-heavy keymaps (Ctrl/Shift/Alt + click), middle-mouse orbit. Needs a touch keymap, larger UI scale presets, on-screen modifier keys, and long-press → context menu. This is the largest *product* effort. |
| W4 | **Multi-touch is not a GHOST concept.** | 🟠 | GHOST has single-pointer + tablet + trackpad gestures, with no multi-touch point stream. Minimum: map 2-finger pan/pinch/rotate to the existing `GHOST_kTrackpadEvent*` → `MOUSEPAN/MOUSEZOOM/MOUSEROTATE`. Later: a proper touch event type for gesture-rich tools. |
| W5 | **Pencil palm rejection and simultaneous touch + pen.** | 🟡 | UIKit provides `UITouch.type` (`.pencil` vs `.direct`). Blender needs a policy: pen draws, fingers navigate. |
| W6 | **File dialogs and the sandbox.** | 🟠 | Blender's own file browser (`space_file`, `fsmenu_system_macos.mm`) walks the filesystem freely. On iPad, anything outside the app container needs `UIDocumentPickerViewController` plus security-scoped bookmarks. Linked libraries, relative paths and "Open Recent" all need rework. |
| W7 | **NDOF / 3Dconnexion, OpenXR, JACK.** | 🟡 | `dlopen` of `/Library/Frameworks/3DconnexionClient.framework` and friends. Compile these out. |

### 3.4 Policy and runtime (App Store)
| # | Blocker | Severity | Notes |
|---|---|---|---|
| P1 | **Embedded Python running downloaded code.** | 🟠 → resolved in §0.2 | App Review Guideline 2.5.2 forbids downloading and executing code that changes app functionality. Add-ons, the extensions platform (`scripts/addons_core/bl_pkg`), scripts embedded in `.blend` files (`WITH_PYTHON_SECURITY`) and drivers are all Python. Options: ship only bundled add-ons, disable remote extension install, and keep user scripting in an "educational/IDE"-style carve-out (Pythonista/Swift Playgrounds precedent). Needs a legal/App Review read before committing. |
| P2 | **Python C extensions and wheels.** | 🔴 | iOS forbids `dlopen` of arbitrary unsigned dylibs. CPython 3.13+ has official iOS support (PEP 730), but every native module (numpy and the like) must be built as a signed framework inside the bundle. `pip`/wheel install at runtime is not possible. |
| P3 | **`fork()` / `execv()`.** | 🟡 → see §0.3 | `BLI_subprocess.cc` and the GL shader compilation subprocess. Also Python's `subprocess`, used by some add-ons and the extensions system. `fork` is unavailable to App Store apps. Compile out and fall back to threads. |
| P4 | **CPU JIT (LLVM / OSL).** | 🔴 | iOS disallows writable+executable memory for third-party apps. Build with `WITH_CYCLES_OSL=OFF` and `WITH_LLVM=OFF`. Cycles SVM shading still works, so the impact is small. |
| P5 | **Licensing: GPL v3 and the App Store.** | 🔴 → TestFlight, see §0.1 | Blender is GPL. App Store terms (DRM, usage restrictions) are widely considered incompatible with GPL. This is why VLC was pulled in 2011. Blender Foundation and all copyright holders would have to agree, or distribution would have to use TestFlight / enterprise / EU alternative marketplaces (DMA). **Get a legal opinion before anything else.** |
| P6 | **Codec licensing.** | 🟡 | FFmpeg with x264/x265 raises patent and licensing questions. Use `AVFoundation`/`VideoToolbox` for video I/O on iOS instead. |
| P7 | **Background execution.** | 🟡 | iPadOS suspends backgrounded apps, so long renders stop. Use `BGContinuedProcessingTask` on iPadOS 26, `beginBackgroundTask` + pause on iPadOS 18. Autosave on `sceneDidEnterBackground`. |

---

## 4. Suggested phased plan

| Phase | Goal | Key work | Exit criteria |
|---|---|---|---|
| 0. Validate (1–2 wks) | De-risk licensing and policy | Legal read on GPL + App Store (P5), App Review stance on Python (P1). Choose a distribution channel. | Go/no-go on channel. |
| 1a. Headless core (4–6 wks) | Blender builds and runs `-b` on iPad | `platform_ios.cmake` (deployment target 18.0), iOS builds of the ✅ dependencies, `WITH_HEADLESS`, static CPython, `BLI_subprocess` compiled out. | Load a `.blend`, evaluate the depsgraph, render one Cycles frame on an iPadOS 18 device. |
| 1b. Full dependency set (6–10 wks, overlaps phase 2) | Full-scope feature parity for the core | Port the 🟠 dependencies (numpy, USD, OpenVDB, MaterialX, FFmpeg, audio). Run Blender's `tests/` on device. | Test-suite pass rate on device matches macOS arm64, apart from the ❌ features. |
| 2. Metal viewport + UIKit GHOST (8–12 wks) | Interactive, desktop-parity UI on iPad | `GHOST_SystemUIKit`/`WindowUIKit`, `CAMetalLayer` context, fix `MTLStorageModeManaged` (G1) and the OS gating (G2), gestures → trackpad events, Pencil → `GHOST_TabletData`, keyboard/trackpad via Magic Keyboard. | EEVEE viewport at 60 fps on a mid-size scene. Sculpt and Grease Pencil usable with Pencil. |
| 3. Touch-first UX (ongoing) | Usable without a keyboard | Touch keymap, larger UI scale, on-screen modifier bar, long-press menus, Files-app document picker, iCloud, autosave and state restoration. | Usability test with 5 artists completing a sculpt or storyboard task. |
| 4. Ecosystem | Scripting and add-ons | Static Python + bundled add-ons, policy-compliant scripting mode, precompiled Metal archives. | Depends on the Phase 0 decision. |

---

## 5. Open questions
1. ~~Distribution~~: **TestFlight** (fallbacks: EU alternative marketplace, Ad Hoc). See §0.1.
2. ~~Target scope~~: **Full Blender.** See §0.6.
3. ~~Minimum hardware~~: **M1+, iPadOS 18 deployment target.** See §0.5.
4. Should the UIKit GHOST backend also target visionOS (shared UIKit and Metal base)?
5. Maintained upstream (merged behind `WITH_GHOST_UIKIT`), or as a downstream fork?
6. Will you ask the Blender Foundation for a statement of no objection before the first external TestFlight build?

## 6. Starting points for engineers
- `intern/ghost/intern/GHOST_ISystem.cc`: backend selection. Add the UIKit case here.
- `intern/ghost/intern/GHOST_SystemCocoa.mm` (`handleTabletEvent`, `handleMouseEvent`, trackpad gestures): the reference for event translation.
- `intern/ghost/intern/GHOST_ContextMTL.mm`: swap the `NSView` host for a `UIView`.
- `source/blender/gpu/metal/mtl_backend.mm`: OS/GPU family gating and capability detection.
- `source/blender/gpu/metal/mtl_vertex_buffer.mm`, `mtl_storage_buffer.mm`: managed storage mode.
- `intern/cycles/device/metal/util.mm`: `@available(macos 13.0, *)` device filter.
- `source/blender/blenlib/intern/BLI_subprocess.cc`: `fork`/`execv`.
- `build_files/cmake/platform/platform_apple.cmake`: the template for `platform_ios.cmake`.
