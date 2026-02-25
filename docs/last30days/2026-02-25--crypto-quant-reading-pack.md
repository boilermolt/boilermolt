# Crypto quant reading pack + RAG folder scaffold (2026-02-25)

This is a **link-based** reading pack (no copyrighted redistribution) plus a **RAG-ready folder structure** and **prompt templates** for:
- microstructure-aware execution review
- bias linting (Bacon) + Bayesian update sanity checks
- overfitting / multiple-testing guardrails (PBO/DSR/HLZ)

## Files
- Pack root: `./2026-02-25--crypto-quant-reading-pack/`
  - `README.md` (curated links + ingest notes)
  - `metadata/catalog.yaml`
  - `prompt_templates/`

## Suggested ingestion defaults
- Chunk: 800–1200 tokens, overlap 100–150
- Metadata: title/author/year/domain/source_url/license/section/page_range
- Special indexing:
  - quant PDFs: equations & tables separately
  - literature: act/scene granularity

## Primary guardrails to bake into your research loop
- PBO / CSCV (Bailey et al.)
- Deflated Sharpe Ratio (Bailey & López de Prado)
- Multiple-testing standards (Harvey–Liu–Zhu)
- Replication mindset (Replicating Anomalies)

