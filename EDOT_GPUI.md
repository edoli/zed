# Edot GPUI branch

This is a GitHub fork of `zed-industries/zed` with the original Git history.
The `edot-gpui` branch starts at upstream commit
`279fe070bb389b79652e52065b2f001edcc0b11b`, the same source revision previously
published as `gpui-pre 0.3.8`. Edot consumes `gpui` and `gpui_platform` directly
from one pinned commit; their workspace dependencies come from this same tree.

Latest integrated upstream main: `3c2118dd69c02bab648e1d5f9f51800b46af7771`
(2026-11-19 integration; upstream commit dated 2026-11-18).
The `gpui-edol` branch merges this revision while retaining the maintained
changes below. Edot pins both GPUI dependencies to the same published fork
commit. Local paths are used only while validating unpublished integrations.

## Maintained changes

Only these GPUI source/manifests are changed:

- `crates/gpui/Cargo.toml`
- `crates/gpui/src/svg_renderer.rs`
- `crates/gpui_windows/Cargo.toml`
- `crates/gpui/src/platform.rs`
- `crates/gpui/src/text_system.rs`
- `crates/gpui_platform/Cargo.toml`
- `crates/gpui_windows/src/direct_write.rs`
- `crates/gpui_windows/src/events.rs`
- `crates/gpui_windows/src/platform.rs`

`extended-image-formats` controls the additional image codecs and image/rayon.
`svg-text` controls SVG text, font loading and shaping, and emoji fallback.
`svg-raster-images` controls raster images embedded in SVGs. Upstream default
features still enable all three. Edot disables defaults and explicitly enables
`windows-manifest`; BMP/GIF/JPEG/PNG/WebP and SVG vectors remain enabled.

The two image dependencies and GPUI's resvg dependency use explicit feature
lists because inheriting the workspace dependency would unconditionally enable
the functionality that Edot excludes. Keep their version requirements aligned
with the corresponding workspace dependencies when merging upstream updates.

`gpui_platform/deferred-fonts` forwards an opt-in Windows policy. It starts with
the cached DirectWrite system font collection and disables automatic change
checks when font selection fails. The consumer calls
`TextSystem::refresh_system_fonts_once` on a background thread after its first
frame. That call attempts an immediate update at most once, including failures,
without holding the rendering state lock during the Windows call. Existing font
IDs and layout caches are preserved. After that update the process keeps its
collection until restart. Without this feature, upstream font initialization and
failed-selection update behavior remain unchanged.

`TextSystem::font_id` exposes the existing fallible lookup without applying
fallbacks. Edot uses it to persist the actual fallback family when the configured
font cannot be resolved, keeping the setting and rendered body font aligned.
It shares upstream's `CachedFont` cache: direct matches succeed, while missing
fonts and entries already resolved through a fallback return an error. A lookup
must not replace a cached fallback or report that fallback as a direct match.

The root README review notice is required by upstream `AGENTS.md`. This branch
also contains this maintenance document. No crates.io snapshot importer or
separate patch file is used.

## Update from upstream

The local checkout uses these remotes:

```sh
git remote add upstream https://github.com/zed-industries/zed.git
```

Add the remote only if it does not already exist. With local changes preserved:

```sh
git switch edot-gpui
git pull --ff-only origin edot-gpui
git fetch upstream
git merge upstream/main
```

Resolve any conflicts while preserving the optional features and upstream
default behavior. Check the workspace image/resvg versions and any GPUI API
changes. Validate the consumer application using local path dependencies if
needed, then publish the branch:

```sh
git push origin edot-gpui
git rev-parse HEAD
```

In Edot, set both normal and test `gpui` dependencies and `gpui_platform` to
the same full Git commit. Update the lockfile and regenerate local path-remap
configuration. Run the application's feature/source guard, formatting, Clippy,
all headless tests, upstream-default feature check, release build and binary
path audit. Check executable size and document the new upstream and fork pins.

Use a commit pin in Edot instead of a floating branch. Do not force-push a
published commit used by an application release. Merging the upstream branch
updates this fork; the application's pin moves only after its checks pass.

## 2026-10-22 merge conflicts

- `crates/gpui/Cargo.toml`: upstream made workspace dependency features explicit
  and moved `util_macros` to dev-dependencies. Keep those changes, including
  regex defaults and schemars derive support, while retaining optional regex
  and ttf-parser dependencies behind `svg-text`.
- `crates/gpui/src/text_system.rs`: upstream replaced cached `Result<FontId>`
  values with `CachedFont::{Direct, Fallback, Missing}`. Adapt the public
  fallible lookup to that cache without accepting fallback IDs as direct matches.
- `crates/gpui_platform/Cargo.toml`: upstream added the default `multithreaded`
  web feature next to the fork's `deferred-fonts`. Retain both; consumers that
  disable defaults still opt into Windows deferred fonts explicitly.

Validation on x86_64 Windows MSVC with Rust 1.98.1:

- Workspace formatting and `git diff --check` passed.
- GPUI and the Windows platform compiled with defaults disabled and
  `gpui/windows-manifest,gpui_platform/deferred-fonts` enabled.
- The platform compiled with `gpui/default` enabled and deferred fonts disabled.
- Both `font_resolution_tests` passed, covering direct lookup caching and
  rejecting fallback entries without disturbing cached fallback resolution.
- `./script/clippy -p gpui -p gpui_platform --target-dir target --locked` passed
  with its release, all-targets and all-features settings.
- The minimal platform's resolved image features were exactly BMP/GIF/JPEG/PNG/
  WebP, and resvg's optional text/font/raster features remained disabled.
- `cargo build --release --target-dir target -p gpui -p gpui_platform
  --no-default-features --features gpui/windows-manifest,gpui_platform/deferred-fonts
  --locked` passed. This builds the fork libraries, not the pinned Edot app.

## 2026-11-19 integration

- Merged upstream main into `gpui-edol`. The seven maintained
  GPUI files remain the only source/manifests that differ from upstream.
- `crates/gpui/Cargo.toml`: retain upstream's `icu_segmenter` dependency together
  with the fork's optional image, SVG text, and raster feature controls.
- `crates/gpui/src/svg_renderer.rs`: keep upstream's parsed SVG cache and font
  options, and gate font initialization, font helpers, and text tests behind
  `svg-text`. The vector-only configuration still uses default SVG options.
- Upstream `Window::cover_bounds` now returns zero size for empty bounds before
  outward rounding. No fork-specific clipping fix is needed. Edot removes its
  `paint_if_visible` workaround and retains the scrolling regression test.
- Consumer compatibility changes: use `Keystroke::parse_with_os` for dynamic
  strings and pass the native OS to `unparse`; remove explicit `IntoElement`
  implementations covered by the new blanket implementation; reference the
  already-resolved `anyhow` crate directly for HTTP errors.
- Use temporary local paths to validate unpublished integrations. Restore
  published, matching Git pins before running Edot's source/feature checker.

Validation on Windows x86_64 MSVC:

- Workspace formatting and the fork diff against upstream passed.
- `./script/clippy -p gpui -p gpui_platform --locked --target-dir target`
  passed with all features and targets enabled.
- Both fallible font lookup tests, four empty/fractional clip and shadow tests,
  and all ten SVG renderer tests passed. Test execution used
  `cargo test --release -p gpui --lib --all-features --locked --target-dir target`
  with command-line overrides `--config profile.release.package.gpui.opt-level=0`
  and `--config profile.release.package.gpui.codegen-units=256`. The initial
  fully optimized test build was stopped during lengthy optimization; these
  overrides affect only the test command and are not saved in the manifests.
- Edot passed 145 tests, all-targets Clippy with warnings denied, formatting,
  the local fork source/feature guard, the release build, and the binary path
  audit. Its normal release optimization settings were retained.
- The actual `target/release/edot.exe` passed a native Windows check with
  separate temporary data: initial frame, 24 scroll samples, and a resized
  frame. No leaked horizontal line appeared in the 26 captured frames.
- The Edot executable is 11,860,480 bytes, compared with 11,676,160 bytes before
  this integration. These validation runs preceded publishing the merge.

## Windows IME cursor recovery (2026-11-19)

- Forward composition end to the text input handler, and release any abandoned
  preedit when a new native composition begins.
- Before mouse-down reaches a view, complete the focused window's live IMM
  composition and clear leftover marked text. Completion may synchronously
  invoke the input handler, so it runs before taking the mouse input callback.
- Before dispatching navigation keys, clear abandoned marked text only when
  IMM has no live composition. Preserve live IME navigation and avoid completing
  a different window's shared input context.
- If native completion fails, consume the mouse-down rather than move the
  selection while the old composition still owns subsequent text.
- The Edot ignored native probe and `tools/test_ime_cursor.py` exercise actual
  Win32 start/end, mouse and navigation dispatch with a deliberately abandoned
  preedit. The published parent fails those recovery cases. The fixed backend
  also preserves undo and normal Microsoft Korean IME composition without
  duplicated or lost characters. The original intermittent trigger remains
  distinct from this controlled reproduction of its broken state.
- Correct the deferred-font test's stale `new(None)` call to the current `new()`
  signature so all-targets Windows Clippy can check that test as well.
