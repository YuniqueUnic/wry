# Main-frame-only initialization for downloaded pages

Upstream base: [`wry-v0.57.0`, `792d0359`](https://github.com/tauri-apps/wry/tree/792d0359ba6501a4fc360ece17de2ae42329a47c).
Applied upstream change: [PR 1874](https://github.com/tauri-apps/wry/pull/1874),
commit [`c839fec0`](https://github.com/haexhub/wry/commit/c839fec0cf1088f0721fa27ab6112dbe37378924).
Maintenance branch: `YuniqueUnic/wry`, `syzygy/main-frame-only-v0.57.0`.

## Why the correction belongs here

Tauri marks its IPC initialization scripts `for_main_frame_only`. Wry 0.57.0
discarded that flag on Windows and Android. A sandboxed downloaded page therefore
received the framework bootstrap, including the authenticated IPC sender. On
Android its native bridge associates messages with the top-level URL, so neither
the page CSP nor a window capability can separate that sender from the host.
Removing framework objects after initialization does not repair this boundary:
they are already installed and are non-configurable. Replacing Tauri's entire IPC
implementation would duplicate binary, channel and lifecycle behavior.

## Scope

- Preserve explicit all-frame scripts.
- Guard main-frame-only scripts before passing them to Android document-start and
  Windows document-created APIs. Guard Windows' internal `window.ipc` creation too.
- In Android's HTML injection fallback, carry the native request's `isForMainFrame`
  bit and omit main-frame-only scripts from subframe responses altogether. Do not
  put guarded authentication source into a subframe's readable DOM.
- Keep the upstream implementation and existing examples/checks. The only cherry-pick
  conflict was the `unsafe` block surrounding Windows IPC registration; the 0.57.0
  function already provides that scope, so its original structure is retained.

The upstream guard introduces block scope for top-level `let`, `const` and `class`.
Its strict-directive helper recognizes only an unambiguous leading, semicolon-ended
`use strict`. It is not a general JavaScript parser. Syzygy's Tauri bootstrap and
official plugin initializers use self-contained closures; no custom initializer
depending on these top-level lexical bindings was found during this integration.

## Verification and removal

Use `mbx test --manifest-path crates/patches/wry/Cargo.toml --lib main_frame_only`,
then `mbx check --manifest-path crates/patches/wry/Cargo.toml --lib --target
aarch64-linux-android` and the corresponding Windows target. Check the actual
Syzygy Tauri consumer separately. Native Windows and Android runs must verify that
the main page still invokes, declared plugin-page calls still work, and a subframe
cannot invoke a host command directly. Cross-compilation is not that runtime proof.

Integration results and their limits are recorded in
`docs/plugins/implementation/assets/clipboard-core-20261029/verification.md` in
the Syzygy repository. Remove this fork when a reviewed upstream release contains
both document-start guarding and the Android HTML fallback filtering, then rerun
the same consumer and frame journeys. Do not remove it solely because a newer
version number exists.
