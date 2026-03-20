# Enact - Cross-Platform Matrix Client Implementation Plan

## Context

Build "Enact", a cross-platform Matrix client targeting Windows, Linux, Android, iOS, and browsers. The project uses Tauri V2 + React/TypeScript for the UI, with matrix-rust-sdk for native platforms and matrix-js-sdk for the browser. Two distinct UI skins (Discord-like and Telegram-like) ship from a single codebase via build-time configuration. Voice/video calls integrate via Element Call widget.

The repo is currently empty (just LICENSE + README.md) on branch `claude/matrix-client-multiplatform-tMbIQ`.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  React/TypeScript UI                 │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Discord Skin │  │ Telegram Skin│  │ Shared UI  │ │
│  └─────────────┘  └──────────────┘  └────────────┘ │
├─────────────────────────────────────────────────────┤
│              Platform Abstraction Layer              │
│         (unified API: Tauri commands OR JS SDK)      │
├──────────────────────┬──────────────────────────────┤
│   Desktop/Mobile     │         Browser              │
│   (Tauri V2)         │    (Vite SPA)                │
│   matrix-rust-sdk    │    matrix-js-sdk             │
│   via Tauri commands │    direct API calls          │
└──────────────────────┴──────────────────────────────┘
```

### SDK Strategy
- **Desktop (Windows, Linux)**: matrix-rust-sdk integrated directly as Tauri Rust backend, exposed via `#[tauri::command]`
- **Mobile (Android, iOS)**: matrix-rust-sdk via Tauri V2 mobile support (same Rust backend)
- **Browser**: matrix-js-sdk (user-approved pragmatic exception since full WASM bindings are experimental)

### Platform Abstraction Layer
A TypeScript abstraction layer sits between the UI and the SDK. On Tauri platforms, it calls `invoke()` to reach the Rust backend. On browser, it calls matrix-js-sdk directly. The UI code never directly touches either SDK.

---

## Project Structure

```
Enact/
├── package.json                    # Root: React/TS dependencies
├── vite.config.ts                  # Vite bundler config
├── tailwind.config.ts              # Tailwind with theme tokens
├── tsconfig.json
├── index.html
│
├── src/                            # React/TypeScript frontend
│   ├── main.tsx                    # App entry point
│   ├── App.tsx                     # Root component (skin router)
│   │
│   ├── platform/                   # Platform abstraction layer
│   │   ├── index.ts                # Exports unified MatrixClient interface
│   │   ├── types.ts                # Shared types (Room, Message, User, etc.)
│   │   ├── tauri-backend.ts        # Tauri invoke() implementation
│   │   └── browser-backend.ts      # matrix-js-sdk implementation
│   │
│   ├── components/                 # UI components
│   │   ├── shared/                 # Skin-agnostic components
│   │   │   ├── MessageBubble.tsx
│   │   │   ├── Avatar.tsx
│   │   │   ├── MediaViewer.tsx
│   │   │   ├── EmojiPicker.tsx
│   │   │   └── ...
│   │   ├── discord/                # Discord skin components
│   │   │   ├── DiscordLayout.tsx
│   │   │   ├── ServerSidebar.tsx
│   │   │   ├── ChannelList.tsx
│   │   │   ├── MemberList.tsx
│   │   │   └── ...
│   │   └── telegram/               # Telegram skin components
│   │       ├── TelegramLayout.tsx
│   │       ├── ChatList.tsx
│   │       ├── ChatView.tsx
│   │       └── ...
│   │
│   ├── hooks/                      # React hooks
│   │   ├── useMatrix.ts            # Main SDK hook
│   │   ├── useRoom.ts
│   │   ├── useMessages.ts
│   │   ├── useAuth.ts
│   │   └── ...
│   │
│   ├── stores/                     # State management (Zustand)
│   │   ├── auth-store.ts
│   │   ├── room-store.ts
│   │   ├── message-store.ts
│   │   └── settings-store.ts
│   │
│   ├── styles/                     # Theme tokens
│   │   ├── discord.css             # Discord color palette & layout tokens
│   │   ├── telegram.css            # Telegram color palette & layout tokens
│   │   └── base.css                # Shared base styles
│   │
│   └── lib/                        # Utilities
│       ├── markdown.ts
│       ├── media.ts
│       ├── crypto.ts
│       └── i18n.ts
│
├── src-tauri/                      # Tauri V2 Rust backend
│   ├── Cargo.toml                  # Rust dependencies (matrix-sdk, etc.)
│   ├── tauri.conf.json             # Base Tauri config
│   ├── tauri.discord.conf.json     # Discord skin overrides (app name, icons)
│   ├── tauri.telegram.conf.json    # Telegram skin overrides
│   ├── build.rs
│   ├── capabilities/
│   │   └── default.json
│   ├── icons/
│   │   ├── discord/                # Discord-themed app icons
│   │   └── telegram/               # Telegram-themed app icons
│   └── src/
│       ├── lib.rs                  # Tauri app setup + command registration
│       ├── main.rs                 # Desktop entry point
│       ├── commands/               # Tauri commands (exposed to frontend)
│       │   ├── mod.rs
│       │   ├── auth.rs             # Login, logout, SSO, QR code
│       │   ├── rooms.rs            # Room CRUD, join, leave, invite
│       │   ├── messages.rs         # Send, edit, delete, search
│       │   ├── sync.rs             # Sliding sync management
│       │   ├── crypto.rs           # E2EE, cross-signing, key backup
│       │   ├── media.rs            # Upload/download files, images, video
│       │   ├── notifications.rs    # Push notification handling
│       │   └── presence.rs         # User presence, typing indicators
│       ├── state.rs                # App state (MatrixClient singleton)
│       └── error.rs                # Error types
│
├── .env.discord                    # VITE_UI_SKIN=discord
├── .env.telegram                   # VITE_UI_SKIN=telegram
│
├── scripts/
│   ├── build-discord.sh            # Build Discord variant
│   └── build-telegram.sh           # Build Telegram variant
│
├── LICENSE
└── README.md
```

---

## Dual Skin Architecture (Tailwind CSS)

### Build-Time Skin Selection
- `VITE_UI_SKIN` env var set to `"discord"` or `"telegram"` at build time
- `App.tsx` conditionally renders `<DiscordLayout />` or `<TelegramLayout />`
- CSS token file loaded based on skin: `discord.css` or `telegram.css`
- Tauri config overrides for app name/icons: `tauri.discord.conf.json` / `tauri.telegram.conf.json`

### Build Commands
```bash
# Discord variant
vite build --mode discord && tauri build --config src-tauri/tauri.discord.conf.json

# Telegram variant
vite build --mode telegram && tauri build --config src-tauri/tauri.telegram.conf.json

# Browser (Discord)
VITE_UI_SKIN=discord vite build

# Browser (Telegram)
VITE_UI_SKIN=telegram vite build
```

### Layout Differences

**Discord Layout:**
```
┌────┬──────────┬─────────────────┬──────────┐
│    │ Channels │   Messages      │ Members  │
│ S  │          │                 │ List     │
│ e  │ #general │  [messages...]  │          │
│ r  │ #random  │                 │ User1    │
│ v  │ #dev     │  [composer]     │ User2    │
│ e  │          │                 │          │
│ r  │          │                 │          │
│ s  │          │                 │          │
└────┴──────────┴─────────────────┴──────────┘
```

**Telegram Layout:**
```
┌──────────────┬──────────────────────────────┐
│  Chat List   │         Messages             │
│              │                              │
│  John Doe    │    [messages...]              │
│  Group Chat  │                              │
│  Alice       │    [composer]                │
│              │                              │
└──────────────┴──────────────────────────────┘
```

### Tailwind Theme Tokens
Each skin defines CSS custom properties consumed by Tailwind:
- `--color-primary`, `--color-sidebar`, `--color-bg`, `--color-text`, etc.
- `--radius-message`, `--spacing-sidebar`, etc.
- Discord: dark theme by default (#36393F backgrounds, #5865F2 accent)
- Telegram: light theme by default (#FFFFFF backgrounds, #0088CC accent)

---

## Voice/Video Calls: Element Call Widget

Element Call is embedded as a widget (iframe) inside the app:
- **Embedded package** mode (widget-only, no standalone UI)
- Communication via Matrix widget postMessage API
- Tauri handles permissions (camera, microphone) via capabilities
- Browser version uses standard WebRTC permissions
- Requires a MatrixRTC backend (LiveKit SFU) on the homeserver side

### Integration Points
- `src/components/shared/CallWidget.tsx` - Element Call iframe wrapper
- `src-tauri/src/commands/calls.rs` - Widget API bridge for Tauri
- Room state events for call signaling handled by the SDK

---

## Implementation Phases

### Phase 1: Project Scaffolding & Core Auth (Start Here)
**Goal: Working app that can log in, list rooms, and display messages**

1. Initialize Tauri V2 + React + TypeScript project (`create-tauri-app`)
2. Set up Rust workspace with `matrix-sdk` dependency
3. Set up Tailwind CSS with dual theme token files
4. Implement platform abstraction layer (types + Tauri backend stub)
5. Implement auth commands: login (password), logout, session persistence
6. Implement sliding sync for room list
7. Basic room list UI (both skins)
8. Basic message timeline (both skins)
9. Message composer (send text messages)
10. E2EE setup (encryption enabled by default)

**Key Files:**
- `src-tauri/Cargo.toml` — add `matrix-sdk` with features: `e2e-encryption`, `sqlite`, `sliding-sync`
- `src-tauri/src/commands/auth.rs` — login/logout commands
- `src-tauri/src/commands/sync.rs` — sliding sync loop
- `src-tauri/src/commands/messages.rs` — send/receive
- `src/platform/tauri-backend.ts` — Tauri invoke wrappers
- `src/components/discord/DiscordLayout.tsx`
- `src/components/telegram/TelegramLayout.tsx`

### Phase 2: Enhanced Messaging
**Goal: Full messaging feature set**

- Message editing and deletion
- Reply threading
- Reactions/emoji reactions
- Read receipts
- Typing indicators
- Rich text/markdown formatting
- Code block formatting
- LaTeX rendering
- Message pinning
- Message forwarding
- Message search
- Message timestamps
- Unread indicators + badge counts
- Jump to unread / jump to date

### Phase 3: Media & Files
**Goal: Full media support**

- File sharing (upload/download)
- Image and video sharing
- Image paste from clipboard
- Drag-and-drop file uploads
- Voice messages (record + playback)
- Link previews / URL previews
- Media gallery per room
- Sticker packs
- Custom emoji

### Phase 4: Room & Social Features
**Goal: Full room management and social features**

- Room creation and management
- Direct messaging
- Group chats
- Spaces (organized room groupings)
- Room directory browsing
- Room invite management
- Room bookmarks/favorites
- Room history export
- User presence/status
- Custom status messages
- User mentions / room mentions
- User ignore/block

### Phase 5: Security & Device Management
**Goal: Full cryptographic feature set**

- Cross-signing device verification
- Key backup and recovery
- SSO login
- QR code login
- Device management
- Session verification
- Multi-account support

### Phase 6: Calls & Real-Time
**Goal: Voice/video calling via Element Call**

- Element Call widget integration
- Voice calls
- Video calls
- Screen sharing
- Push notifications
- Notification customization per room

### Phase 7: Browser Support
**Goal: Browser version using matrix-js-sdk**

- Implement `browser-backend.ts` using matrix-js-sdk
- Platform detection (Tauri vs browser)
- Browser-specific build configuration
- Service worker for offline support
- Browser notification API integration

### Phase 8: Mobile Deployment
**Goal: Android + iOS via Tauri V2 mobile**

- Tauri mobile project initialization (`tauri android init`, `tauri ios init`)
- Mobile-specific UI adjustments (responsive layout)
- Mobile push notifications (FCM/APNs)
- Mobile-specific capabilities (camera, file picker)
- App store preparation

### Phase 9: Polish & Accessibility
**Goal: Production-ready quality**

- Dark mode / theming (within each skin)
- Compact/comfortable layout toggle
- Font size customization
- Keyboard shortcuts
- Accessibility features (ARIA, screen reader support)
- Localization/language support (i18n)
- Low-bandwidth mode
- In-app browser for links
- Offline message queuing
- Sync across devices

---

## Key Dependencies

### Rust (src-tauri/Cargo.toml)
```toml
[dependencies]
tauri = { version = "2", features = ["protocol-asset"] }
matrix-sdk = { version = "0.9", features = ["e2e-encryption", "sqlite", "sliding-sync"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
```

### TypeScript (package.json)
```json
{
  "dependencies": {
    "@tauri-apps/api": "^2",
    "react": "^19",
    "react-dom": "^19",
    "zustand": "^5",
    "matrix-js-sdk": "^36"  // browser-only, tree-shaken out of Tauri builds
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2",
    "vite": "^6",
    "tailwindcss": "^4",
    "typescript": "^5"
  }
}
```

---

## Verification Plan

After Phase 1 implementation:
1. `pnpm install` succeeds
2. `pnpm tauri dev` launches a desktop window
3. Login with a Matrix homeserver (e.g., matrix.org) works
4. Room list populates via sliding sync
5. Clicking a room shows message timeline
6. Sending a message works and appears in the timeline
7. Both Discord and Telegram skins render correctly:
   - `VITE_UI_SKIN=discord pnpm tauri dev`
   - `VITE_UI_SKIN=telegram pnpm tauri dev`
8. E2EE is active (messages are encrypted in E2EE rooms)

---

## Initial Implementation Scope

For the first commit/PR, we will implement **Phase 1** (scaffolding + core auth + basic messaging). This gives us a working foundation that can be iterated on. The subsequent phases will be implemented incrementally.
