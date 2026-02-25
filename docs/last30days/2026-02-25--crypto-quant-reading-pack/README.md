# Reading Pack (links) + RAG-ready structure

Source: email from Chris Wylde (education-only reading list). This pack **does not include copyrighted PDFs**; it provides:
- A consistent folder structure for a local knowledge base (RAG)
- A curated link list + suggested metadata
- Prompt templates for: microstructure-aware execution review, bias linting, and overfitting guardrails

## Folder structure

```
reading_pack/
  quant/
  time_series/
  ml/
  optimization/
  reinforcement_learning/
  crypto_microstructure/
  arbitrage/
  momentum_pairs/
  guardrails/
  protocols/
  classic_lit/
  prompt_templates/
  metadata/
```

## Link catalog (start here)

### Core foundations
- Forecasting: Principles and Practice (FPP3): https://otexts.com/fpp3/
- ISLR (2e): https://trevorhastie.github.io/ISLR/
- Elements of Statistical Learning (ESL): https://hastie.su.domains/ElemStatLearn/
- Convex Optimization (Boyd & Vandenberghe): https://web.stanford.edu/~boyd/cvxbook/
- Reinforcement Learning (Sutton & Barto, 2e PDF): https://web.stanford.edu/class/psych209/Readings/SuttonBartoIPRLBook2ndEd.pdf

### Crypto markets
- Systematic review of crypto market microstructure (Springer, incl. PDF): https://link.springer.com/article/10.1007/s10479-023-05627-5
- Easley/O’Hara et al. 2024 (PDF): https://stoye.economics.cornell.edu/docs/Easley_ssrn-4814346.pdf
- Crypto Trading survey (arXiv): https://arxiv.org/abs/2003.11352

### Arbitrage & cross-venue
- Makarov & Schoar — Trading and Arbitrage in Cryptocurrency Markets (PDF): https://dspace.mit.edu/bitstream/handle/1721.1/130495/SSRN-id3171204.pdf?sequence=2

### Momentum / reversal
- Dobrynskaya (2023) momentum & reversal (PDF): https://www.pm-research.com/content/iijaltinv/early/2023/03/25/jai.2023.1.189.full.pdf
- “Pure Momentum” in Crypto (Fracassi & Kogan) (PDF): https://assets.ctfassets.net/c5bd0wqjc7v0/4RzmvaUG64ixNPXWuZGXbo/7115cc7bef963d2ff5abbacf879f5b1e/SSRN-id4138685.pdf

### Pairs / stat-arb
- Cointegration-based pairs trading (arXiv): https://arxiv.org/abs/2109.10662
- MDPI Statistical Arbitrage in Crypto (landing): https://www.mdpi.com/1911-8074/12/1/31

### Overfitting guardrails
- Probability of Backtest Overfitting (PBO) (PDF): https://www.davidhbailey.com/dhbpapers/backtest-prob.pdf
- Deflated Sharpe Ratio (DSR) (PDF): https://www.davidhbailey.com/dhbpapers/deflated-sharpe.pdf
- Harvey–Liu–Zhu factor zoo / multiple testing (PDF): https://people.duke.edu/~charvey/Research/Published_Papers/P118_and_the_cross.PDF
- Replicating Anomalies (NBER PDF): https://www.nber.org/system/files/working_papers/w23394/w23394.pdf

### Protocol literacy
- Bitcoin whitepaper (PDF): https://bitcoin.org/bitcoin.pdf
- Ethereum yellow paper (PDF): https://ethereum.github.io/yellowpaper/paper.pdf

### Classics / narrative priors (public domain)
- Mackay — Madness of Crowds: https://www.gutenberg.org/ebooks/636
- Le Bon — The Crowd: https://www.gutenberg.org/ebooks/445
- Sun Tzu — Art of War (Giles): https://www.gutenberg.org/files/132/132-h/132-h.htm
- Machiavelli — The Prince: https://www.gutenberg.org/files/1232/1232-h/1232-h.htm
- Adam Smith — Wealth of Nations: https://www.gutenberg.org/ebooks/3300
- J.S. Mill — On Liberty: https://www.gutenberg.org/ebooks/34901
- Marcus Aurelius — Meditations (Long): https://standardebooks.org/ebooks/marcus-aurelius/meditations/george-long
- Bacon — Novum Organum: https://www.gutenberg.org/ebooks/45988
- Descartes — Discourse on Method: https://www.gutenberg.org/ebooks/59
- Shakespeare — Complete Works: https://www.gutenberg.org/ebooks/100
- Tolstoy — War and Peace: https://www.gutenberg.org/files/2600/2600-h/2600-h.htm

## How to ingest (RAG notes)

- Chunk size: 800–1200 tokens; overlap: 100–150.
- Store metadata for each chunk: `title, author, year, domain, source_url, license, section, page_range`.
- Index **equations/tables** separately for quant PDFs; index **acts/scenes** for plays.
- Tie every backtest/edge proposal prompt to guardrail docs (PBO/DSR/HLZ) and require a significance/overfitting report.

See:
- `metadata/catalog.yaml`
- `prompt_templates/*.md`
