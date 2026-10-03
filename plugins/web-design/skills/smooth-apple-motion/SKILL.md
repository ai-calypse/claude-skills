---
name: smooth-apple-motion
description: >-
  Field notes on building Tauri v2 desktop and iOS apps — IPC commands and
  handler registration, capabilities and permissions, Rust backend state and
  Mutex safety, window creation and lifecycle, sidecar/Node bundling, dev-server
  wiring, WKWebView and WebKit quirks, macOS bundling and deployment traps.
  Use when writing, debugging, or reviewing Tauri code: anything under
  src-tauri/, tauri.conf.json, capabilities JSON, #[tauri::command], invoke(),
  @tauri-apps/api or @tauri-apps/plugin-* imports, cargo tauri dev/build/ios.
  Also use when a WebView behaves differently from a browser (drag-drop does
  nothing, images go blank, CORS, a frozen window, a beach ball), when an app
  works in dev but fails launched from Finder, or when planning iOS/App Store
  work for a Tauri app.
user-invocable: true
argument-hint: "[topic keyword, e.g. 'ipc commands', 'mutex', 'sidecar', 'ios signing']"
---

# Tauri v2 Wisdom

Hard-won patterns for Tauri v2, drawn from production macOS and iOS apps.
Source: Takazudo's Tauri notes (<https://zudo-tauri-wisdom.takazudomodular.com/>).

## How to use this

1. **Match the task to an article** in the index below — titles and one-line
   summaries are there so you can pick without opening anything.
2. **Read only the article(s) you need** from `reference/`. Do not load the
   whole folder; it is ~64,000 words.
3. **Apply the pattern, then cite the file** (e.g. `reference/rust-backend/mutex-safety.mdx`)
   so the user can read further.

If nothing in the index matches, say so rather than guessing — these notes are
opinionated and specific, and inventing a "Tauri pattern" that is not in them is
worse than admitting the gap.

Articles are `.mdx` with YAML frontmatter. `<Warning>` / `<Danger>` blocks are
the load-bearing parts — they mark traps that cost the author real debugging
time.

## Rules that are always true

These recur across articles and cause silent, hard-to-debug failures. Apply
them without needing to open a file; open the cited article when you need the
reasoning or the full code.

**Rust backend**

- Never `.lock().unwrap()` a `Mutex` in a Tauri command. One panic poisons it
  and every later lock panics, taking down the whole app. → `rust-backend/mutex-safety.mdx`
- Never block `setup()` waiting on a build or a sidecar. No window appears and
  the app looks hung. Spawn a background thread and return. → `architecture/loading-screen.mdx`
- Never hold a mutex guard across a slow call (curl, subprocess). Scope the
  guard to the read. → `architecture/process-lifecycle.mdx`
- A command missing from `generate_handler!` fails **silently** at runtime —
  there is no compile-time check. → `frontend/ipc-commands.mdx`
- Command errors must be `String` (or `Into<InvokeError>`); custom error structs
  do not work. `.map_err(|e| format!(...))`. → `frontend/ipc-commands.mdx`
- Call `kill_port()` before spawning a sidecar, and `app_handle.exit(0)` on
  window close, or processes and ports leak. → `architecture/process-lifecycle.mdx`

**Frontend / WebView**

- Never use `useLayoutEffect` for `invoke()`. It blocks paint and produces a
  1–2s macOS beach ball. Use `useEffect`. → `frontend/use-effect-pitfall.mdx`
- Browser `onDrop` never fires for OS file drops. Use `onDragDropEvent` from
  `getCurrentWebviewWindow()`. → `frontend/drag-drop.mdx`
- A missing capability fails only at runtime, when the button is clicked —
  never at compile time. → `frontend/capabilities.mdx`
- `withGlobalTauri: true` is required if any page touches `window.__TAURI__`,
  and its absence is silent. → `frontend/ipc-commands.mdx`
- `"csp": null` disables CSP entirely. Fine for a local tool, not for anything
  distributable. → `frontend/asset-protocol.mdx`
- Playwright's default Chromium gives false confidence — Tauri uses WebKit on
  macOS. Put `webkit` in the project matrix. → `frontend/playwright-engine-pitfall.mdx`

**Dev vs production**

- macOS Finder launches apps with a minimal PATH (`/usr/bin:/bin:/usr/sbin:/sbin`).
  Homebrew/nvm tools are absent. Resolve absolute paths. This is the #1 cause of
  "works in `cargo tauri dev`, blank screen from Finder". → `getting-started/dev-vs-production.mdx`
- `beforeDevCommand` runs from the **repo root**, not the `tauri.conf.json`
  directory, and only during `cargo tauri dev`. → `dev-server/vite-integration.mdx`
- Without `beforeBuildCommand`, `cargo tauri build` silently ships stale
  frontend assets. A suspiciously fast build is the tell. → `deployment/build-bundle.mdx`
- Tauri strips the target triple from sidecar binary names in the production
  bundle (`node-aarch64-apple-darwin` → `node`). Handle both. → `architecture/sidecar-pattern.mdx`
- Never `cp -rf` over an existing `.app` — the inner binary stays stale.
  `rm -rf` first. → `deployment/macos-pitfalls.mdx`

**iOS**

- Keep the bundle identifier alphanumeric + dots; hyphens and underscores have
  caused breakage. → `mobile/ios-project-structure.mdx`
- Do not set `NSAllowsArbitraryLoads` — it triggers App Store Guideline 4.2
  problems. Scope `NSExceptionDomains` instead. → `mobile/ios-dev-loop.mdx`
- A pure WebView wrapper gets rejected under Guideline 4.2. → `mobile/app-store-review-4-2.mdx`

## Article index

All paths are relative to `reference/`.

### getting-started/
- `getting-started/dev-vs-production.mdx` — **Dev vs Production**. The fundamental differences between dev mode and production mode, and why apps break when launched from Finder.
- `getting-started/project-setup.mdx` — **Project Setup**. Cargo.toml, tauri.conf.json, capabilities, and directory structure.

### architecture/
- `architecture/backend-bridge.mdx` — **Backend Bridge / Adapter Pattern**. A swappable backend abstraction enabling three dev modes from one frontend codebase.
- `architecture/loading-screen.mdx` — **Loading Screen**. Showing a loading page immediately while background processes start up.
- `architecture/process-lifecycle.mdx` — **Process Lifecycle**. Port cleanup, signal handling, process groups, clean shutdown.
- `architecture/shared-packages.mdx` — **Shared Packages Overview**. How a 16-package monorepo around Tauri apps is organised.
- `architecture/sidecar-pattern.mdx` — **Sidecar Pattern**. Bundling and managing Node.js or other binaries as sidecar processes.

### rust-backend/
- `rust-backend/core-crate-pattern.mdx` — **Core Crate Testing Pattern**. Extracting logic into a Tauri-free crate for fast cross-platform tests.
- `rust-backend/external-tool-actions.mdx` — **External-Tool Preview/Accept/Cancel**. Bundled Node CLIs as one-shot subprocesses, temp-dir staging, and a deletion guard.
- `rust-backend/file-watchers.mdx` — **Debounced File Watchers**. Write markers that distinguish app writes from external changes.
- `rust-backend/macos-window-tiling.mdx` — **macOS Window Tiling Shortcuts**. Fixing Tile Left/Right, broken by a muda menu library bug.
- `rust-backend/menu-events.mdx` — **Menu Event Handlers**. Handling menu events without blocking the main thread.
- `rust-backend/mutex-safety.mdx` — **Mutex Safety in Tauri Commands**. Poisoned mutexes, and the AppState pattern that avoids them.
- `rust-backend/node-detection.mdx` — **Node Detection with Version Managers**. Finding Node at absolute paths (nodenv, nvm, volta, fnm) for Finder-launched apps.
- `rust-backend/settings-cache.mdx` — **Settings Cache with External Edit Detection**. mtime-based cache invalidation in AppState.
- `rust-backend/settings-persistence.mdx` — **Persisting Settings to the OS Config Dir**. Corrupt-file fallback and path validation.
- `rust-backend/settings-validation.mdx` — **Settings Validation Pattern**. Shared TS validation enforcing types, ranges, enums, and legacy field migration.
- `rust-backend/window-management.mdx` — **Window Creation and Lifecycle**. Splash screens, dev-server polling, macOS tweaks, destroy cleanup.

### frontend/
- `frontend/asset-protocol.mdx` — **Asset Protocol and CSP Scope**. convertFileSrc, the `csp: null` shortcut, the assetProtocol allowlist, and why images go blank in a hardened build.
- `frontend/capabilities.mdx` — **Permissions and Capabilities**. Controlling what the frontend can access, including plugins and remote URLs.
- `frontend/color-theme-system.mdx` — **Color Theme System**. Two-tier colour architecture from raw palette to semantic CSS custom properties.
- `frontend/deep-links.mdx` — **Deep Links and the useEffect Teardown Race**. Custom URL schemes, and not leaking the async onOpenUrl subscription.
- `frontend/drag-drop.mdx` — **Native File Drag-and-Drop**. Why browser onDrop never fires, and the onDragDropEvent replacement.
- `frontend/e2e-test-split.mdx` — **E2E Test Split Strategy**. Splitting Playwright tests into CI-safe and @interactive categories.
- `frontend/find-in-page.mdx` — **Find-in-Page**. DOM-based Ctrl+F text search in a Tauri webview.
- `frontend/http-plugin-cors.mdx` — **Bypassing CORS with plugin-http**. Calling third-party REST APIs from the WebView without CORS.
- `frontend/ipc-commands.mdx` — **Tauri IPC Command Patterns**. Registration, signatures, state access, error handling, async.
- `frontend/playwright-engine-pitfall.mdx` — **Playwright: WebKit Engine Mismatch**. Why Chromium tests give false confidence.
- `frontend/use-effect-pitfall.mdx` — **useLayoutEffect Causes Beach Ball**. Why IPC in useLayoutEffect freezes the app.
- `frontend/webkit-form-autocomplete.mdx` — **WebKit Form Autocomplete Pill**. The floating "recent value" pill is WKWebView autofill; when to disable it.

### dev-server/
- `dev-server/development-modes.mdx` — **Development Modes**. Full native, frontend-only mock, and hybrid REST.
- `dev-server/sse-live-reload.mdx` — **SSE-Based Live Reload**. Server-Sent Events reload for dev servers that do full rebuilds.
- `dev-server/vite-integration.mdx` — **Vite Integration**. Vite as the frontend dev server and build tool, and the beforeDevCommand cwd trap.
- `dev-server/watcher-loops.mdx` — **Preventing Watcher Loops**. Avoiding infinite rebuilds when output lands in a watched directory.

### deployment/
- `deployment/build-bundle.mdx` — **Building App Bundles**. cargo tauri build, targets, icons, installation, beforeBuildCommand.
- `deployment/cargo-cache.mdx` — **Cargo Cache Invalidation**. Forcing Cargo to re-embed frontend assets.
- `deployment/macos-pitfalls.mdx` — **macOS Deployment Pitfalls**. Subtle, hard-to-debug macOS-specific failures.
- `deployment/node-download.mdx` — **Bundling Node.js**. Shipping a standalone Node binary, with SHA256 verification.

### mobile/
- `mobile/app-store-review-4-2.mdx` — **App Store Review Guideline 4.2**. Why WebView wrappers get rejected and what native functionality passes.
- `mobile/ios-dev-loop.mdx` — **iOS Dev Loop**. cargo tauri ios dev, TAURI_DEV_HOST, ATS, Safari Web Inspector.
- `mobile/ios-input-auto-zoom.mdx` — **iOS Input Auto-Zoom**. Why WKWebView zooms into sub-16px inputs, and the fix that keeps pinch-zoom.
- `mobile/ios-prerequisites.mdx` — **iOS Prerequisites**. Xcode, Rust targets, CocoaPods, Apple ID.
- `mobile/ios-project-structure.mdx` — **iOS Project Structure**. gen/apple, Info.ios.plist, bundle.iOS config.
- `mobile/ios-signing-free-team.mdx` — **Signing With a Free Personal Team**. The 7-day profile, blocked capabilities, upgrade path.
- `mobile/native-plugins.mdx` — **iOS Native Plugins**. Haptics and local notifications via guarded dynamic import.
- `mobile/wkwebview-gotchas.mdx` — **WKWebView Gotchas**. Safe-area insets, keyboard, position fixed, service workers, cookies.

### recipes/
- `recipes/app-generation.mdx` — **App Generation System**. Config-driven generator producing app variants from one codebase.
- `recipes/display-scale-system.mdx` — **Display Scale System**. Scaling the whole UI via a CSS custom property instead of browser zoom.
- `recipes/doc-viewer-app.mdx` — **Doc Viewer App**. A lightweight Tauri wrapper around a pnpm dev server.
- `recipes/image-viewer-app.mdx` — **Image Viewer App**. HEIC decode, asset-protocol delivery, two-tier thumbnail cache.
- `recipes/multi-config.mdx` — **Multi-Config App Variants**. Multiple app variants from shared code via overlaid config files.
- `recipes/text-editor-app.mdx` — **Text Editor App**. Vite frontend with rich Rust IPC.

Each section also has an `index.mdx` overview.

## Provenance

These notes were written against a specific stack (macOS wrapper apps, Node
sidecars, Vite). Where a pattern is tied to that context rather than to Tauri
itself — the shared-packages and app-generation articles especially — say so
rather than presenting it as a general Tauri requirement.
