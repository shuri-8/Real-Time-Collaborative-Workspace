# Real-Time-Collaborative-Workspace
# Collab Workspace

A real-time collaborative workspace in the style of Notion/Figma: nested pages, a rich-text editor that merges concurrent and offline edits (CRDT), live cursors, roles, comments and version history. It scales horizontally: two WebSocket servers share state through Redis Pub/Sub.

**Stack:** Next.js 14 (App Router) · TypeScript · Tailwind · TipTap · Yjs · `ws` · Redis (`ioredis`) · PostgreSQL · Prisma · JWT cookie auth · Docker

```
collab-workspace/
├── docker-compose.yml        postgres, redis, web:3000, ws-server-1:4001, ws-server-2:4002
├── Dockerfile                Next.js image
├── ws-server/                Yjs WebSocket server (Dockerfile, server.ts, room.ts, adapters.ts, memory.ts)
├── shared/                   rbac.ts, protocol.ts, yjs-utils.ts, auth-token.ts, access.ts (used by app + server + tests)
├── prisma/                   schema.prisma, seed.ts
├── src/                      Next.js app (pages, /api routes, components, browser provider)
├── tests/                    crdt-unit.test.ts, load-test.ts, helpers/
└── docs/                     ARCHITECTURE.md, API.md, PRESENTATION.md
```

## 1. Prerequisites

- **Node.js 20+** (`node -v`)
- **Docker Desktop** or Docker Engine with Compose (`docker compose version`)

## 2. Setup, step by step

```bash
# 1. Configure
cp .env.example .env
#    edit JWT_SECRET to any long random string

# 2. Install dependencies (also generates the Prisma client)
npm install
npx prisma generate

# 3. Start only the databases first
docker-compose up -d postgres redis

# 4. Create the tables and load demo data
npx prisma migrate dev --name init
npx prisma db seed

# 5. Build and start everything (web + two WebSocket servers)
docker-compose up --build
```

Open **http://localhost:3000** and sign in with a demo account (password `password123`):

| User | Access in the demo workspace |
|---|---|
| alice@example.com | owner (full access) |
| bob@example.com | can edit every page |
| carol@example.com | view only |
| dave@example.com | guest: only the *Welcome* page, comment-only |

> `docker-compose` and `docker compose` are the same command; use whichever your install provides.

**Running without Docker for the apps** (hot reload): keep `postgres` and `redis` up, then run in three terminals: `npm run dev`, `npm run ws:1`, `npm run ws:2`.

## 3. Try every feature

1. **Auth and workspaces**: register a new account at `/register` (you get your own workspace and a first page), or sign in as Alice. Use **+ New page** and the **+** on a page to create nested pages. Rename a page in the header.
2. **Real-time editing**: open the *Welcome* page in two windows (one signed in as Alice in a normal window, one as Bob in a private window).
3. **Multi-instance Redis sync**: the page picks a WebSocket server per tab. Force them explicitly:
   - Tab A: `http://localhost:3000/workspace/demo-workspace/demo-page-welcome?ws=1` (server on **4001**)
   - Tab B: `http://localhost:3000/workspace/demo-workspace/demo-page-welcome?ws=2` (server on **4002**)

   The header shows `Synced · ws:4001` / `ws:4002`. Type in A and it appears in B, along with cursors and presence, even though the tabs are on different servers. Watch `docker-compose logs -f ws-server-1 ws-server-2`.
4. **Offline merge**: click **Go offline** in both tabs, type different things (even in the same paragraph), then click **Reconnect** in both. Everything merges.
5. **Presence**: coloured cursors with names, an avatar stack, and "N online" in the header.
6. **Permissions**: sign in as Carol (view only: no toolbar, typing is blocked; even a tampered client is rejected by the server). In Alice's window click **Share**, invite `carol@example.com` as *Can edit* and Carol can type immediately without reloading. Changing a role to *Can view* flips her back live. Invite a brand-new user (register first) to see a guest only get that page.
7. **Comments**: select text, click **Comment**, write it. The highlight follows the text while others type. Reply, resolve and delete from the panel. Carol (view) cannot comment; Dave (comment) can.
8. **Version history**: **History → Save version**. Change the text, select the old version, preview it, **Restore**. Every open tab updates live. Versions are also saved automatically when the last user leaves a page.
9. **Persistence**: stop and restart `docker-compose restart ws-server-1 ws-server-2`: documents rehydrate from PostgreSQL. Writes happen at most every 2 s while editing, and immediately when the last user disconnects.

## 4. Tests

```bash
npm test                  # unit + integration tests (no Docker needed)
npm run typecheck         # TypeScript for the app and the server
```

`tests/crdt-unit.test.ts` covers offline merge convergence, RBAC (pure rules and enforcement by the real room manager), snapshot restoration, debounced persistence + rehydration, and Redis-style multi-instance sync.

### Load test (50 clients, both servers)

With the stack running and the database seeded:

```bash
npm run load                                   # 50 clients, 20 s, 2 edits/s each, ports 4001 + 4002
npm run load -- --clients=50 --duration=30 --rate=3 --gutenberg
```

Without Docker (in-process servers using the same room code, in-memory store and bus):

```bash
npm run load -- --harness
```

It prints p50/p95/p99 propagation latency, throughput, memory usage (and each server's `/health`), then verifies all 50 clients and a fresh verifier hold the **identical document** and that every edit is present. Results are also written to `tests/load-test-results.json`. Exit code is non-zero if convergence fails.

## 5. Troubleshooting

| Symptom | Fix |
|---|---|
| `JWT_SECRET` error from compose | `cp .env.example .env` first |
| Editor says "Syncing…" forever / WebSocket 403 | Open the app at `http://localhost:3000` (not `127.0.0.1`), or add your origin to `ALLOWED_ORIGINS` |
| Port already in use | Stop local Postgres/Redis or change the published ports in `docker-compose.yml` |
| `migrate dev` cannot reach the database | `docker-compose ps`: postgres must be *healthy*; `DATABASE_URL` in `.env` must use `localhost` |
| Reset everything | `docker-compose down -v` then repeat step 3–5 |

## 6. Design notes and limits

- Auth is a signed JWT in an `httpOnly` cookie. Browsers send cookies to any port of `localhost`, so the WebSocket servers authenticate the same cookie; the WebSocket `Origin` is also checked. For production use HTTPS (`COOKIE_SECURE=true`) and put the web app and WebSocket servers on one parent domain.
- Comments refresh by polling every 4 s (the document itself is real time).
- Awareness (cursor/presence) payloads are trusted from authenticated users; it is not an authorization boundary.
- Page titles are plain metadata (not in the CRDT), so simultaneous renames are last-write-wins.

See `docs/ARCHITECTURE.md`, `docs/API.md` and `docs/PRESENTATION.md`.
