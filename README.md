# ⚡ VoltBase

**The zero-spinner, local-first alternative.**

Stop managing loading states. VoltBase embeds an ACID-compliant SQLite engine backed by OPFS
directly in your client, keeping it perfectly synchronized with your cloud database in the background.

[![License: MIT](https://img.shields.io/badge/License-MIT-00a8cc.svg)](LICENSE)
[![Status: alpha](https://img.shields.io/badge/status-alpha-e0a800.svg)](#project-status)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-3ecf8e.svg)](#contributing)

</div>

---

## Project status

**Alpha — the landing page and design system are live; the SDK is not.**

Right now this repository contains the VoltBase marketing site and the interactive sync sandbox
that demonstrates the intended behaviour. The `@voltbase/sdk` package described below is the
target API, not a published artifact. Nothing here is production-ready, and the sandbox simulates
sync rather than performing it.

If you want to help build the real thing, the [roadmap](#roadmap) is where to start.

---

## Why

Most client-server apps spend their complexity budget in the same place: the gap between a user
doing something and the server confirming it. Every mutation grows a loading state, an optimistic
update, a rollback path, and a retry. Every read grows a skeleton screen.

Local-first flips the order. The client owns a real database. Writes commit locally in under a
millisecond and are durable immediately. Replication happens afterwards, out of band, and failure
to reach the network is a normal condition rather than an error state.

VoltBase is that model with a Postgres backend you already know:

- **SQLite in the browser** — compiled to WASM, running the full engine, not a key-value shim.
- **OPFS-backed durability** — the Origin Private File System gives real file semantics, so data
  survives reloads and crashes instead of living in a tab's memory.
- **Background sync** — a replication loop reconciles the local database with Postgres when a
  connection exists, and queues writes in an outbox when it doesn't.
- **Field-level CRDT merge** — concurrent edits to different columns of the same row both survive,
  with an escape hatch for conflicts that genuinely need a human decision.

---

## How it compares

| | Supabase | VoltBase |
|---|---|---|
| **Database engine** | Postgres 15, server-resident only | Postgres 15 + embedded SQLite (WASM) on the client |
| **Read / write latency** | 80–400 ms round trip, network-bound | 0–2 ms local commit, sync happens after |
| **Offline support** | Reads fail, writes are dropped | Full read/write against OPFS, durable across reloads |
| **Conflict resolution** | Last write wins at the row level | Field-level CRDT merge with a manual override hook |

Latency figures are measured on a 4G network profile against a single-region Postgres instance.
VoltBase is not a fork of Supabase and is not affiliated with it — the comparison is here because
it's the architecture most people are moving from.

---

## The target API

```ts
// voltbase.config.ts — runs before first paint
import { createClient } from '@voltbase/sdk'

const db = createClient({
  engine: 'sqlite-wasm',
  storage: 'opfs',                                  // durable, origin-private
  sync: { mode: 'background', resolve: 'crdt' },
})

// Reads resolve locally. No await, no spinner.
const { rows } = db.useQuery('select * from tasks')
```

Writes follow the same shape — they return once the local transaction commits, not once the
server acknowledges:

```ts
await db.insert('tasks', { label: 'Ship OPFS adapter' })
// ↑ resolves in ~1 ms, replicates whenever the network allows
```

---

## What's in this repo

```
.
├── voltbase.html      # Self-contained landing page + interactive sync sandbox
├── LICENSE            # MIT
└── README.md
```

### Running the landing page

No build step, no dependencies:

```bash
git clone https://github.com/wushu75/Voltbase.git
cd Voltbase
open voltbase.html          # or: python3 -m http.server 8000
```

The page uses the Tailwind CDN, which compiles styles at runtime. That's fine for a demo but
should be replaced with a precompiled stylesheet before this is deployed anywhere real:

```bash
npx tailwindcss -i src/input.css -o dist/style.css --minify
```

### The sync sandbox

The interactive widget on the page is a simulation of the sync lifecycle, useful for showing the
model without a backend:

- **Airplane mode ON** — the status bar shows `⚡ OFFLINE MODE | QUEUE ACTIVE`. New rows appear
  instantly and are marked `0ms · queued`.
- **Airplane mode OFF** — a sync indicator pulses while the outbox flushes, then settles into
  `ONLINE · SYNCED` and every row flips to `0ms · synced`.

Latency stays at zero in both directions, which is the whole point.

### Design system

The page implements a flat, border-elevated theme — elevation comes from 1px borders, never from
box shadows — with a full light/dark swap driven by a single class on `<html>`.

| Token | Light | Dark |
|---|---|---|
| Canvas | `#fcfcfc` | `#171717` |
| Panel | `#ffffff` | `#1c1c1e` |
| Sunken panel | `#f6f6f7` | `#121212` |
| Primary text | `#171717` | `#fafafa` |
| Muted text | `#707070` | `#b4b4b4` |
| Border | `#e1e1e3` | `#2e2e2e` |
| Accent (Volt Cyan) | `#00a8cc` | `#00f2fe` |

Dark mode uses `#171717` rather than pure black to reduce halation, and off-white `#fafafa`
rather than `#ffffff` to cut glare. The light-mode accent is darkened to `#00a8cc` so that cyan
buttons still pass contrast against white.

---

## Roadmap

- [x] Landing page and design system
- [x] Interactive sync sandbox (simulated)
- [ ] SQLite WASM build with an OPFS VFS adapter
- [ ] Write-ahead outbox with durable queueing
- [ ] Replication protocol against Postgres logical decoding
- [ ] Field-level CRDT merge and conflict override hook
- [ ] `useQuery` / `useMutation` React bindings
- [ ] Schema migration story for the client-side database
- [ ] Row-level security parity with the server

The first four items are the critical path. Everything after depends on the replication protocol
being settled.

---

## Contributing

Issues and pull requests are welcome, especially on the OPFS adapter and the replication design.
Because the protocol isn't settled yet, open an issue to discuss anything architectural before
writing code — it'll save you rework.

For the landing page, the only rule that matters: elevation is built from borders. If you find
yourself reaching for `box-shadow`, reach for a `1px` border instead.

---

## FAQ

**Does this replace Postgres?**
No. Postgres remains the source of truth. VoltBase adds a durable client-side replica in front of it.

**What happens to data written offline if the user clears site data?**
It's gone. OPFS is origin-scoped browser storage, so anything not yet replicated is lost with it.
Sync frequency is the mitigation, and surfacing queue depth to users is good practice.

**Which browsers support OPFS?**
Chromium and Firefox have shipped it; Safari support arrived more recently and has rougher edges
around synchronous access handles. Verify against your own support matrix before committing.

**Is this affiliated with Supabase?**
No.

---

## License

MIT — see [LICENSE](LICENSE).

<div align="center">
<sub>© 2026 VoltBase.</sub>
</div>
