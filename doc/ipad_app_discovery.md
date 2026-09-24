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
| G1 | **`MTLStorageModeManaged` is macOS-only.** | 🟠 | Used in `mtl_vertex_buffer.mm` and `mtl_storage_buffer.mm`, with `didModifyRange` sync paths (~23 references). On iOS, use `Shared` (unified memory) and remove the manual sync. Apple Silicon Macs could share this path too. |
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
| P1 | **Embedded Python running downloaded code.** | 🔴 | App Review Guideline 2.5.2 forbids downloading and executing code that changes app functionality. Add-ons, the extensions platform (`scripts/addons_core/bl_pkg`), scripts embedded in `.blend` files (`WITH_PYTHON_SECURITY`) and drivers are all Python. Options: ship only bundled add-ons, disable remote extension install, and keep user scripting in an "educational/IDE"-style carve-out (Pythonista/Swift Playgrounds precedent). Needs a legal/App Review read before committing. |
| P2 | **Python C extensions and wheels.** | 🔴 | iOS forbids `dlopen` of arbitrary unsigned dylibs. CPython 3.13+ has official iOS support (PEP 730), but every native module (numpy and the like) must be built as a signed framework inside the bundle. `pip`/wheel install at runtime is not possible. |
| P3 | **`fork()` / `execv()`.** | 🔴 | `BLI_subprocess.cc` and the GL shader compilation subprocess. Also Python's `subprocess`, used by some add-ons and the extensions system. `fork` is unavailable to App Store apps. Compile out and fall back to threads. |
| P4 | **CPU JIT (LLVM / OSL).** | 🔴 | iOS disallows writable+executable memory for third-party apps. Build with `WITH_CYCLES_OSL=OFF` and `WITH_LLVM=OFF`. Cycles SVM shading still works, so the impact is small. |
| P5 | **Licensing: GPL v3 and the App Store.** | 🔴 | Blender is GPL. App Store terms (DRM, usage restrictions) are widely considered incompatible with GPL. This is why VLC was pulled in 2011. Blender Foundation and all copyright holders would have to agree, or distribution would have to use TestFlight / enterprise / EU alternative marketplaces (DMA). **Get a legal opinion before anything else.** |
| P6 | **Codec licensing.** | 🟡 | FFmpeg with x264/x265 raises patent and licensing questions. Use `AVFoundation`/`VideoToolbox` for video I/O on iOS instead. |
| P7 | **Background execution.** | 🟡 | iPadOS suspends backgrounded apps, so long renders stop. Use `BGProcessingTask` / `BGContinuedProcessingTask` (iPadOS 26) or warn the user. Autosave on `sceneDidEnterBackground`. |

---

## 4. Suggested phased plan

| Phase | Goal | Key work | Exit criteria |
|---|---|---|---|
| 0. Validate (1–2 wks) | De-risk licensing and policy | Legal read on GPL + App Store (P5), App Review stance on Python (P1). Choose a distribution channel. | Go/no-go on channel. |
| 1. Headless core (4–6 wks) | Blender builds and runs `-b` on iPad | `platform_ios.cmake`, iOS dep builds for a minimal set, `WITH_HEADLESS`, Python off or static, no OSL/LLVM/USD/VDB/FFmpeg, `BLI_subprocess` stubbed. | Load a `.blend`, evaluate the depsgraph, render one Cycles frame on-device (unit tests pass). |
| 2. Metal viewport + UIKit GHOST (8–12 wks) | Interactive, desktop-parity UI on iPad | `GHOST_SystemUIKit`/`WindowUIKit`, `CAMetalLayer` context, fix `MTLStorageModeManaged` (G1) and the OS gating (G2), gestures → trackpad events, Pencil → `GHOST_TabletData`, keyboard/trackpad via Magic Keyboard. | EEVEE viewport at 60 fps on a mid-size scene. Sculpt and Grease Pencil usable with Pencil. |
| 3. Touch-first UX (ongoing) | Usable without a keyboard | Touch keymap, larger UI scale, on-screen modifier bar, long-press menus, Files-app document picker, iCloud, autosave and state restoration. | Usability test with 5 artists completing a sculpt or storyboard task. |
| 4. Ecosystem | Scripting and add-ons | Static Python + bundled add-ons, policy-compliant scripting mode, precompiled Metal archives. | Depends on the Phase 0 decision. |

---

## 5. Open questions
1. Distribution: App Store, TestFlight/beta only, or EU alternative marketplace? (drives P1, P2, P5)
2. Target scope: full Blender, or a focused "Blender Sculpt / Grease Pencil" app from the same codebase?
3. Minimum hardware: M1+ iPads only (recommended: same GPU family as supported Macs and 8 GB+ RAM), or A-series too?
4. Should the UIKit GHOST backend also target visionOS (shared UIKit and Metal base)?
5. Maintained upstream (merged behind `WITH_GHOST_UIKIT`), or as a downstream fork?

## 6. Starting points for engineers
- `intern/ghost/intern/GHOST_ISystem.cc`: backend selection. Add the UIKit case here.
- `intern/ghost/intern/GHOST_SystemCocoa.mm` (`handleTabletEvent`, `handleMouseEvent`, trackpad gestures): the reference for event translation.
- `intern/ghost/intern/GHOST_ContextMTL.mm`: swap the `NSView` host for a `UIView`.
- `source/blender/gpu/metal/mtl_backend.mm`: OS/GPU family gating and capability detection.
- `source/blender/gpu/metal/mtl_vertex_buffer.mm`, `mtl_storage_buffer.mm`: managed storage mode.
- `intern/cycles/device/metal/util.mm`: `@available(macos 13.0, *)` device filter.
- `source/blender/blenlib/intern/BLI_subprocess.cc`: `fork`/`execv`.
- `build_files/cmake/platform/platform_apple.cmake`: the template for `platform_ios.cmake`.
