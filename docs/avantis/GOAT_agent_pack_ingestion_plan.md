# Avantis Trade Bot — GOAT Crypto Trading Agent Pack (Ingestion + Use Plan)

Date: 2026-02-25

## Where the pack lives
Local path (confirmed):
- `/home/boilerrat/clawd/GOAT_Crypto_Trading_Agent_Pack`

Note: earlier you referenced `/home/claude/...` but this machine only has `/home/boilerrat/`.

## What it is
A ready-to-ingest syllabus + prompt/checklist library intended to become the **knowledge base + operating doctrine** for the Avantis trading agent.

Top-level domains (per manifest):
- quant foundations
- crypto microstructure
- strategies baselines
- overfitting/guardrails
- protocols
- philosophy/crowd
- prompts/templates
- checklists/policies

## Decision
- This pack is the canonical “reading list + doctrine” for the Avantis agent.
- We will incorporate it in two ways:
  1) **RAG retrieval** for factual/technical questions and justification
  2) **Deterministic checklists** (pre-trade / deployment / incident) that gate execution

## Implementation (handoff-proof)

### Phase 0 — Freeze + mirror into a stable knowledge path
Create a stable path (so code doesn’t depend on ad-hoc locations):
- `/home/boilerrat/clawd/knowledge/GOAT_Crypto_Trading_Agent_Pack/`

Option A: symlink (fastest)
```bash
mkdir -p /home/boilerrat/clawd/knowledge
ln -sfn /home/boilerrat/clawd/GOAT_Crypto_Trading_Agent_Pack /home/boilerrat/clawd/knowledge/GOAT_Crypto_Trading_Agent_Pack
```

### Phase 1 — Build a baseline index (cheap, robust)
Use SQLite + FTS5 to index markdown files for keyword search.
- Index fields: `path`, `domain`, `title`, `content`
- Domain inferred from folder name (`01_quant_foundations`, etc.)

Deliverables:
- `state/goat_kb.db`
- `scripts/goat_kb_index.py` (build/update)
- `scripts/goat_kb_query.py` (search)

### Phase 2 — Optional semantic index (vector)
Only if we need fuzzy recall.
- Chunk docs: 800–1200 tokens, ~120 overlap (matches README guidance)
- Store metadata: `domain`, `doc_type`, `source_path`

### Phase 3 — Wire it into the Avantis agent/bot

#### 3.1 Retrieval rule
Before writing strategy changes or producing “trade rationale”, the agent must:
- query KB with: market + strategy type + risk topic
- cite 1–3 retrieved snippets (path + short quote)

#### 3.2 Checklist gates (non-optional)
Hard gates (from pack’s `08_checklists_policies/`):
- **Pre-trade checklist** must pass before any new position is allowed.
- **Overfitting check** must pass before enabling a new strategy configuration.
- **Execution review** must be run after trades (paper or live) to attribute slippage.

#### 3.3 Personality / doctrine
Convert the pack’s “prompts/templates” into a consistent agent voice:
- skeptical, execution-aware
- anti-overfitting
- explicit about uncertainty
- risk-first

### Phase 4 — Journal + publication workflow
Create an Obsidian note series:
- “GOAT Pack — Reading Notes” (one note per folder)
- “GOAT Pack — Rules we adopted” (bullets that become RISK_POLICY / PARAMS)

Each time we adopt a rule, we:
- add a bullet to `DECISIONS.md` (shared memory)
- add a fact into the memory DB (when we implement that plan)

## Immediate next step
Confirm: do you want me to **symlink** the pack into `/home/boilerrat/clawd/knowledge/` (safe, reversible), then I’ll implement the SQLite+FTS indexer + query CLI and wire the Avantis agent prompt to consult it.
