# Tenure — Loan EMI Workspace

A loan EMI calculator where every open browser tab is a window into the same shared workspace. Change the loan amount in one tab and every other tab updates instantly — no server, no polling, no `localStorage` event hacks. Tabs talk to each other directly over the [`BroadcastChannel` API](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API).

## Live Demo & Repo

| | Link |
|---|---|
| **Live** | _[add your deployment URL after deploying]_ |
| **Repo** | https://github.com/ketarora/EMI-Calculator |

## Running Locally

Requires Node 18.18+ (Node 20/22 recommended).

```bash
npm install
npm run dev
```

Open http://localhost:3000, then open the same URL in a second tab to see the sync in action.

| Command | Description |
|---------|-------------|
| `npm run dev` | Start dev server |
| `npm run build` | Production build |
| `npm run start` | Serve production build |
| `npm run test` | Run reducer + finance math + live sync tests |
| `npm run lint` | ESLint |

## Tech Stack

Next.js 14 (App Router) · React 18 (hooks only) · TypeScript (strict) · Tailwind CSS · Recharts · PapaParse · BroadcastChannel

No backend, no state-management library, no UI component library — everything in `components/ui` is hand-rolled.

---

## Features

### Core

| Feature | Details |
|---------|---------|
| **EMI Calculator** | Amount (₹10K–₹50L), rate (1–36%), tenure (1–84mo) with synced slider + number inputs |
| **Summary Cards** | Monthly EMI, total interest, total payable, principal vs interest split bar |
| **Amortization Schedule** | Paginated table (12 rows/page), break-even row highlighted, table ↔ stacked bar chart toggle |
| **Comparison Mode** | Up to 3 scenarios side by side, lowest total cost highlighted |
| **Sensitivity Grid** | Rate × tenure what-if table (±1/2/3% and ±6/12/24mo), current cell highlighted |
| **Prepayment Planner** | Schedule lump-sum payments, see interest saved and tenure reduced |
| **Cross-Tab Sync** | All inputs, scenarios, prepayments, mode, and theme sync via BroadcastChannel |
| **Tab Identity** | Unique tab ID, live tab count, leader badge |
| **Dark / Light Theme** | Synced across all tabs like any other field |

### Bonus

| Feature | Details |
|---------|---------|
| **Tab Leadership** | New tabs get state from the leader; auto re-election when leader closes |
| **Cross-Tab Undo** | Ctrl/Cmd+Z reverts the last change across all tabs |
| **CSV Export** | Download amortization schedule as CSV (PapaParse) |
| **URL State** | `?amount=1500000&rate=11&tenure=48` loads and stays in sync |

---

## Architecture Overview

### Cross-Tab Sync — Two Channels

```
┌──────────────────────────┐     ┌──────────────────────────┐
│  PRESENCE channel         │     │  DATA channel              │
├──────────────────────────┤     ├──────────────────────────┤
│  HEARTBEAT (every 1s)     │     │  REQUEST_INIT              │
│  BYE (on clean unload)    │     │  SYNC_INIT                 │
│                            │     │  DISPATCH_ACTION           │
│  → who's alive, leader    │     │  → document state          │
└──────────────────────────┘     └──────────────────────────┘
     hooks/usePresence.ts          context/WorkspaceProvider.tsx
```

Presence (heartbeats) and data (state mutations) are on separate channels so they never interfere with each other.

### Sync Model

- **Primary path:** Every user edit is dispatched locally first (optimistic), then broadcast as a `DISPATCH_ACTION`. Every other tab applies the same action to its own reducer.
- **Fallback path:** A new tab broadcasts `REQUEST_INIT`. The leader replies with `SYNC_INIT` (full state snapshot). If nobody answers within 700ms, it keeps defaults.

### Leader Election

Each tab heartbeats its `tabId` and `joinedAt` every second. Leader = earliest `joinedAt` among surviving peers. No votes, no consensus — works because BroadcastChannel is a true local broadcast medium. Re-election happens naturally when a peer drops off.

### Shared vs Local State

- **`SharedState`** — calculator inputs, theme, mode, scenarios, prepayments, undo history. Identical across all tabs.
- **`PresenceSnapshot`** — tab ID, leadership, tab count. Per-tab, never serialized into shared state.

### Reducer Purity

The reducer never calls `Date.now()`, `Math.random()`, or `crypto.randomUUID()`. Entity IDs are generated at the dispatch call site and carried in the action payload — so every tab replaying the same action computes the same result. This was a real bug caught by `tests/live-sync-check.ts` and fixed.

### Cross-Tab Undo

The undo stack (`history.past`) is just another field on `SharedState` — it replicates automatically through the same `DISPATCH_ACTION` pipeline. No special wiring needed.

### SSR / Hydration

Tab identity is generated in `useEffect` after mount (never during render) to avoid hydration mismatches. `useSearchParams()` is wrapped in `<Suspense>` per Next.js App Router conventions.

---

## Project Structure

```
src/
  app/                      Next.js App Router entry
  components/
    workspace/              Header, tab presence, theme toggle
    calculator/             Input sliders, summary cards, sensitivity grid
    amortization/           Paginated table, Recharts chart, CSV export
    comparison/             Compare-mode scenario cards
    prepayment/             Prepayment planner
    ui/                     Card, Button, Badge, SegmentedControl
  context/
    workspaceReducer.ts     Pure reducer (single source of truth)
    WorkspaceProvider.tsx   Wires reducer to data channel + presence
  hooks/
    useBroadcastChannel.ts  Generic typed BroadcastChannel hook
    usePresence.ts          Heartbeats, peer registry, leader election
    useDerivedFinance.ts    useMemo wrappers for finance functions
    useUndoShortcut.ts      Ctrl/Cmd+Z → UNDO
    useUrlState.ts          URL query-string sync
  lib/
    finance/                emi, amortization, sensitivity, comparison,
                            prepayment, format — pure functions, no React
    sync/                   peers.ts (leader election), ids.ts
    export/                 csv.ts
  types/                    state, actions, sync, presence
tests/
  sanity-check.ts           Reducer + finance math + purity assertions
  live-sync-check.ts        Real BroadcastChannel sync (two simulated tabs)
```

---

## Edge Cases Handled

- Zero interest rate → straight-line principal split
- Prepayment > remaining balance → capped at balance, loan closes early
- Prepayment month > tenure → validated and rejected
- Multiple prepayments in same month → summed
- Sensitivity grid near bounds → values clamped and de-duplicated
- Concurrent edits → last-write-wins (documented trade-off, not a CRDT)

## Known Limitations

- **No persistence across reloads** — closing all tabs and reopening starts from defaults (by design)
- **Safari's BroadcastChannel on teardown** can be unreliable — the 2.6s heartbeat timeout is the real fallback
- **Leader hydration race** — narrow window where a new tab opening during leader transition falls back to defaults

## Deploying

Push to GitHub and import at [vercel.com/new](https://vercel.com/new) — no env variables needed, works with default Next.js settings. Also runs on Netlify, Cloudflare Pages, or any Node host.
