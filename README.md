# Spin Check 🕵️

**Do a company's words match its numbers?**

Spin Check is an AI system that detects when a company's management *narrative* (how confident and upbeat they sound on earnings calls) diverges from its actual *financial fundamentals* — and tests whether that gap, the "spin," predicts a future disappointment.

---

## The problem it solves

Every earnings call sounds optimistic. Executives are professionally confident, even when the business is quietly slipping. Analysts at hedge funds and asset managers spend enormous effort — and get paid a lot — to read between the lines and separate real signal from management spin.

**Spin Check automates that judgment at scale:** it reads the transcript, scores the narrative, compares it against the hard financials, and flags companies where the story and the numbers don't line up.

## How it works

```
Earnings call transcript ──► LLM scores the NARRATIVE (confidence, optimism, hedging)
                                              │
Financial filings ──────────► the actual NUMBERS (revenue, margin, guidance trends)
                                              │
                                              ▼
                          DIVERGENCE ("spin") signal  ──►  model predicts
                                                            probability of a
                                                            fundamentals surprise
                                                            next quarter
```

**What it predicts:** a *probability* of a next-quarter fundamentals disappointment (revenue/margin/guidance) — not a stock price. Predicting the business is more honest and more learnable than predicting the market's reaction, and the whole system is validated with a **leakage-aware backtest** (no peeking at the future).

## Why this is different

Most finance-AI portfolio projects are one of two saturated things: a "chat with a 10-K" RAG chatbot, or FinBERT sentiment → price. Spin Check's differentiator is the **narrative-vs-fundamentals divergence signal** combined with **honest, leakage-controlled evaluation** — the parts that signal real judgment rather than tutorial-following.

## Tech stack

- **Python**
- **RAG** — `sentence-transformers` (embeddings) + `ChromaDB` (vector store)
- **LLM** — structured extraction of narrative features
- **PyTorch** — the prediction model
- Financial data APIs (TBD)

## Roadmap

- [ ] **Phase 1 — RAG infrastructure:** pull an earnings-call transcript, chunk it, embed it, store it, and retrieve relevant passages *(in progress)*
- [ ] **Phase 2 — Narrative scoring:** use an LLM to turn call transcripts into structured scores (confidence / optimism / hedging)
- [ ] **Phase 3 — Fundamentals:** pull the actual financials (revenue, margin, guidance)
- [ ] **Phase 4 — Divergence signal:** combine narrative + numbers into the "spin" score
- [ ] **Phase 5 — Prediction model:** train a PyTorch model: spin → probability of fundamentals surprise
- [ ] **Phase 6 — Honest backtest:** leakage-aware, out-of-sample evaluation
- [ ] **Phase 7 — Package it:** results write-up + a small dashboard

## Concepts learned along the way

*(updated as the project grows)*

- Embeddings & vector search
- Retrieval-Augmented Generation (RAG)
- LLM structured extraction
- Neural networks (PyTorch)
- Leakage-aware backtesting

## Status

🚧 Early development — Phase 1. This is a learning-by-building project; the README grows with it.
