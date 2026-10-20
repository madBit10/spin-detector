# Spin Check: Interview Notes

Read this before an interview to get back on par with the project. Plain language first, details after.
Companion docs: `DESIGN.md` (the locked decisions, in full), `FLOW.md` (how the code connects), `CHANGES_MADE.md` (what changed and why, line by line).

*Last updated: 2026-10-20 · Status: design phase. Decisions 1, 2a, 2b locked; 3 notebooks of evidence; no pipeline code yet.*

---

## 1. The 30-second pitch

> "Every earnings call sounds optimistic. Spin Check uses an LLM to score **how management talks** (confidence, hedging, evasiveness), compares that with **what the numbers actually did**, and turns the gap, the 'spin', into a **probability that the company disappoints next quarter**. It's built to catch the case where executives sound upbeat while the business is quietly slipping. It predicts the *business*, not the stock price, and it's tested with a backtest that can't peek at the future."

**Why not predict stock prices?** Prices are noisy and react to everything; a portfolio project claiming to beat the market gets eye-rolls. Fundamentals (sales growth, margins) are slower, more learnable, and it's honest framing: *risk-flagging, not trading*.

**What makes it different from the usual finance-AI projects:** most are "chat with a 10-K" RAG bots or "FinBERT sentiment → price". Spin Check's edge is the **words-vs-numbers divergence signal** plus **leakage-aware evaluation**.

---

## 2. How it works, in one picture

```
Earnings call transcript ──► LLM scores the WORDS (confidence, hedging, evasion)
                                         │
SEC financial filings ─────► the NUMBERS (sales growth, gross margin, inventory)
                                         │
                                         ▼
                     SPIN = words score − numbers score   (feature)
                                         │
                                         ▼
               model → P(fundamentals disappointment next quarter)
```

The numbers are used **twice**:
- **this quarter's numbers** → part of the input (the spin feature)
- **next quarter's numbers** → the answer key (the label: did growth slow / margin drop / guidance get cut?)

**Planned stack:** Python, sentence-transformers + vector store (RAG), an LLM for structured scoring, PyTorch, Postgres (+ pgvector), FastAPI from day 1, Streamlit/dashboard, deployed on Render.

**Roadmap:** 0 data sourcing → 1 RAG → 2 LLM narrative scoring → 3 fundamentals → 4 divergence score → 5 model → 6 honest backtest → 7 package + dashboard. The API grows with every phase.

---

## 3. Decisions so far, and *why* (the part interviewers probe)

### Decision 1: Scope ✅
| What | Choice | Why |
|---|---|---|
| Sector | **Retail only**, store-based: discretionary (Gap, Macy's) **+ staples** (Walmart, Costco) | One sector keeps "confident" meaning the same thing across companies; retail has clear metrics and lots of real disappointments |
| Size | **50 companies**: ~25 S&P 500 + ~25 mid/small caps | Small enough to run cheaply; the pipeline doesn't care about the size (it's a config list), so 100 is just a re-run |
| Selection | Index membership **as of 2015-01-01**, not today | Picking from today's list drops every company that died → **survivorship bias**. Starting in 2015 keeps Macy's, Kohl's, Bed Bath & Beyond, Big Lots in |
| Window | Calls **2015–2025**; 2015 = warm-up history | YoY comparisons need a year of history |
| COVID | **Included**, flagged | Excluding it would hide the hard part. But YoY comparisons break (2021 vs closed-store 2020 looks like +80%) → `covid_period` flag, vs-2019 comparisons, backtest with and without |
| Bankruptcies | **Kept**; final call before bankruptcy = positive label; acquisition = dropped | A bankruptcy is the ultimate disappointment; a buyout isn't a fundamentals miss |
| Excluded | Online-only (no same-store sales), foreign filers (no quarterly XBRL), < 12 quarters of transcripts **unless delisted** | The 12-quarter rule would otherwise leak the future (drop companies that died early) |

**Numbers to remember:** ~2,000 usable rows, ~400 positives → enough for logistic regression/GBM, thin for a neural net (and saying so is fine).

### Decision 2a: Transcripts ✅ (free hybrid)
- **Option A**: free Hugging Face dataset (`kurry/sp500_earnings_transcripts`, ~33k transcripts). **Tested in notebook 01:** it covers companies *only while they're in the S&P 500*. Bed Bath & Beyond stops in 2018, Macy's/Kohl's in 2021, Gap and JCPenney are missing, and the dataset ends May 2025. It's missing exactly the decline years the model needs most.
- **Alpha Vantage** (free tier, ~25 requests/day) **fills the gaps. Tested in notebook 02:** Bed Bath & Beyond 2021, Gap 2019, small-cap Big 5 2017, all split by speaker with Q&A.
- **Rule:** Option A wins when both have a call (no quota spent); Alpha Vantage only for what's missing. ~1,000 requests ≈ 6 weeks of background downloading.
- **Rejected:** API Ninjas ($39 tier = only 5 years of history; $99 tier = not needed once the free option passed); press releases as transcripts (no Q&A, see confounding below).

### Decision 2b: Financials ✅ (SEC EDGAR)
- **Source: SEC companyfacts (XBRL).** Free, official, one request per company = full history, **every number has its own `filed` date**, filings are never deleted (bankrupt companies included).
- **Raw:** net sales, cost of goods sold, inventory. **Derived:** YoY sales growth, gross margin, **inventory–sales spread** (inventory growing faster than sales = unsold stock → markdowns → margin crash next quarter; the classic spin setup).
- **YoY, never quarter-over-quarter**: retail is seasonal.
- **Tag lists per metric**, first-reported across tags, tagged Q4 else derived, missing = mark + warn, join to calls = first call ≤60 days after the quarter's `end` date.

---

## 4. The talking points: problems found and how they're handled

These are the stories to tell. Each one shows judgment, not tutorial-following.

**① Survivorship bias.** Building the company list from today's index silently removes every retailer that went bankrupt, and those are the best examples of disappointments. Fix: pick from the 2015 index.

**② The LLM already knows the future.** The LLM was trained on data from *after* these calls, so it may "know" Macy's had a bad 2019. Fix: **mask company names, tickers, product names and dates** before scoring, and compare masked vs unmasked scores. *(Best interview talking point.)*

**③ Source confounding.** If one data source supplies mostly the decliners (positives), the model can learn "this source's writing style = surprise" instead of spin. Fixes: second source must be full transcripts with Q&A (not press releases), one shared schema, a `source` column that's **never a feature**, backtest per source, and score overlapping calls from both sources to check they agree.

**④ COVID base effects.** 2022 vs the 2021 stimulus boom makes almost every retailer "disappoint". That's the calendar, not spin. Fix: `covid_period` flag + vs-2019 comparisons + results with and without COVID.

**⑤ Tickers lie, IDs don't.** Gap's 2019 call is empty under its 2019 ticker `GPS` but exists under today's `GAP`. Bed Bath & Beyond isn't in the SEC's ticker file at all (current tickers only). Fix: identify every company by its **SEC CIK** (never changes), keep a ticker-history map, and check the company name appears in the operator's opening line.

**⑥ Restatements and "first-reported".** The same quarter's number appears in several filings, sometimes revised. The model may only see **what was public at the time**: the earliest `filed` value. Macy's Q1 2018 was first filed under an **old** tag; searching the new tag alone would make it look public 10 months late → compute first-reported **across tags**.

**⑦ Same name, different number.** After 2022, Macy's `Revenues` tag is ~$160–200M/quarter higher than its net sales tag (it includes other revenue, likely credit-card income). Mixing them would create fake growth jumps that fire the label. Fix: one definition per metric, ordered tag lists, a backup tag flagged when used.

**⑧ Fiscal calendars.** Retail years end in late Jan/early Feb; Walmart's "Q4 FY2025" ends Jan 31, 2025. Both data sources use **fiscal** quarters (verified: Macy's Alpha Vantage "2018Q1" = the May 2018 call), so everything joins on fiscal period + date, never calendar quarter.

**⑨ The numbers become public later than the call.** Macy's FY2018 10-K was filed 5 weeks after its Q4 call. Which date counts as "public" is an open leakage question (Decision 4). Safe default: the filing date.

**⑩ Secrets in URLs.** Alpha Vantage puts the API key in the URL, and `raise_for_status()` prints the URL on errors → it would have leaked the key into the notebook (and GitHub). Fix: sanitised error messages. Also: Alpha Vantage returns **HTTP 200 for errors**, so the response body is checked too.

---

## 5. Likely interview questions, with short answers

**"Why retail?"** Interpretable fundamentals (same-store sales, margins, inventory), plenty of real disappointments, and analysts already read retail calls for spin.

**"How do you know it isn't just learning from the future?"** Four guards: point-in-time data (first-reported, `filed` dates), walk-forward time splits (train on past years, test on the next), masking names/dates before LLM scoring, and an automated test asserting every feature was public before the prediction time.

**"How many rows? Will a neural net work?"** ~2,000 rows, ~400 positives. Probably not better than logistic regression, and the honest result is the ablation: *does adding the narrative features beat fundamentals alone?*

**"Why not use the vendor's sentiment score?"** Third-party scores may be produced later (hindsight leakage), and narrative scoring *is* the project.

**"Why FastAPI from day 1?"** The API grows with each phase, so it's tested and deployed continuously instead of bolted on. It's a thin, read-only layer: heavy work (embedding, LLM scoring, training) runs as batch jobs, and no public endpoint triggers paid LLM calls.

**"What would you do with more money?"** Point-in-time Compustat + consensus estimates (to define surprise vs analyst expectations), and a single transcript source for every call.

**"What's the biggest weakness?"** Label definition without analyst consensus (surprise is measured against the company's own trend), and a small sample. Both are stated up front.

---

## 6. Concepts learned (one line each)

- **Survivorship bias**: a dataset built from survivors overstates how well things go.
- **Point-in-time data**: each value stored with the date it became public, so the past can be replayed honestly.
- **Leakage**: the model seeing information it couldn't have had at prediction time.
- **Confounding**: a hidden variable (e.g. data source) tied to both input and label.
- **Base effects**: a growth rate distorted by an abnormal comparison period.
- **XBRL / tags**: machine-readable labels on numbers in SEC filings; they change over time.
- **Duration vs instant facts**: sales cover a period (`start`→`end`); inventory is a snapshot (`end`).
- **CIK**: the SEC's permanent company ID.
- **Jaccard similarity**: share of distinct words two texts have in common (used to show two sources had the same call).
- **Cache-first ingestion**: save every API response to disk, never request it twice (vital under rate limits).

---

## 7. Where things stand and what's next

**Evidence notebooks:**
| Notebook | Question | Answer |
|---|---|---|
| `01_explore_transcripts` | Is the free Hugging Face dataset enough? | No: misses decliners after index exit, Gap, small caps |
| `02_test_alphavantage` | Does Alpha Vantage fill the gaps? | Yes, free, fiscal quarters, history under current tickers |
| `03_explore_sec_financials` | Is SEC companyfacts enough for fundamentals? | Yes, with tag lists, first-reported across tags, computed gross margin |

**Design progress:**
| # | Decision | Status |
|---|---|---|
| 1 | Scope & universe | ✅ |
| 2a / 2b | Transcripts / financials | ✅ / ✅ |
| 2c | Same-store sales + guidance (not in XBRL → press releases / transcripts) | ⬜ **next** |
| 2d | Index membership 2015 + delisting reasons | ⬜ |
| 3–12 | Label, leakage, storage, RAG, LLM scoring, features, model, backtest, pipeline, API | ⬜ |

**Open questions carried forward:** which date counts as "public" (call vs filing) → Decision 4 · Chroma vs pgvector → Decision 5 · label thresholds X/Y → Decision 3.

**Housekeeping still owed:** rebuild `env/` on Python 3.11+ (3.9 is end-of-life; the SSL warning comes from it) · commit the notebooks + docs (`data/` and `.env` are git-ignored).
