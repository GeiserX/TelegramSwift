# Building this fork on Xcode 26 (macOS 26 SDK)

This fork builds on the latest Xcode (26.6 / macOS 26.5 SDK). Upstream was written for
Xcode ~16, so a few things need handling. The one **source** fix is already committed
(see the `DT_TOOLCHAIN_DIR` note below); the rest are build-environment steps.

## The one source fix (committed)

`Telegram.xcodeproj/project.pbxproj` force-links the Swift AppKit overlay with
`OTHER_LDFLAGS = "$(TOOLCHAIN_DIR)/usr/lib/swift-5.0/macosx/libswiftAppKit.dylib"`.
On Xcode 26 the Metal compiler is a **separate downloadable toolchain**, and
`$(TOOLCHAIN_DIR)` flips to that Metal toolchain for any target that compiles `.metal`
(e.g. `TelegramShare`) — which has no `swift-5.0` dir, so the link dies with
`no such file … Metal.xctoolchain/usr/lib/swift-5.0/macosx/libswiftAppKit.dylib`.
Fix: use `$(DT_TOOLCHAIN_DIR)` (always the default Xcode toolchain, which has the file).
Identical on Xcode 16, so no regression. (4 occurrences.)

## Build-environment steps (not in source)

1. **Submodules use SSH URLs on github.com AND gitlab.com** (Sparkle). Rewrite to HTTPS or
   the recursive init aborts and leaves partial/corrupt checkouts:
   ```
   git config --global url.https://github.com/.insteadOf git@github.com:
   git config --global url.https://gitlab.com/.insteadOf git@gitlab.com:
   git submodule update --init --recursive -j8
   ```
   If a checkout is corrupt (rlottie/libprisma missing files), `git submodule update --init --recursive --force`.

2. **Build tools** (use brew's full path in non-login shells; PATH must include them for
   the framework build's script phases):
   ```
   brew install cmake ninja yasm nasm meson pkg-config
   # + gas-preprocessor.pl on PATH for ffmpeg
   export PATH="/opt/homebrew/bin:$PATH"
   export CMAKE_POLICY_VERSION_MINIMUM=3.5   # new CMake removed pre-3.5 policy compat (mozjpeg)
   ```

3. **Native frameworks:** `sh scripts/configure_frameworks.sh`. If it leaves any framework
   without installed headers (mozjpeg `jpeglib.h`, OpenH264 `wels/*`) or with stale libs
   (ffmpeg), rebuild that one solo: `xcodebuild ... -project core-xprojects/<lib>/<Fw>.xcodeproj -scheme <Fw> build` after `rm -rf core-xprojects/<lib>/build`.

4. **webrtc (tg_owt) + clang-26:** abseil's `ABSL_ATTRIBUTE_LIFETIME_BOUND [[clang::lifetimebound]]`
   is a hard error on void-returning functions. Make the macro empty in
   `submodules/tg_owt/src/third_party/abseil-cpp/absl/base/attributes.h` (it's a static-analysis
   hint only). tg_owt is a submodule, so this is applied at build time, not committed here.

4b. **FlatBuffers unaligned-read crash (REQUIRED for a usable app).** Runtime crash, not a build
   error: opening any chat row with Instant View / rich web-page content trapped with
   `EXC_BREAKPOINT` in `ByteBuffer.read → UnsafeRawPointer.load → _assertionFailure`. FlatBuffers
   reads packed data at unaligned offsets; Swift's `.load(as:)` *requires* alignment and traps
   (at ANY optimization level — this is not a debug-only check). In
   `submodules/telegram-ios/submodules/TelegramCore/FlatBuffers/Sources/ByteBuffer.swift`, `read`
   picks `.load` unless `allowReadingUnalignedBuffers` is true (it defaults false). Fix: make the
   fallback use `loadUnaligned` too — change the `return …advanced(by: position).load(as: T.self)`
   line to `.loadUnaligned(as: T.self)` (same call already used on the true branch; safe on all
   archs). Body-only change → incremental rebuild (recompile FlatBuffers + relink, ~2 min).
   telegram-ios is a submodule (overtake/Telegram-iOS) → build-time patch, not committed here.

5. **Firebase macOS xcframework slices are shallow** (iOS-style), which Xcode 26 rejects at
   embed validation. Convert the macos-arm64_x86_64 slices of `FirebaseAnalytics`,
   `GoogleAppMeasurement`, `GoogleAppMeasurementIdentitySupport` to non-shallow
   (`Versions/A` + `Current` + top-level symlinks) and re-sign ad-hoc, then re-embed.

6. **App build** — parallel, no coverage/LTO, ad-hoc, non-sandboxed friendly. Coverage
   (`-profile-generate`) + LTO + WMO make the main-app compile pathologically slow; single-file
   parallelises across all cores.

   **Build `-O`, NOT `-Onone`.** `-Onone` compiles fast but ships Swift's debug preconditions.
   This codebase reads FlatBuffers at unaligned byte offsets (fine on arm64 hardware), and the
   `UnsafeRawPointer.load` alignment `_debugPrecondition` then *traps* (`EXC_BREAKPOINT`) —
   e.g. opening any chat row with Instant View / rich web-page content crashes on
   `RichText.init(flatBuffersObject:)`. `-O` compiles that precondition out (as the official
   app ships), and is snappier for daily use. `SWIFT_COMPILATION_MODE=singlefile` keeps `-O`
   parallel (per-file optimization, no WMO bottleneck). A tell: `-Onone` emits a separate
   `Telegram.debug.dylib`; `-O` yields a single `Telegram` binary.
   ```
   xcodebuild -workspace Telegram-Mac.xcworkspace -scheme Telegram -configuration Release \
     -derivedDataPath <dd> \
     CODE_SIGN_IDENTITY=- CODE_SIGNING_REQUIRED=NO CODE_SIGN_STYLE=Manual DEVELOPMENT_TEAM= \
     MACOSX_DEPLOYMENT_TARGET=11.0 ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES=NO \
     SWIFT_OPTIMIZATION_LEVEL=-O SWIFT_COMPILATION_MODE=singlefile \
     CLANG_ENABLE_CODE_COVERAGE=NO LLVM_LTO=NO ENABLE_TESTABILITY=NO build
   ```

7. **Launch an ad-hoc build:** the built app is sandboxed with team-signed app-group +
   keychain-access-group entitlements → launchd refuses an ad-hoc signature (spawn error 163).
   Re-sign ad-hoc with those team entitlements stripped (keep `get-task-allow`):
   ```
   codesign --force --deep --sign - --entitlements minimal.entitlements /Applications/Telegram.app
   ```
   (Stripping the app-group means no access to the official app's session container → one-time re-login.)
