# Shared Memory (Main ↔ VPS) — Decisions + Implementation Plan

Date: 2026-02-25 (America/Toronto)

This doc captures the decisions we made about **OpenClaw memory** and a concrete implementation plan so we don’t lose the thread when the chat is compacted.

## Context: what problem we’re solving
We want the **trading ops VPS agent** and the **main agent** to share a small set of durable “truths” (risk rules, bot params, key operational decisions) without:
- bloating every prompt with giant markdown
- relying on embeddings / vector DB for basic facts
- leaking personal/non-trading context into trading ops

We also discussed “DB-backed memory” as a longer-term improvement.

## Key decisions (locked in)

### 1) Shared memory location and sync model
- **Source of truth at runtime:** the **VPS**.
- **Primary mechanism (A):** the bot reads **local files on the VPS** every cycle.
- **Optional update mechanism (B):** use **Nextcloud/WebDAV** only to *push updates*, never as a hard dependency for the trading loop.

Rationale: trading ops needs high reliability and low latency. Networked memory (WebDAV) adds failure modes. Local reads are simple and robust.

### 2) Scope: what is shared vs not shared
**Shared (trading ops only):**
- risk policy + hard limits
- strategy parameters
- “paper-first” / go-live gate
- operational invariants (cadence, kill-switch, leverage cap, max deployed, etc.)

**Not shared:**
- personal life context
- raw chat logs
- secrets (private keys, API keys, cookies/tokens)

### 3) Memory format (v1)
We will start with a small, human-auditable “shared folder” of files:

Folder name (logical): `OpenClawShared/`

Files:
- `RISK_POLICY.json` — machine-readable risk invariants
- `PARAMS.json` — machine-readable strategy knobs
- `DECISIONS.md` — dated bullets of why we changed something
- `FACTS.md` — small set of stable facts relevant to trading ops

### 4) DB-backed memory (future)
A DB can help, but we will not start there.

**Where DB-backed memory helps:**
- structured factual lookups (entity/key/value)
- auditability (source/timestamp)
- scale without prompt bloat

**Where it hurts early:**
- extra complexity and new failure modes
- embeddings add latency + cost + dependency (if used)

If we do a DB later, preferred approach is **SQLite + FTS5** for structured facts, with optional vector search only if we truly need fuzzy recall.

## Trading-specific constraints referenced in the memory discussions
These are trading ops invariants that should live in `RISK_POLICY.json`:
- **Paper-first** by default; go-live only when Boiler explicitly approves.
- **No LLM dependency in the execution loop.** LLM can be observer/summarizer or bounded modifier.
- **Max leverage:** 2x (starter).
- **Max deployed:** 75% equity (keep 25% reserve).
- **Daily kill-switch:** pause trading for the day at -10% equity.
- **Per-trade risk target:** 1% equity **with USD floor/ceiling**.
  - We updated to: **min risk = $10**, **min collateral = $20** for small starting balances.

## Implementation plan (step-by-step)

### Phase 0 — Create the shared folder on the VPS (A)
1) Pick a local path that is simple and permission-safe:
   - Recommended: `/opt/openclaw-shared/`
2) Ensure perms:
   - owned by `boilerrat:boilerrat`
   - bot process: read-only access

### Phase 1 — Define v1 file schemas
Create these files on the VPS under `/opt/openclaw-shared/`:

#### `RISK_POLICY.json`
Minimum fields:
- `version`
- `updated_at`
- `paper_first` (bool)
- `max_leverage`
- `max_deployed_pct`
- `daily_kill_switch_pct`
- `risk_pct`
- `min_risk_usd`
- `max_risk_usd`
- `min_collateral_usd`
- `notes`

#### `PARAMS.json`
- strategy choice + knobs (trend windows, MR z-score thresholds, ATR multipliers, slippage model params)

#### `DECISIONS.md`
- append-only bullets; include date + reason

#### `FACTS.md`
- tiny “always true” facts for ops

### Phase 2 — Wire the bot to read shared configs
In `boilermolt/avantis-paper-bot`:
- add `APB_SHARED_DIR=/opt/openclaw-shared` support
- load shared `RISK_POLICY.json` and `PARAMS.json`
- validate JSON; if invalid, keep last known good config

### Phase 3 — Optional updater (B)
Choose one bridge from Nextcloud → VPS local:

**Option B1: Nextcloud sync client on the VPS**
- Sync `OpenClawShared/` to `/opt/openclaw-shared/`

**Option B2: WebDAV pull script (recommended for deterministic ops)**
- A small script that:
  1) downloads `RISK_POLICY.json` + `PARAMS.json` to temp files
  2) validates JSON
  3) atomic rename into `/opt/openclaw-shared/`
- run every 5 minutes via cron

### Phase 4 — Governance and safety
- Never store secrets in these files.
- Require explicit manual confirmation for any change that increases risk (e.g., leverage, risk_pct, disabling kill-switch).
- Keep `DECISIONS.md` updated whenever risk/params change.

## Open questions / TODO
- Confirm where Nextcloud files live on the VPS (path) and/or decide WebDAV pull vs sync client.
- Decide whether the main agent should have read-only or read/write ability to the shared folder.

---

If you want this turned into code immediately:
- I can generate the initial `/opt/openclaw-shared/` file set + add a loader module in `avantis-paper-bot` that merges shared config over defaults, then push to GitHub.
