# billion-context-pi

In-process context-compression extension for pi — model-driven compaction
(`compress`/`decompress`) with **no proxy**. Local source copy of
[ranxianglei/billion-context-pi](https://github.com/ranxianglei/billion-context-pi)
master (v0.1.83), built with `npm ci && npm run build` (dist is self-contained:
`acp-kernel` + `typebox` bundled; only the pi packages are external and aliased
by pi's loader to its own install tree). Replaces the previous proxy-based
`billion-context` install (moved to `~/.pi/agent/deprecated/extensions/billion-context/`).

## Fork notes — changes vs upstream

This is a **fork of [ranxianglei/billion-context-pi](https://github.com/ranxianglei/billion-context-pi)**
(upstream `master`, v0.1.83).

- **upstream** → `https://github.com/ranxianglei/billion-context-pi.git`
- **origin (this fork)** → `https://github.com/meviang/billion-context-pi.git`

Everything below is a local change on top of upstream `master`; nothing has been sent
upstream.

| File | Change |
|---|---|
| `src/index.ts` | **Stale-anchor invalidation marker** (fixes `max_tokens=1`, see *How it works*) + host-floor-fallback debug event |
| `src/floor-stale.ts` | `hostFreshnessGate` / `isStaleHostFloor` (new); `hostFloorGate` → `hostFloorFreshnessGate` (renamed) |
| `package.json` | `@earendil-works/pi-agent-core` `0.0.x → 0.0.14` |
| `tests/floor-stale.test.ts` | tests for `isStaleHostFloor` + the rename |
| `tests/host-floor-fallback.test.ts` | new — host-floor-fallback event test |
| `README.md` | rewritten for local use (303 → 68 lines) + stale-anchor bullet |

**Verification:** `npm run typecheck` clean; `npm test` 1072/1075 pass (3 skipped).
Direct `estimateContextTokens` check on a ~50K pruned list with a stale anchor: no
marker → est ≈ 165K → `max_tokens=1`; with marker → est ≈ 50K → `max_tokens=8192`.

## What it provides

- `compress` — fold a message range (by `mNNNNN` refs) into a summary block
- `decompress` — restore a compressed block (tier-up, `full` for originals, `toFile`)
- `search_context` — keyword search over compressed block summaries
- `acp_status` — context usage overview + compressible ranges
- `acp_cache` — prompt-cache ledger: hit rate, miss attribution, per-fold economics
- `/acp` — TUI status panel (token breakdown, blocks, usage)
- `acp_delegate*` — **disabled** (see Configuration)

## How it works

- Intercepts pi's `context` event before each LLM call; 8-stage pipeline:
  assign refs → sync blocks → prune → filter → hide calls → recommend → nudge →
  emergency truncate.
- Tags every message with an invisible `<acp>` ref (`m00001`…) the model uses to
  target compression ranges.
- Cancels pi's built-in auto-compaction — this extension is the sole context manager.
- Compression state persists in a sidecar next to the session log:
  `<sessionFile>.jsonl.acp.json` (atomic writes; on a session without a sidecar it
  rebuilds state by replaying `compress` calls from the transcript).
- Refuses to run on OMP hosts; stands down automatically if a bili proxy is
  detected (`BILLION_CONTEXT_PROXY` env or `/bili/` baseURL) — no double compression.
- **Stale-anchor invalidation** (fixes `max_tokens=1` on local / small-context
  models): pi-ai derives `max_tokens` from `estimateContextTokens`, which anchors on
  the last assistant's real usage. After a compaction that anchor is *stale* — it
  measured the pre-compaction full thread — and the injected summaries carry no
  timestamp, so `usageAppliesToPrefix` never invalidates it. pi-ai then estimates the
  context as the full thread (e.g. ~456K) and clamps `max_tokens` to 1; a local
  llama.cpp server ends every reply after one token. When a compaction is active this
  extension unshifts a fresh-timestamp marker (`[acp] compaction active — context
  re-estimated`) into the sent view, forcing pi-ai's accurate char-based estimate of
  the pruned list (~50K → sane `max_tokens`). The marker's timestamp is deterministic
  (`maxExistingTs + 1`, **not** `Date.now()`) so the outbound view stays
  byte-identical across a restart. Gated on an active block so ordinary turns keep the
  (accurate) real-usage anchor.

## Configuration

- `~/.pi/acp.json`: `{ "delegate": { enabled: true, maxConcurrent: 1, agents: … } }`
  — `acp_delegate*` is the delegation path (the local `subagents` extension was
  removed 2026-11-16). `maxConcurrent: 1` caps background delegates; all 5 roles
  default to `llama-pc/qwen3.8-27b` + `thinkingLevel: medium`. Compression tools
  are unaffected.
- Logs: `~/.pi/acp.log` (rotates at 10 MB).

## Dependencies

- none at runtime — everything bundled in `dist/`. `node_modules` removed after
  build; recreate with `npm ci` if rebuilding from source.

## Files

```
billion-context-pi/
├── dist/            ← built extension (index.js + contract.js, pi entry point)
├── src/             ← upstream source (for rebuilds / patching)
├── package.json     ← pi.extensions → ./dist/index.js
└── README.md
```
