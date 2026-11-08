# Fork notes — meviang/billion-context-pi

This is a **fork of [ranxianglei/billion-context-pi](https://github.com/ranxianglei/billion-context-pi)** (upstream `master`, v0.1.83), maintained locally as the source copy that pi loads from `~/.pi/agent/extensions/billion-context-pi/`.

- **upstream** → `https://github.com/ranxianglei/billion-context-pi.git`
- **origin (this fork)** → `https://github.com/meviang/billion-context-pi.git`

Everything below is a **local change on top of upstream `master`**. Nothing here has been sent upstream; this fork is the record of what this machine runs.

---

## Changes vs upstream

### 1. `src/index.ts` — stale-anchor invalidation (fixes `max_tokens=1`)

**Problem.** pi-ai derives `max_tokens` from `estimateContextTokens`, which anchors on the
last assistant message's *real* usage. After an ACP compaction that anchor is **stale** — its
usage was measured on the *pre-compaction* full thread — and the injected summary messages
carry **no timestamp**, so pi-ai's `usageAppliesToPrefix` check never invalidates it. pi-ai
then estimates the context as the full thread (e.g. ~456K) and clamps
`max_tokens = min(cap, max(1, window − est − 4096))` to **1**; a local llama.cpp server ends
every reply after one token.

**Fix.** When a compaction is active (`turn.state.blocks.some((b) => b.active)`), unshift a
fresh-timestamp marker into the sent view (`rebuilt`):

```ts
const maxTs = rebuilt.reduce((m, msg) => Math.max(m, msg.timestamp ?? 0), 0);
rebuilt.unshift({
  role: "user",
  content: [{ type: "text", text: "[acp] compaction active — context re-estimated" }],
  timestamp: maxTs + 1,
} as AgentMessage);
```

The fresh timestamp invalidates every stale anchor, forcing pi-ai's accurate char-based
estimate of the *pruned* list (~50K → sane `max_tokens`). The timestamp is **deterministic**
(`maxExistingTs + 1`, not `Date.now()`) so the outbound view stays byte-identical across a
restart. Gated on an active block so ordinary turns keep the (accurate) real-usage anchor.

### 2. `src/index.ts` — host-floor-fallback debug event

Replaced the old `host-floor` debug event with a `host-floor-fallback` event that fires when
the host-reported context floor is stale and the extension falls back to the static floor
(see `src/floor-stale.ts` below).

### 3. `src/floor-stale.ts` — host-floor freshness guard

- **`hostFreshnessGate`** (new, exported) — returns a fallback to the static floor when the
  host-reported floor's `lastSeenAt` is older than the staleness threshold.
- **`isStaleHostFloor`** (new, exported predicate) — the staleness check used by the gate and
  by tests.
- **`hostFloorGate` → `hostFloorFreshnessGate`** (renamed) — the freshness gate, renamed to
  make the freshness semantics explicit.

Purpose: a stale host-reported context floor must not under-size the sent view; fall back to
the static floor when the host floor is old.

### 4. `package.json` — dependency pin

- `@earendil-works/pi-agent-core` (devDependency): `0.0.x` → `0.0.14`.

### 5. `tests/floor-stale.test.ts` — new tests

Covers `isStaleHostFloor` and the `hostFloorGate` → `hostFloorFreshnessGate` rename.

### 6. `tests/host-floor-fallback.test.ts` — new file

End-to-end test for the `host-floor-fallback` debug event: a fresh host floor is kept (no
event); a stale host floor falls back to the static floor and emits the event.

### 7. `README.md` — rewritten for local use

Trimmed from 303 → 68 lines: removed upstream marketing / paper / badges / community / OMP
sections; reframed as a local source copy of upstream `master`. Added the
**"Stale-anchor invalidation"** bullet under *How it works* (item 1 above).

---

## Verification

- `npm run typecheck` — clean.
- `npm test` — 1075 tests, 1072 pass, 0 fail, 3 skipped.
- Direct `estimateContextTokens` check on a realistic ~50K pruned list with a stale anchor:
  - no marker → est ≈ 165K → `max_tokens = 1`
  - with marker → est ≈ 50K → `max_tokens = 8192` (capped)

## Backup

Pre-edit copy (before the stale-anchor change) is at
`/tmp/acp-backup-20261108-131006/` (`original/` = pre-edit, `with-fix/` = post-edit, plus
tarballs).

---

> 🤖 This document is written and maintained by **Argos** (AI assistant). Last updated: 2026-11-08.
