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

# RUNBOOK (handoff-proof)

This section is designed so a new agent can implement the plan with minimal additional context.

## Target outcome
- VPS has local shared config at: `/opt/openclaw-shared/`
- Trading bot reads shared config using: `APB_SHARED_DIR=/opt/openclaw-shared`
- Optional: a WebDAV pull job updates the VPS local files safely.

## 0) Safety rules (non-negotiable)
- **Never** store secrets (private keys, API keys, cookies, OAuth tokens) in shared files.
- Treat these files as **configuration**, not an append-only diary.
- Changes that increase risk require explicit human confirmation.

## 1) Create the VPS folder (one-time)
Run on the VPS:

```bash
sudo mkdir -p /opt/openclaw-shared
sudo chown -R boilerrat:boilerrat /opt/openclaw-shared
sudo chmod 750 /opt/openclaw-shared
```

## 2) Create initial shared files (templates)

### `/opt/openclaw-shared/RISK_POLICY.json`
```json
{
  "version": 1,
  "updated_at": "2026-02-25",
  "paper_first": true,
  "execution_loop_requires_llm": false,

  "max_leverage": 2.0,
  "max_deployed_pct": 0.75,
  "daily_kill_switch_pct": -0.10,

  "risk_pct": 0.01,
  "min_risk_usd": 10.0,
  "max_risk_usd": 50.0,
  "min_collateral_usd": 20.0,

  "notes": "v1 shared trading ops policy (paper-first; bounded risk)"
}
```

### `/opt/openclaw-shared/PARAMS.json`
```json
{
  "version": 1,
  "updated_at": "2026-02-25",
  "pair": "ETH/USD",
  "tf_min": 15,

  "strategy": {
    "mode": "combo",
    "trend_fast": 20,
    "trend_slow": 50,
    "mr_window": 50,
    "mr_entry_z": 1.5,
    "atr_n": 14,
    "atr_mult": 2.0,
    "tp_rr": 2.0
  },

  "paper": {
    "slippage": {
      "model": "vol_based",
      "base_bps": 2.0,
      "k_atr": 15.0
    }
  }
}
```

### `/opt/openclaw-shared/DECISIONS.md`
```md
# Shared Decisions (Trading Ops)

- 2026-02-25: Shared memory design chosen: VPS local files as runtime source of truth; optional Nextcloud/WebDAV for updates.
- 2026-02-25: Risk floor updated for small accounts: min_risk_usd=10, min_collateral_usd=20.
```

### `/opt/openclaw-shared/FACTS.md`
```md
# Shared Facts (Trading Ops)

- Repo: https://github.com/boilermolt/avantis-paper-bot
- Trading mode: paper-first; go-live requires explicit approval.
```

## 3) Optional: WebDAV pull updater (Nextcloud → VPS)

### a) Create a read-only Nextcloud app password
- Create an app password in Nextcloud.
- Store it as an environment variable on the VPS (do **not** commit it):
  - `NEXTCLOUD_USER`
  - `NEXTCLOUD_APP_PASSWORD`
  - `NEXTCLOUD_WEBDAV_BASE` (ends with `/OpenClawShared/`)

### b) Example pull script (atomic + validated)
Create `/opt/openclaw-shared/webdav_pull.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${NEXTCLOUD_WEBDAV_BASE:?}"
: "${NEXTCLOUD_USER:?}"
: "${NEXTCLOUD_APP_PASSWORD:?}"

DST="/opt/openclaw-shared"
TMP="${DST}/.tmp"
mkdir -p "$TMP"

fetch() {
  local name="$1"
  curl -fsS -u "${NEXTCLOUD_USER}:${NEXTCLOUD_APP_PASSWORD}" \
    "${NEXTCLOUD_WEBDAV_BASE}${name}" -o "${TMP}/${name}"
}

validate_json() {
  local file="$1"
  node -e "JSON.parse(require('fs').readFileSync('${file}','utf8'));" >/dev/null
}

fetch "RISK_POLICY.json"
fetch "PARAMS.json"

validate_json "${TMP}/RISK_POLICY.json"
validate_json "${TMP}/PARAMS.json"

mv "${TMP}/RISK_POLICY.json" "${DST}/RISK_POLICY.json"
mv "${TMP}/PARAMS.json" "${DST}/PARAMS.json"

# Optional pull of markdown (non-critical)
for f in DECISIONS.md FACTS.md; do
  if curl -fsS -u "${NEXTCLOUD_USER}:${NEXTCLOUD_APP_PASSWORD}" "${NEXTCLOUD_WEBDAV_BASE}${f}" -o "${TMP}/${f}"; then
    mv "${TMP}/${f}" "${DST}/${f}"
  fi
done
```

Then:
```bash
sudo chmod +x /opt/openclaw-shared/webdav_pull.sh
```

### c) Cron (every 5 minutes)
```bash
crontab -e
*/5 * * * * /opt/openclaw-shared/webdav_pull.sh >/tmp/webdav_pull.log 2>&1
```

## 4) Wire into the trading bot
In `boilermolt/avantis-paper-bot`, implement:
- `APB_SHARED_DIR=/opt/openclaw-shared`
- load/merge `RISK_POLICY.json` + `PARAMS.json`
- if JSON invalid/unavailable: continue using last-known-good config

---

If you want this turned into code immediately:
- Generate the initial `/opt/openclaw-shared/` file set on the VPS and add a loader module in `avantis-paper-bot` that merges shared config over defaults.
