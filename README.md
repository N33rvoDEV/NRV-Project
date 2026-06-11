# NRV Project

**Retroserver open-source stile Habbo Hotel** — mondo isometrico multiplayer, stanze, furni, catalogo, social e badge.

> *NRV o morte.*

---

## Panoramica

NRV è un monorepo a tre pacchetti: un **emulatore ad alte prestazioni** (Bun + WebSocket binario), un **client web moderno** (Next.js + PixiJS) e un **CDN statico** per asset furni e grafica Habbo-style.

```
Browser (:3002)
    │
    ├─► GET /assets/*  ──rewrite──►  nrv-assets (:8080)
    ├─► WS  /ws        ────────────►  nrv-core   (:3001)   ← gioco real-time
    └─► HTTP /api/*    ──proxy────►  nrv-core   (:3001)   ← feed, news, profili
```

| Pacchetto | Ruolo | Porta dev |
|-----------|--------|-----------|
| [`nrv-core`](./nrv-core) | Server emulatore (HTTP + WebSocket binario, Prisma/SQLite) | `3001` |
| [`nrv-web`](./nrv-web) | Client web (Next.js, React, PixiJS, Sunrise OS UI) | `3002` |
| [`nrv-assets`](./nrv-assets) | Asset statici (furni JSON, spritesheet, reception, badge) | `8080` |

---

## Funzionalità

- **Homepage** stile Habbo 2026 — reception, login, product picker, news
- **Hotel client** (`/hotel`) — lobby Sunrise OS, navigatore stanze, inventario e catalogo live via WebSocket
- **Rendering isometrico** — pavimento, furni e avatar con PixiJS 8
- **Protocollo binario custom** — frame `[length][headerId][body]`, ID ispirati ad Arcturus
- **Persistenza** — utenti, stanze, furni, catalogo, news, amicizie, badge (Prisma + SQLite)
- **Sicurezza** — Argon2id, validazione Zod, Zero-Trust sul client

---

## Stack tecnologico

### nrv-core

| Layer | Tecnologia |
|-------|------------|
| Runtime | [Bun](https://bun.sh) |
| Linguaggio | TypeScript |
| Validazione | Zod |
| Password | Argon2id |
| ORM / DB | Prisma + SQLite |

### nrv-web

| Layer | Tecnologia |
|-------|------------|
| Framework | Next.js 16 (App Router) |
| UI | React 19 |
| Styling | Tailwind CSS v4 + design system Habbo 2026 |
| Game engine | PixiJS 8 |
| Linguaggio | TypeScript |

### nrv-assets

Repository statico di furni, manifest e immagini promozionali. Servito via HTTP e proxato dal client Next.js su `/assets/*`.

---

## Prerequisiti

- [Bun](https://bun.sh) ≥ 1.1 (runtime consigliato per core e web)
- [Python 3](https://www.python.org/) (solo per servire `nrv-assets` in dev)
- Git

---

## Avvio rapido

Clona il repository e avvia i tre servizi in terminali separati.

### 1. Core (backend)

```bash
cd nrv-core
cp .env.example .env
bun install
bun run db:push
bun run db:generate
bun run db:seed      # utente dev + stanza di benvenuto
USER_REPOSITORY=prisma bun run dev
```

Il motore ascolta su **http://127.0.0.1:3001** — WebSocket su `ws://127.0.0.1:3001/ws`.

### 2. Assets (CDN locale)

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

Apri **http://localhost:3002**.

### Credenziali di sviluppo

Dopo `db:seed` in `nrv-core`:

| Campo | Valore |
|-------|--------|
| Username | `nrvadmin` |
| Password | `NrvDevPass1!` |

---

## Variabili d'ambiente

### nrv-core (`.env`)

| Variabile | Descrizione | Default |
|-----------|-------------|---------|
| `DATABASE_URL` | Connection string Prisma | `file:./dev.db` |
| `USER_REPOSITORY` | `mock` o `prisma` | `mock` (usa `prisma` con DB seedato) |

### nrv-web (`.env.local`)

| Variabile | Descrizione | Default |
|-----------|-------------|---------|
| `NEXT_PUBLIC_NRV_WS_URL` | URL WebSocket verso nrv-core | `ws://127.0.0.1:3001/ws` |
| `NRV_CORE_HTTP_URL` | URL HTTP core (API routes Next) | `http://127.0.0.1:3001` |
| `NRV_ASSETS_URL` | Origin asset server-side (rewrite) | `http://127.0.0.1:8080` |
| `NEXT_PUBLIC_NRV_ASSETS_URL` | Base URL asset lato browser | `/assets` |

---

## Struttura del repository

```
NRV-Project/
├── nrv-core/                 # Emulatore Bun
│   ├── prisma/                 # Schema, seed, migrazioni
│   └── src/
│       ├── index.ts            # Entry point HTTP + WS
│       ├── networking/         # Protocollo binario
│       ├── game/               # Handler stanza, furni, catalogo, social
│       ├── database/           # Repository e adapter Prisma
│       └── security/           # Auth, Argon2, Zod
│
├── nrv-web/                    # Client Next.js
│   └── src/
│       ├── app/                # Route App Router
│       ├── components/         # UI (sunrise/, habbo-site/, habbo-it/)
│       ├── context/            # GameContext, AuthContext
│       ├── game/               # RoomScene, layer PixiJS
│       └── network/            # SocketClient, codec pacchetti
│
├── nrv-assets/                 # CDN statico
│   └── assets/
│       ├── furni/              # JSON + spritesheet furni
│       └── habbo/              # Manifest e promo ufficiali
│
└── docs/                       # Documentazione interna
```

---

## Protocollo di rete

Wire format (identico tra client e server):

```
[int32 BE totalLength][int16 BE headerId][body bytes…]
```

Stringhe in stile Habbo: `int16 BE utf8Length` + UTF-8.

Gli ID pacchetto sono sincronizzati in:

- `nrv-web/src/network/protocol/ids.ts`
- `nrv-core/src/networking/protocol/ids.ts`

| Flusso | IN | OUT |
|--------|-----|-----|
| Login | `2419` | `2491`, `2725`, `3475` |
| Navigatore | `249` | `2690` |
| Entrata stanza | `2312` | `758`, `2031`, `1301`, `1778` |
| Camminata | `3320` | `1640` |
| Chat | `1314` | `1446` |
| Inventario | `3150` | `994` |
| Placement furni | `1258` | `1534` |
| Catalogo | `1195`, `412` | `1032`, `804` |

---

## Script utili

### nrv-core

```bash
bun run dev              # Server con hot-reload
bun run start            # Avvio produzione locale
bun run typecheck        # Controllo TypeScript
bun run db:push          # Sincronizza schema SQLite
bun run db:seed          # Seed utente e stanza
bun run test:login       # Test handshake WebSocket
bun run furni:seed       # Seed catalogo furni
bun run assets:compile   # Compila asset furni
```

### nrv-web

```bash
bun run dev              # Dev server su :3002
bun run build            # Build produzione
bun run start            # Avvio build
bun run lint             # ESLint
bun run sync-habbo-assets  # Sync asset da CDN Habbo
```

---

## Flusso utente

1. Visita `/` — homepage con gate reception, login e news
2. Registrazione su `/register` o login su `/login`
3. Entra in `/hotel` — connessione WebSocket, lobby Sunrise OS
4. Navigatore → seleziona stanza → `GameCanvas` (PixiJS) renderizza il mondo
5. Chat, inventario, catalogo e furni — tutto via pacchetti binari NRV

---

## Filosofia di progetto

| Principio | Implementazione |
|-----------|-----------------|
| **Latenza minima** | Hot path binario su WebSocket, niente JSON nel gioco |
| **Zero-Trust** | Zod su ogni input, Argon2 sulle password |
| **Modularità** | Repository swappable, handler event-driven |
| **Separazione** | Core, client e asset sono pacchetti indipendenti |

---

## Disclaimer

NRV Project è un **progetto fan / educativo** e **non è affiliato, approvato o sponsorizzato da Sulake Corporation Oy** (Habbo Hotel). Habbo® è un marchio dei rispettivi proprietari. Gli asset Habbo-style sono utilizzati a scopo di emulazione retro e sviluppo; verifica le licenze applicabili prima di qualsiasi distribuzione pubblica.

---

## Licenza

Progetto privato. Contatta i maintainer per informazioni su contribuzioni e licenza.

---
- [README nrv-core](./nrv-core/README.md)
- [README nrv-web](./nrv-web/README.md)
