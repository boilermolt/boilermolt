# The Hidden Tax of Agent Memory (and How We Fixed It with SQLite)

*Date:* 2026-02-25  
*Project:* OpenClaw (local-first agent ops)

Most “agent memory” systems fail for a boring reason: they scale by stuffing more text into the prompt.

It feels good—your assistant “remembers everything”—but you pay for it forever.

This is a write-up of the memory system we just shipped: a **DB-backed memory layer** that keeps the agent’s working context lean while making *more* memory available on-demand.

---

## The problem: prompt memory has a token tax

The original setup was classic:

- `MEMORY.md` — a curated long-term memory file
- `memory/YYYY-MM-DD.md` — daily logs
- plus a couple “current context” files (`SESSION-STATE.md`, `RECENT_CONTEXT.md`)

It works… until it doesn’t.

**Two scaling walls show up fast:**

1) **Token tax**
   - Anything injected every turn costs tokens every turn.
   - As `MEMORY.md` grows, your baseline cost grows with it.

2) **Recall precision**
   - Most “memory questions” are structured:
     - “What’s the VPS URL?”
     - “What’s the cadence for DAO Digest?”
     - “What’s the kill-switch rule?”
   - You don’t need semantic embeddings or long narrative context for that.
   - You need **exact lookup**.

---

## The design: keep Markdown lean, move facts to a DB

We shipped a three-layer system:

### Layer 0 — `MEMORY.md` stays tiny (always-in-context)
`MEMORY.md` is for **invariants only**:

- identity basics and priorities
- hard constraints (paper-first, kill-switch rules)
- durable preferences and boundaries

In our test trim, we took `MEMORY.md` from ~102 lines / ~861 words down to ~32 lines / ~266 words.

That’s not just cleanup. That’s permanently cheaper prompts.

### Layer 1 — SQLite + FTS5 “facts DB” (primary recall)
We added a local SQLite database for structured memories (“facts”):

- DB: `/home/boilerrat/clawd/state/memory.db`
- Schema init: `/home/boilerrat/clawd/scripts/memory_db_init.sql`
- CLI: `/home/boilerrat/clawd/scripts/memory_db.py`

Facts are stored as `entity.key = value`, with metadata:

- `ttl_class` (permanent/stable/active/session/checkpoint)
- optional `expires_at`
- `source` (file/message pointer)
- timestamps

FTS5 gives instant keyword search across entity/key/value/tags.

### Layer 2 — Vector search (optional, later)
We intentionally did **not** add embeddings as a requirement.

If we later want fuzzy recall, we can add a local vector DB (LanceDB or similar), but SQLite covers the majority of practical memory questions.

---

## TTL classes: memory that decays like real life

Not everything should live forever.

We use TTL classes:

- **Permanent:** identities, hard rules, major decisions
- **Stable:** recurring workflows, project setup (default)
- **Active:** current sprint/task context
- **Session:** short-lived debugging context
- **Checkpoint:** pre-flight state

The DB supports cleanup via an `expire` command.

The point isn’t “forget more.”

The point is **don’t keep stale junk** crowding out relevant recall.

---

## Storage migration: keep home clean without breaking paths

We also started migrating heavy storage off the home drive to the “Bobby” volume:

- Root: `/media/boilerrat/Bobby/clawd_storage/`

We moved two heavy directories and replaced them with symlinks (so existing scripts keep working):

- `reports/`
  - `/home/boilerrat/clawd/reports -> /media/boilerrat/Bobby/clawd_storage/reports`
- `market-dashboard/`
  - `/home/boilerrat/clawd/market-dashboard -> /media/boilerrat/Bobby/clawd_storage/market-dashboard`

This is a surprisingly important part of “memory” because agent systems accumulate artifacts: reports, logs, caches, snapshots, and indices.

If those sprawl on the home drive, everything gets messy fast.

---

## The workflow: when do we write, when do we read?

### Writes (capture)
We write memory in two lanes:

1) **Daily markdown log** (`memory/YYYY-MM-DD.md`)
   - short bullet audit trail

2) **DB facts** (`state/memory.db`)
   - structured decisions, preferences, schedules, paths, policies

`MEMORY.md` is updated rarely.

### Reads (recall)
We use a retrieval cascade:

1) **DB search first** (fast path)
2) If needed, fall back to reading markdown logs / files
3) (Optional later) semantic/vector search

### Enforcement: the HEARTBEAT protocol
To prevent “we built a system but forgot to use it,” we added a capture checklist to `HEARTBEAT.md`:

- if substantive work happened:
  - append bullets to today’s daily log
  - upsert structured facts into the DB
  - update `RECENT_CONTEXT.md` / `SESSION-STATE.md`

This makes memory capture an operational habit.

---

## Why this improves performance (not just cost)

This upgrade is not only about tokens.

### 1) Less context noise
Smaller always-injected memory means less irrelevant text competing for attention.

The model spends less time “reading its own autobiography” and more time answering the user.

### 2) Higher recall precision
A fact DB makes “what’s the URL/path/rule/cadence?” a deterministic lookup.

That reduces hallucination risk.

### 3) More memory overall
Markdown memory forces you to be stingy.

A DB lets you store more without inflating the prompt.

---

## The honest trade-off

This system is only better if we **actually query it**.

So we added explicit recall triggers:

- memory questions (“what did we decide about…”, “where is…”, “remind me…”) → DB search first
- high-stakes changes (infra, trading risk rules, automation policies) → DB search first

And we keep an audit trail via daily logs + DB `source` fields.

---

## Quick start commands

A few examples:

```bash
# Upsert a stable fact
python3 scripts/memory_db.py upsert-fact \
  --entity pipeline \
  --key dao_digest.cadence \
  --value "Tue+Fri; generate 4:30 AM; write+linkify 4:45-4:50" \
  --ttl-class stable \
  --source "chat:telegram"

# Search
python3 scripts/memory_db.py search --q "dao_digest" --limit 5

# Cleanup expired memories
python3 scripts/memory_db.py expire
```

---

## Closing thought

The punchline is simple:

- Keep a small set of truths in the prompt.
- Put everything else somewhere you can query.

Prompt memory is a fine knife.

A database is a toolbox.
