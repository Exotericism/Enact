# Enact - Cross-Platform Matrix Client

## Project Overview

Enact is a cross-platform Matrix client built with **Tauri V2 + React/TypeScript** and **matrix-rust-sdk**. It ships two UI skins (Discord-like and Telegram-like) from a single codebase via build-time configuration.

### Target Platforms
- **Desktop**: Windows, Linux, macOS (Tauri V2 + matrix-rust-sdk native)
- **Mobile**: Android, iOS (Tauri V2 mobile + matrix-rust-sdk native)
- **Browser**: Standalone SPA (matrix-rust-sdk compiled to WASM via wasm-bindgen)

## Architecture

```
React/TypeScript UI
  ├── Discord Skin (3-panel: servers | channels | chat)
  └── Telegram Skin (2-panel: chat list | chat)
         ↓
  SDK Abstraction Layer (packages/app/src/sdk/)
    ├── TauriBackend  →  Tauri invoke()  →  enact-tauri-commands  →  enact-core  →  matrix-sdk
    └── WasmBackend   →  WASM bindings   →  enact-wasm            →  enact-core  →  matrix-sdk
```

All Matrix protocol logic lives in `enact-core` (Rust). The frontend never touches matrix-sdk directly.

### Key Architectural Decisions
- **enact-core** uses feature flags: `native` (SQLite + tokio) vs `wasm` (IndexedDB + wasm-bindgen-futures)
- **Skin selection is build-time** via `VITE_SKIN` env var — enables tree-shaking of unused skin
- **Voice/video calls** embed Element Call as a widget (iframe), not native WebRTC
- **State management** uses Zustand (lightweight, TypeScript-first)
- **Platform detection** at runtime: `window.__TAURI__` present → TauriBackend, else → WasmBackend

## Project Structure

```
Enact/
├── Cargo.toml                      # Workspace root
├── package.json                    # Root pnpm workspace
├── pnpm-workspace.yaml
├── crates/
│   ├── enact-core/                 # Shared Matrix SDK wrapper (feature-gated native/wasm)
│   ├── enact-wasm/                 # WASM bindings for browser target
│   └── enact-tauri-commands/       # Tauri command definitions
├── src-tauri/                      # Tauri app shell (desktop + mobile entry)
│   ├── tauri.conf.json             # Base config
│   ├── tauri.discord.conf.json     # Discord skin overlay
│   └── tauri.telegram.conf.json    # Telegram skin overlay
└── packages/
    └── app/                        # React frontend
        └── src/
            ├── sdk/                # Platform abstraction layer
            │   ├── types.ts        # Shared TS types
            │   ├── tauri-backend.ts
            │   ├── wasm-backend.ts
            │   └── platform.ts     # Runtime backend selection
            ├── components/
            │   ├── shared/         # Skin-agnostic components
            │   ├── discord/        # Discord skin components
            │   └── telegram/       # Telegram skin components
            ├── hooks/              # React hooks (useAuth, useRoomList, useTimeline, etc.)
            ├── stores/             # Zustand stores (authStore, roomStore, messageStore, etc.)
            ├── layouts/            # SkinProvider, DiscordShell, TelegramShell
            ├── calls/              # Element Call widget integration
            └── styles/
                └── tokens/         # discord.css, telegram.css (CSS custom properties)
```

## Rust Crates

| Crate | Purpose | Key Dependencies |
|---|---|---|
| `enact-core` | All Matrix logic: auth, sync, rooms, messages, crypto, media | `matrix-sdk`, `matrix-sdk-ui` |
| `enact-tauri-commands` | `#[tauri::command]` wrappers around enact-core | `enact-core[native]`, `tauri` |
| `enact-wasm` | `#[wasm_bindgen]` exports wrapping enact-core | `enact-core[wasm]`, `wasm-bindgen` |

### enact-core Feature Flags
- `native` (default): enables `matrix-sdk/sqlite`, `tokio/rt-multi-thread`
- `wasm`: enables `matrix-sdk/indexeddb`, uses `wasm-bindgen-futures`

## Build Commands

```bash
# Development
VITE_SKIN=discord pnpm tauri dev                    # Desktop, Discord skin
VITE_SKIN=telegram pnpm tauri dev                   # Desktop, Telegram skin

# Desktop builds
VITE_SKIN=discord pnpm tauri build --config src-tauri/tauri.discord.conf.json
VITE_SKIN=telegram pnpm tauri build --config src-tauri/tauri.telegram.conf.json

# Mobile
VITE_SKIN=discord pnpm tauri android build
VITE_SKIN=telegram pnpm tauri ios build

# Browser (Vite-only, no Tauri)
VITE_SKIN=discord pnpm --filter app build

# WASM bindings
./scripts/build-wasm.sh
```

## Development Guidelines

### Rust Code
- All Matrix protocol interaction goes through `enact-core` — never add matrix-sdk calls directly in `enact-tauri-commands` or `enact-wasm`
- Use `#[cfg(not(target_arch = "wasm32"))]` and `#[cfg(target_arch = "wasm32")]` sparingly in enact-core; prefer abstracting platform differences behind traits
- Tauri commands in `enact-tauri-commands` should be thin wrappers: validate input, call enact-core, serialize output
- Use `thiserror` for error types, `tracing` for logging
- All async functions use `tokio` on native, `wasm-bindgen-futures` on WASM

### TypeScript/React Code
- UI code calls the SDK abstraction layer (`packages/app/src/sdk/`), never Tauri `invoke()` or WASM functions directly
- Both `TauriBackend` and `WasmBackend` implement the same `MatrixBackend` interface
- Skin-specific components go in `components/discord/` or `components/telegram/`
- Shared components go in `components/shared/` — these must work with both skins
- Use Zustand stores for state; hooks wrap store access + SDK calls
- CSS custom properties (design tokens) in `styles/tokens/` — Tailwind references these
- Use `import.meta.env.VITE_SKIN` for build-time skin selection

### Skin Architecture
- `SkinProvider` at the React tree root reads `VITE_SKIN` and renders the correct layout shell
- Discord skin: 3-panel (server bar | channel list | chat), dark theme, flat messages
- Telegram skin: 2-panel (chat list | chat view), light theme, bubble messages
- Both skins share the same data hooks and stores — only presentation differs
- Mobile viewports: both skins collapse to single-panel with drawer navigation

### Element Call Integration
- Voice/video calls use Element Call embedded as an iframe widget
- Widget API communication via `postMessage` (host side in `calls/WidgetApi.ts`)
- Call state managed through MatrixRTC membership state events via enact-core
- Element Call bundle shipped as static asset in `packages/app/public/element-call/`

## Key Dependencies

### Rust
- `matrix-sdk` 0.14+ (with `e2e-encryption`, `sqlite`/`indexeddb`, `rustls`)
- `matrix-sdk-ui` 0.14+ (TimelineBuilder, room list)
- `tauri` 2.10+
- `tokio` 1.x (native only)
- `wasm-bindgen` 0.2 (WASM only)

### TypeScript
- React 19+, React DOM 19+
- `@tauri-apps/api` 2.x
- Zustand 5+
- Vite 6+
- Tailwind CSS 4+

## Testing

- Rust: `cargo test` in workspace root runs all crate tests
- TypeScript: `pnpm test` (Vitest)
- E2E: Playwright for desktop/browser
- Lint: `cargo clippy` (Rust), `pnpm lint` (ESLint + TypeScript)
- Format: `cargo fmt`, `pnpm format` (Prettier)

## Implementation Phases

1. **Foundation**: Project scaffolding, auth, sliding sync, room list, message timeline, E2EE (desktop)
2. **Enhanced Messaging**: Reactions, editing, replies, media upload/download, search (+ browser/WASM)
3. **Social Features**: Spaces hierarchy, presence, notifications, mentions
4. **Calls & Mobile**: Element Call integration, Android/iOS builds, push notifications
5. **Polish**: Multi-account, i18n, accessibility, themes, app store distribution
