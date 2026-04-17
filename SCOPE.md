# SCOPE — buildthisniw desktop-skeleton

The goal of this skeleton: **from `/skeleton desktop` to a signed, notarized, auto-updating desktop app that authenticates users and talks to Supabase — in a weekend, not a quarter.**

Today the repo is a bare Tauri v2 + Vite + React 19 + TypeScript scaffold with Supabase and TanStack Query wired. This doc is both the destination and the order to get there.

---

## Why Tauri over Electron

- **Bundle size**: 5–10MB installer vs 100MB+ for Electron
- **Memory**: uses the system webview instead of bundling Chromium
- **Security**: Rust backend with capability-based permissions, not a full Node runtime exposed to the renderer
- **Code signing + updater + tray + dialogs**: first-class Tauri plugins, not a zoo of npm packages

Tradeoff: webview rendering differs across macOS (WKWebView), Windows (WebView2), and Linux (WebKitGTK). Budget time for cross-platform QA; don't rely on the latest CSS features without checking all three.

---

## MVP — the critical path

The capabilities a buyer must have before their first feature ships. Everything else is roadmap.

| # | Capability | Status | Notes |
|---|-----------|--------|-------|
| 1 | Tauri v2 scaffold (Rust + Vite + React 19 + TS) | ✅ shipped | identifier `com.buildthisniw.desktop` |
| 2 | Supabase client (anon + session persistence) | ✅ shipped | `src/lib/supabase.ts` |
| 3 | TanStack Query provider | ✅ shipped | `src/lib/query-client.ts` |
| 4 | Routing (multi-window-ready) | ⚠️ planned | Task D1 |
| 5 | UI library + design tokens | ⚠️ planned | Task D2 |
| 6 | Zustand for local UI state | ⚠️ planned | Task D3 |
| 7 | Auth flow (sign-in screen + session guard) | ⚠️ planned | Task D4 |
| 8 | Tauri updater plugin wired | ⚠️ planned | Task D5 |
| 9 | Code signing + notarization docs | ⚠️ planned | Task D6 |
| 10 | IPC boundary (Rust command pattern) | ⚠️ planned | Task D7 |
| 11 | Native menu + system tray | ⚠️ planned | Task D8 |
| 12 | File system access via Tauri APIs | ⚠️ planned | Task D9 |

Beyond these twelve, anything else is a feature, not skeleton infrastructure.

---

## Roadmap — task cards

Each card is the smallest unit of useful work. Acceptance criteria are what "done" looks like so a fresh session can pick it up cold.

### D1 · Routing

**Why:** Every non-trivial desktop app has at least a sign-in view, a main view, and a settings view. Hash routing works with Tauri's file-URL origin; no server-side routing needed.

**Acceptance:**
- `react-router-dom` installed, hash router configured
- Routes: `/sign-in`, `/`, `/settings`, `/about`
- Auth guard redirects unauthenticated sessions to `/sign-in`

**Files:**
- `src/App.tsx` — replace default with `<RouterProvider />` (~30 lines)
- `src/routes/` — NEW folder, one file per route (~4 × 40 lines)
- `src/lib/auth-guard.tsx` — NEW (~20 lines)

Estimate: ~200 lines.

### D2 · UI library + design tokens

**Why:** Desktop users expect native-feeling chrome. shadcn/ui + Tailwind v4 gives the webapp consistency (buyer can copy components across) while respecting macOS / Windows type scales.

**Acceptance:**
- Tailwind v4 configured with `@theme` block
- shadcn/ui installed with 6 core primitives: `button`, `input`, `card`, `dialog`, `dropdown-menu`, `toast`
- OKLCH tokens match webapp-skeleton (dark mode auto-switches with OS)
- `index.html` sets native font stack per platform

**Files:**
- `tailwind.config.ts` — NEW (~40 lines)
- `src/styles/globals.css` — NEW, tokens + `@layer` setup (~60 lines)
- `components/ui/*` — shadcn primitives (generated)
- `components.json` — shadcn config (~15 lines)

Estimate: ~120 lines authored + generated components.

### D3 · Zustand stores

**Why:** TanStack Query owns server state; something needs to own local UI state (sidebar collapsed? which project is selected? keyboard shortcuts enabled?). Zustand is lighter than Redux and fits desktop's longer-lived sessions.

**Acceptance:**
- `src/store/ui-store.ts` with sidebar + theme preference
- Persisted to disk via Tauri's `store` plugin (not localStorage — survives window reload properly)

**Files:**
- `src/store/ui-store.ts` — NEW (~40 lines)
- `src/lib/tauri-storage.ts` — NEW, Zustand middleware backed by `@tauri-apps/plugin-store` (~40 lines)
- `src-tauri/Cargo.toml` — add `tauri-plugin-store` (~1 line)

Estimate: ~80 lines.

### D4 · Auth flow

**Why:** Sign in has to work before anything else is worth building.

**Acceptance:**
- Sign-in screen accepts email + password
- Email OTP path available ("Send a code" button)
- Session persisted across app restarts (Supabase's default localStorage works in webview)
- Sign-out clears session and redirects to `/sign-in`
- Auth state drives the router guard from D1

**Files:**
- `src/routes/sign-in.tsx` — NEW (~80 lines)
- `src/hooks/use-auth.ts` — NEW, wraps `supabase.auth` (~40 lines)
- `src/components/sign-out-button.tsx` — NEW (~20 lines)

Estimate: ~140 lines. Depends on D1 + D2.

### D5 · Tauri updater

**Why:** Users won't manually download updates. The Tauri updater plugin signs and verifies releases cryptographically.

**Acceptance:**
- `tauri-plugin-updater` installed and configured
- Update manifest hosted (GitHub Releases works, CloudFront in production)
- App checks for updates on launch + every 4 hours
- Update signing key generated; public key in `tauri.conf.json`, private key in `.env.local`
- `docs/updates.md` documents the release process

**Files:**
- `src-tauri/tauri.conf.json` — `plugins.updater` block (~15 lines)
- `src-tauri/Cargo.toml` — add plugin (~1 line)
- `src/lib/updater.ts` — NEW (~50 lines)
- `docs/updates.md` — NEW (~80 lines)
- `.github/workflows/release.yml` — NEW (~60 lines)

Estimate: ~200 lines. The GitHub Actions release workflow is the bulk.

### D6 · Code signing + notarization docs

**Why:** Unsigned apps scare users and get blocked by Gatekeeper on macOS and SmartScreen on Windows. The steps aren't hard, but they're undocumented friction.

**Acceptance:**
- `docs/signing-macos.md` covers Developer ID cert, `codesign`, `notarytool submit`, stapling
- `docs/signing-windows.md` covers EV / OV certs, `signtool`, SmartScreen reputation timeline
- `docs/signing-linux.md` covers AppImage signing + optional Snap/Flatpak notes
- Each includes the exact Tauri bundle config to enable it

Estimate: ~300 lines of prose across three docs. No code.

### D7 · IPC boundary pattern

**Why:** Rust is for native work the webview can't do — raw file access, shell commands, keychain, system APIs. A clean `#[tauri::command]` convention prevents the IPC boundary from becoming spaghetti.

**Acceptance:**
- `src-tauri/src/commands/` folder with one file per domain
- Each command typed end-to-end: Rust signature → TS wrapper in `src/lib/tauri-commands.ts` → typed TS caller
- Error handling: Rust errors serialize to a discriminated union the TS layer can match on
- One worked example: `read_project_file(path) -> Result<FileContents, AppError>`

**Files:**
- `src-tauri/src/commands/mod.rs` — NEW (~10 lines)
- `src-tauri/src/commands/fs.rs` — NEW example (~60 lines)
- `src-tauri/src/error.rs` — NEW error enum with `thiserror` (~40 lines)
- `src/lib/tauri-commands.ts` — NEW typed invoke wrapper (~50 lines)

Estimate: ~160 lines.

### D8 · Native menu + system tray

**Why:** Desktop apps need an app menu (Cmd+, for settings, Cmd+Q to quit) and a tray icon for background-running apps. Tauri handles both natively.

**Acceptance:**
- macOS: native app menu with standard items + one app-specific action
- System tray icon with context menu (Show / Hide / Quit)
- Tray icon updates (e.g., unread count badge on macOS)

**Files:**
- `src-tauri/src/menu.rs` — NEW (~80 lines)
- `src-tauri/src/tray.rs` — NEW (~60 lines)
- `src-tauri/src/lib.rs` — wire up menu + tray on startup (~10 lines)

Estimate: ~150 lines.

### D9 · File system access

**Why:** This is often why a user chooses desktop over web. Need to scope and surface file access through the IPC pattern from D7, with capability declarations so the plugin can't overreach.

**Acceptance:**
- `src-tauri/capabilities/default.json` declares file-system permissions with a narrow allowlist
- One example command: "Open project folder" → native dialog → Rust reads file list → TS displays
- Drag-and-drop of files into the window works

**Files:**
- `src-tauri/capabilities/default.json` — extend allowlist (~15 lines)
- `src-tauri/src/commands/fs.rs` — extend from D7 (~40 lines)
- `src/components/drop-zone.tsx` — NEW (~40 lines)

Estimate: ~100 lines. Depends on D7.

---

## Non-goals for this skeleton

- **No Electron fallback** — if Tauri is wrong for the buyer's use case, they should pick a different skeleton, not fork this one.
- **No SSR** — Vite builds static bundles served from Tauri's file-URL origin. No Next.js, no SvelteKit.
- **No state-sync between desktop and mobile** — if the buyer needs that, it lives in Supabase, not in skeleton plumbing.
- **No bundled AI models** — desktop is a good place for on-device inference, but the skeleton stays generic. Add llama.cpp or ONNX Runtime per-project.

---

## How codekit commands compose on top

| Command | What it does here |
|---------|-------------------|
| `/desktop <feature>` | Scaffolds a route + IPC command + typed wrapper using the D1 / D7 conventions |
| `/test desktop` | Runs `cargo test` for Rust + `vitest` for TS, plus `gui-desktop-mac`/`-windows`/`-linux` smoke tests |
| `/deploy desktop` | Builds signed + notarized artifacts and cuts a GitHub Release (uses D5 + D6) |
| `/audit desktop` | Checks capabilities JSON for overly broad permissions |

The skills that know this skeleton: `gui-desktop`, `gui-desktop-mac`, `gui-desktop-windows`, `gui-desktop-linux`, plus the shared `supabase-*` and `stripe-*` families.

---

## Suggested order to fill the roadmap

For a buyer starting from zero:

1. **D2 (UI library)** — makes everything else visible
2. **D1 (routing)** — unblocks multi-screen work
3. **D4 (auth)** — unblocks any feature that touches user data
4. **D3 (state)** — small but pays off across every feature
5. **D7 (IPC pattern)** — before writing any Rust commands, establish the convention
6. **D9 (file system)** — first native capability that's usually the reason the app is on desktop
7. **D5 + D6 (updater + signing)** — required before first public release
8. **D8 (menu + tray)** — polish, but users notice its absence
9. **D4 extras** — social sign-in, magic links, passkey support, etc.

Estimated total to reach shippable v0.1: ~1,400 lines of code + ~500 lines of docs, spread across a weekend of focused work.

---

## Version

This scope doc tracks the repo `main` branch. For the version wired into codekit, see `SKELETONS.desktop.commit` in `codekit/lib/constants.js`.
