# NRV Project

**Open-source Habbo Hotel–style retroserver** — isometric multiplayer world, rooms, furni, catalog, social features, and badges.

> *NRV o morte.*

🇮🇹 [Italian version](./README.md)

---

## Overview

NRV is a three-package monorepo: a **high-performance emulator** (Bun + binary WebSocket), a **modern web client** (Next.js + PixiJS), and a **static CDN** for furni assets and Habbo-style graphics.

```
Browser (:3002)
    │
    ├─► GET /assets/*  ──rewrite──►  nrv-assets (:8080)
    ├─► WS  /ws        ────────────►  nrv-core   (:3001)   ← real-time game
    └─► HTTP /api/*    ──proxy────►  nrv-core   (:3001)   ← feed, news, profiles
```

| Package | Role | Dev port |
|---------|------|----------|
| [`nrv-core`](./nrv-core) | Emulator server (HTTP + binary WebSocket, Prisma/SQLite) | `3001` |
| [`nrv-web`](./nrv-web) | Web client (Next.js, React, PixiJS, Sunrise OS UI) | `3002` |
| [`nrv-assets`](./nrv-assets) | Static assets (furni JSON, spritesheets, reception, badges) | `8080` |

---

## Features

- **Homepage** in Habbo 2026 style — reception gate, login, product picker, news
- **Hotel client** (`/hotel`) — Sunrise OS lobby, room navigator, live inventory and catalog over WebSocket
- **Isometric rendering** — floor, furni, and avatars with PixiJS 8
- **Custom binary protocol** — `[length][headerId][body]` frames, packet IDs inspired by Arcturus
- **Persistence** — users, rooms, furni, catalog, news, friendships, badges (Prisma + SQLite)
- **Security** — Argon2id, Zod validation, Zero-Trust client model

---

## Tech stack

### nrv-core

| Layer | Technology |
|-------|------------|
| Runtime | [Bun](https://bun.sh) |
| Language | TypeScript |
| Validation | Zod |
| Passwords | Argon2id |
| ORM / DB | Prisma + SQLite |

### nrv-web

| Layer | Technology |
|-------|------------|
| Framework | Next.js 16 (App Router) |
| UI | React 19 |
| Styling | Tailwind CSS v4 + Habbo 2026 design system |
| Game engine | PixiJS 8 |
| Language | TypeScript |

### nrv-assets

Static repository of furni, manifests, and promotional images. Served over HTTP and proxied by the Next.js client at `/assets/*`.

---

## Prerequisites

- [Bun](https://bun.sh) ≥ 1.1 (recommended runtime for core and web)
- [Python 3](https://www.python.org/) (only to serve `nrv-assets` in dev)
- Git

---

## Quick start

Clone the repository and start all three services in separate terminals.

### 1. Core (backend)

```bash
cd nrv-core
cp .env.example .env
bun install
bun run db:push
bun run db:generate
bun run db:seed      # dev user + welcome room
USER_REPOSITORY=prisma bun run dev
```

The engine listens on **http://127.0.0.1:3001** — WebSocket at `ws://127.0.0.1:3001/ws`.

### 2. Assets (local CDN)

```bash
cd nrv-assets
python3 -m http.server 8080
```

### 3. Web (client)

```bash
cd nrv-web
cp .env.example .env.local
bun install
bun run dev
```

Open **http://localhost:3002**.

### Development credentials

After running `db:seed` in `nrv-core`:

| Field | Value |
|-------|-------|
| Username | `nrvadmin` |
| Password | `NrvDevPass1!` |

---

## Environment variables

### nrv-core (`.env`)

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | Prisma connection string | `file:./dev.db` |
| `USER_REPOSITORY` | `mock` or `prisma` | `mock` (use `prisma` with a seeded DB) |

### nrv-web (`.env.local`)

| Variable | Description | Default |
|----------|-------------|---------|
| `NEXT_PUBLIC_NRV_WS_URL` | WebSocket URL to nrv-core | `ws://127.0.0.1:3001/ws` |
| `NRV_CORE_HTTP_URL` | Core HTTP URL (Next API routes) | `http://127.0.0.1:3001` |
| `NRV_ASSETS_URL` | Asset origin for server-side rewrite | `http://127.0.0.1:8080` |
| `NEXT_PUBLIC_NRV_ASSETS_URL` | Asset base URL in the browser | `/assets` |

---

## Repository structure

```
NRV-Project/
├── nrv-core/                 # Bun emulator
│   ├── prisma/                 # Schema, seed, migrations
│   └── src/
│       ├── index.ts            # HTTP + WS entry point
│       ├── networking/         # Binary protocol
│       ├── game/               # Room, furni, catalog, social handlers
│       ├── database/           # Repositories and Prisma adapters
│       └── security/           # Auth, Argon2, Zod
│
├── nrv-web/                    # Next.js client
│   └── src/
│       ├── app/                # App Router routes
│       ├── components/         # UI (sunrise/, habbo-site/, habbo-it/)
│       ├── context/            # GameContext, AuthContext
│       ├── game/               # RoomScene, PixiJS layers
│       └── network/            # SocketClient, packet codec
│
├── nrv-assets/                 # Static CDN
│   └── assets/
│       ├── furni/              # Furni JSON + spritesheets
│       └── habbo/              # Manifests and official promos
│
└── docs/                       # Internal documentation
```

---

## Network protocol

Wire format (identical on client and server):

```
[int32 BE totalLength][int16 BE headerId][body bytes…]
```

Habbo-style strings: `int16 BE utf8Length` + UTF-8.

Packet IDs are kept in sync in:

- `nrv-web/src/network/protocol/ids.ts`
- `nrv-core/src/networking/protocol/ids.ts`

| Flow | IN | OUT |
|------|-----|-----|
| Login | `2419` | `2491`, `2725`, `3475` |
| Navigator | `249` | `2690` |
| Room entry | `2312` | `758`, `2031`, `1301`, `1778` |
| Walking | `3320` | `1640` |
| Chat | `1314` | `1446` |
| Inventory | `3150` | `994` |
| Furni placement | `1258` | `1534` |
| Catalog | `1195`, `412` | `1032`, `804` |

---

## Useful scripts

### nrv-core

```bash
bun run dev              # Server with hot-reload
bun run start            # Local production start
bun run typecheck        # TypeScript check
bun run db:push          # Sync SQLite schema
bun run db:seed          # Seed user and room
bun run test:login       # WebSocket login handshake test
bun run furni:seed       # Seed furni catalog
bun run assets:compile   # Compile furni assets
```

### nrv-web

```bash
bun run dev              # Dev server on :3002
bun run build            # Production build
bun run start            # Start production build
bun run lint             # ESLint
bun run sync-habbo-assets  # Sync assets from Habbo CDN
```

---

## User flow

1. Visit `/` — homepage with reception gate, login, and news
2. Register at `/register` or log in at `/login`
3. Enter `/hotel` — WebSocket connection, Sunrise OS lobby
4. Navigator → pick a room → `GameCanvas` (PixiJS) renders the world
5. Chat, inventory, catalog, and furni — all via NRV binary packets

---

## Project philosophy

| Principle | Implementation |
|-----------|----------------|
| **Minimal latency** | Binary hot path over WebSocket, no JSON in-game |
| **Zero-Trust** | Zod on every input, Argon2 for passwords |
| **Modularity** | Swappable repositories, event-driven handlers |
| **Separation** | Core, client, and assets are independent packages |

---

## Disclaimer

NRV Project is a **fan / educational project** and is **not affiliated with, endorsed by, or sponsored by Sulake Corporation Oy** (Habbo Hotel). Habbo® is a trademark of its respective owners. Habbo-style assets are used for retro emulation and development purposes; review applicable licenses before any public distribution.

---

## License

Private project. Contact the maintainers for contribution and licensing information.

---

## Links

- [Monorepo documentation](./docs/gemini-dumps/00-MONOREPO-OVERVIEW.md)
- [nrv-core README](./nrv-core/README.md)
- [nrv-web README](./nrv-web/README.md)
