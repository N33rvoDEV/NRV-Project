Analisi stack tecnologici — NRV Project
NRV è un monorepo per un retroserver stile Habbo Hotel, organizzato in tre pacchetti indipendenti senza workspace root unificato (nessun package.json alla radice).

Browser
Backend
CDN
WS /ws
HTTP /api/* proxy
rewrite /assets/*
nrv-web :3002Next.js + React + PixiJS
nrv-core :3001Bun + WS binario + HTTP
SQLitePrisma
nrv-assets :8080Static files
1. nrv-core — Server emulatore
Layer	Tecnologia	Ruolo
Runtime
Bun 1.x
Bun.serve con HTTP + WebSocket nativo, hot reload in dev
Linguaggio
TypeScript 5.9 (strict, ESM)
Tutto il backend
Validazione
Zod 3.x
Input Zero-Trust su login, chat, placement, ecc.
Password
Argon2id (argon2)
Hash e verify credenziali
ORM
Prisma 6.x
Accesso dati, migrazioni, seed
Database
SQLite (dev.db)
Dev locale; schema pronto per swap adapter
Compressione
Pako
Decompressione bundle Nitro (tooling asset)
Architettura applicativa
Event-driven, factory functions, niente OOP legacy
Repository pattern: mock vs prisma via USER_REPOSITORY
Handler modulari: login, navigator, room, walk, chat, inventory, catalog, social
Protocollo binario custom (ispirato ad Arcturus per gli ID pacchetto): frame [int32 BE length][int16 BE headerId][body]
HTTP REST affiancato al WS: feed hotel, news, profili utente, registrazione, stats
Dominio persistito (Prisma)
User, Room, Furniture, RoomItem, CatalogPage, CatalogItem, HotelNews, Friendship, UserBadge, RoomRight, reazioni/commenti news, ecc.

2. nrv-web — Client web
Layer	Tecnologia	Versione	Ruolo
Framework
Next.js (App Router)
16.2.6
Routing, SSR/SSG, API routes BFF
UI library
React
19.2.4
Componenti client/server
Linguaggio
TypeScript
5.x
Type-safe end-to-end
Styling
Tailwind CSS
v4
Utility + @theme inline
PostCSS
@tailwindcss/postcss
v4
Pipeline CSS
Game engine
PixiJS
8.18.x
Rendering isometrico stanza
Pixi React
@pixi/react
8.0.5
In package.json, ma il canvas usa Pixi imperativo (Application in GameCanvas.tsx)
Lint
ESLint 9 + eslint-config-next
—
Qualità codice
Package manager
Bun
—
bun install / bun run dev
Architettura frontend
Area	Stack / pattern
Homepage /
React + CSS custom Habbo 2026 (habbo-2026.css, habbo-home.css, habbo-reception.css)
Auth
AuthContext + sessione in localStorage (auth-session)
Hotel /hotel
GameContext + SocketClient (unica connessione WS)
UI hotel
Sunrise OS (SunriseOS, SunriseWindow, HabboHotelBar)
Rendering stanza
GameCanvas → RoomScene → layer Pixi (floor, furni, avatar)
Copy/i18n
Centralizzato in hotel-copy.ts (italiano)
Font
Google Fonts via next/font (Inter, Bebas Neue, Nunito, Press Start 2P, Ubuntu, IBM Plex Mono)
API Routes (BFF Next.js → core)
Proxy verso nrv-core per evitare CORS e nascondere l’URL backend:

/api/auth/register
/api/hotel/feed, /api/hotel/news/*, /api/hotel/users/*
/api/avatar-image, /api/badge-image, /api/furni-asset
/api/stats/clients
Protocollo client
Mirror del core in src/network/protocol/:

ids.ts — header ID sincronizzati col server
packets.ts — encode/decode tipizzati
binary.ts — frame parser
SocketClient.ts — gestione connessione e dispatch eventi verso GameContext
3. nrv-assets — CDN statico
Aspetto	Dettaglio
Tipo
Repository di asset statici (no package.json)
Contenuto
Furni JSON/spritesheet, manifest Habbo, badge, promo, reception vista hotel
Serving
python3 -m http.server 8080 o server dedicato
Integrazione
Rewrite Next.js: /assets/* → NRV_ASSETS_URL (default http://127.0.0.1:8080)
Tooling
scripts/sync-habbo-assets.mjs (sync da CDN Habbo ufficiale)
Core tooling
assets:compile, assets:devour, assets:harvest, assets:download in nrv-core
Formato furni: JSON Habbo-style (*_data.json + spritesheet) caricati da Pixi Assets / Spritesheet.

4. Comunicazione e protocolli
GET / (SSR/SSG)
GET /assets/furni/...
rewrite proxy
WS ws://host:3001/ws (binario)
GET /api/hotel/feed
HTTP proxy NRV_CORE_HTTP_URL
Login 2419→2491, Navigator 249→2690, Room 2312→...
Browser
nrv-web
nrv-core
nrv-assets
Canale	Formato	Uso
WebSocket
Binario custom
Gioco real-time (hot path)
HTTP core
JSON
Feed, news, profili, registrazione
HTTP assets
File statici
Immagini, furni, avatar rendering
Next API
JSON
BFF tra browser e core
5. Sicurezza
Aspetto	Implementazione
Password
Argon2id lato core
Validazione input
Zod su tutti gli handler critici
WS
Solo messaggi binari (stringhe rifiutate con close 1008)
Auth client
Sessione locale + login WS con username/password
Zero-Trust
Nessuna fiducia implicita nel payload client
6. DevOps e ambiente
Elemento	Stato attuale
Containerizzazione
Nessun Dockerfile/compose nel repo
CI/CD
Non configurato nel repo
Monorepo tool
Nessun Turborepo/Nx; pacchetti autonomi
Porte dev
Web :3002, Core :3001, Assets :8080
Variabili d’ambiente chiave
Variabile	Pacchetto	Default
NEXT_PUBLIC_NRV_WS_URL
nrv-web
ws://127.0.0.1:3001/ws
NRV_CORE_HTTP_URL
nrv-web (server)
http://127.0.0.1:3001
NRV_ASSETS_URL
nrv-web
http://127.0.0.1:8080
DATABASE_URL
nrv-core
file:./dev.db
USER_REPOSITORY
nrv-core
mock / prisma
7. Osservazioni strategiche
Punti di forza

Separazione netta: UI (Next), logica gioco (Bun), asset (CDN)
Protocollo binario sul WS — adatto a latenza bassa
Repository pattern — migrazione DB senza toccare la logica di gioco
Stack moderno: React 19, Next 16, Tailwind 4, Pixi 8, Bun
Aree di attenzione

SQLite in dev — per produzione serve PostgreSQL + adapter Prisma
@pixi/react dichiarato ma non usato — candidato a rimozione o adozione futura
CSS ibrido — Tailwind + molti fogli custom Habbo (~7k righe in globals.css); funziona ma aumenta il debito di manutenzione
Nessun test runner visibile in package.json (né Jest, Vitest, Playwright)
Auth client-side — sessione in localStorage; per produzione servono ticket/JWT e hardening
Confronto con l’ecosistema Habbo classico

Habbo classico	NRV
Flash client
PixiJS + WebGL/Canvas
Java emulator (Arcturus/Morningstar)
Bun + TypeScript custom
MySQL
SQLite (dev) / Prisma
Nitro assets
nrv-assets + tool harvest/compile
In sintesi: NRV è uno stack TypeScript full-stack con Bun come runtime server, Next.js + React 19 come shell web, PixiJS come motore di gioco 2D isometrico, Prisma/SQLite per la persistenza e un protocollo binario WebSocket proprietario — tutto orchestrato da un monorepo a tre pacchetti senza orchestrazione centralizzata.
