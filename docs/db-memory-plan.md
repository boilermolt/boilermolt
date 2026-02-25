# OpenClaw Memory Upgrade (DB-backed) — Summary + Implementation Plan (handoff-proof)

Date: 2026-02-25 (America/Toronto)

This doc captures our plan to improve long-term memory in OpenClaw while keeping per-turn context lean. It’s written to be publication-friendly and handoff-proof.

## Why we’re doing this
Markdown memory (`MEMORY.md` + daily logs) works, but it hits two scaling walls:
1) **Token tax:** anything injected every turn costs tokens forever.
2) **Recall precision:** many memory questions are structured (“what is X’s Y?”) and do not need semantic search.

Goal: keep `MEMORY.md` small and durable, and move most recall into a local, queryable store.

## Design principles
- **Local-first.** Avoid adding cloud dependencies for memory retrieval.
- **Two lanes:**
  - **Facts/decisions (structured):** exact lookup
  - **Notes (unstructured):** for narrative recall
- **Traceability:** store a source pointer + timestamp for every memory.
- **Decay:** not everything should live forever; TTL/refresh-on-access prevents memory rot.
- **Safety:** never store secrets (private keys, API keys, cookies) in memory DB.

## Proposed architecture

### Layer 0 — Keep `MEMORY.md` lean (always-in-context)
Keep ~20–50 lines of high-value invariants:
- identity basics (name, timezone)
- hard constraints (paper-first, kill-switch rule)
- durable preferences (writing style, boundaries)
- key infrastructure facts that must never be missed

Everything else moves out.

### Layer 1 — SQLite + FTS5 “facts DB” (primary)
Use SQLite for structured memories:
- **Table:** `facts`
  - `id` (pk)
  - `entity` (e.g., "trading", "dao_digest", "vps")
  - `key` (e.g., "cadence", "risk.min_risk_usd")
  - `value` (string / json)
  - `tags` (optional)
  - `ttl_class` (permanent|stable|active|session|checkpoint)
  - `created_at`, `updated_at`
  - `expires_at` (nullable)
  - `source` (file path / URL / message id)
  - `confidence` (0–1, optional)

- **FTS index:** `facts_fts` over (`entity`, `key`, `value`, `tags`) for instant lookups.

Why: most queries are “structured lookups” and SQLite is fast, local, and dependency-free.

### Layer 2 — Optional vector store (later)
Only if needed for fuzzy recall. Do not make this required.

- Use a local vector DB (LanceDB or similar).
- Prefer local embeddings if available; otherwise accept cloud embeddings as an optional enhancement.

### Retrieval cascade (recommended)
1) Query SQLite FTS5 for exact/near-exact matches (fast path).
2) If results are weak or query is clearly semantic, do vector search (slow path).
3) Merge, dedupe, sort.
4) Inject top N snippets into context for the current turn.

## Memory lifecycle / decay policy
Not all memories should live forever.

Suggested TTL classes:
- **Permanent:** identities, hard rules, major decisions — never expires
- **Stable:** project setup, recurring workflows — 90 days; refresh on access
- **Active:** current sprint/task context — 14 days; refresh on access
- **Session:** debugging context — 24 hours
- **Checkpoint:** pre-flight state — 4 hours

Implement a periodic cleanup job that:
- prunes expired rows
- refreshes TTL on retrieval

## What gets written where

### Write to `MEMORY.md` (rare)
Only invariant, high-value facts.

### Write to DB (often)
- decisions (“we chose X because Y”)
- conventions (“always do X before deploy”)
- stable preferences
- operational parameters

### Write to daily markdown logs (raw)
Keep raw conversational notes in `memory/YYYY-MM-DD.md` for audit/tracing.

## Handoff-proof runbook

### 0) Choose DB location
On the main host:
- `/home/boilerrat/clawd/state/memory.db` (recommended)

### 1) Create schema (SQLite)
Create `scripts/memory_db_init.sql`:

```sql
PRAGMA journal_mode=WAL;

CREATE TABLE IF NOT EXISTS facts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  entity TEXT NOT NULL,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  tags TEXT,
  ttl_class TEXT NOT NULL DEFAULT 'stable',
  confidence REAL,
  source TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  expires_at TEXT
);

CREATE UNIQUE INDEX IF NOT EXISTS facts_entity_key ON facts(entity, key);
CREATE INDEX IF NOT EXISTS facts_expires_at ON facts(expires_at);

-- FTS5 virtual table
CREATE VIRTUAL TABLE IF NOT EXISTS facts_fts USING fts5(
  entity,
  key,
  value,
  tags,
  content='facts',
  content_rowid='id'
);

-- Triggers to keep FTS in sync
CREATE TRIGGER IF NOT EXISTS facts_ai AFTER INSERT ON facts BEGIN
  INSERT INTO facts_fts(rowid, entity, key, value, tags)
  VALUES (new.id, new.entity, new.key, new.value, new.tags);
END;

CREATE TRIGGER IF NOT EXISTS facts_ad AFTER DELETE ON facts BEGIN
  INSERT INTO facts_fts(facts_fts, rowid, entity, key, value, tags)
  VALUES('delete', old.id, old.entity, old.key, old.value, old.tags);
END;

CREATE TRIGGER IF NOT EXISTS facts_au AFTER UPDATE ON facts BEGIN
  INSERT INTO facts_fts(facts_fts, rowid, entity, key, value, tags)
  VALUES('delete', old.id, old.entity, old.key, old.value, old.tags);
  INSERT INTO facts_fts(rowid, entity, key, value, tags)
  VALUES (new.id, new.entity, new.key, new.value, new.tags);
END;
```

Run:
```bash
sqlite3 /home/boilerrat/clawd/state/memory.db < scripts/memory_db_init.sql
```

### 2) Add a tiny CLI wrapper
Create `scripts/memory_db.py` with commands:
- `upsert-fact --entity --key --value --ttl-class --source --tags`
- `search --q "..." --limit 5`
- `get --entity --key`
- `expire` (cleanup)

Use Python stdlib `sqlite3` only.

### 3) Update the agent workflow
- Keep `MEMORY.md` short.
- On “decision moments”, write:
  1) a brief daily log line (`memory/YYYY-MM-DD.md`)
  2) an upsert into `memory.db` for structured facts.

### 4) Retrieval hook (optional)
Add a pre-response step that:
- runs `memory_db.py search` with the user query
- injects the top hits into context under a “Retrieved memory” section.

(If OpenClaw doesn’t support a global hook cleanly, do it inside the agent turn logic where you already control prompt assembly.)

### 5) Token-control / trimming plan
- Monthly: review `MEMORY.md` and keep it under a hard token budget.
- Move stale details into DB facts or archived markdown.

## Publication notes (what’s interesting to share)
- The “token tax” of always-injected markdown memory is the hidden cost people underestimate.
- SQLite+FTS5 is an underrated baseline: it solves 80% of memory questions without embeddings.
- TTL + refresh-on-access prevents stale memory from polluting retrieval.

## Open questions
- Where to run this: main host only vs mirrored onto VPS trading ops.
- Whether to add a vector DB at all.
- Whether to auto-extract facts/decisions from chat logs (later; not required for v1).
