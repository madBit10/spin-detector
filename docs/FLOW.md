# Spin Check: How the Code Connects

Read this before reading the code. It answers three questions:
1. **What is connected to what?** (the system map)
2. **What happens when I run X?** (one flow per function or cell)
3. **Where does data live, and who writes or reads it?** (the data map)

Status markers: ✅ **exists now** · 🔜 **planned** (designed in `DESIGN.md`, not written yet).
Updated every time code changes. If a diagram and the code disagree, the code wins and this file gets fixed.

*Last updated: 2026-10-20 (notebooks 01–03 exist; no `src/` package yet).*

---

## 1. The system map: the whole app, end to end

```mermaid
flowchart LR
    subgraph SOURCES["External sources"]
        HF[("Hugging Face<br/>Option A transcripts")]
        AV[("Alpha Vantage API<br/>transcripts, ≤25/day")]
        SEC[("SEC EDGAR<br/>companyfacts XBRL")]
    end

    subgraph INGEST["Ingest (batch, CLI)"]
        I1["✅ explore A<br/>notebook 01"]
        I2["✅ fetch_transcript()<br/>notebook 02"]
        I3["✅ sec_get_json()<br/>notebook 03"]
    end

    subgraph DISK["data/ (git-ignored)"]
        R1[("✅ data/raw/alphavantage/*.json")]
        R2[("✅ data/raw/sec/*.json")]
        HFC[("✅ ~/.cache/huggingface<br/>Option A Arrow files")]
    end

    subgraph PIPE["Pipeline stages 🔜 (src/spincheck/)"]
        N["normalise → shared schema"]
        CH["chunk by speaker turn"]
        EM["embed → vectors"]
        SC["LLM narrative scoring"]
        FU["first-reported fundamentals"]
        LA["labels: next-quarter surprise"]
        FE["divergence features"]
        TR["train + walk-forward backtest"]
    end

    DB[("🔜 Postgres + pgvector")]
    API["🔜 FastAPI /v1/..."]
    UI["🔜 dashboard"]

    HF --> I1 --> HFC
    AV --> I2 --> R1
    SEC --> I3 --> R2
    HFC --> N
    R1 --> N
    R2 --> FU
    N --> CH --> EM --> DB
    CH --> SC --> DB
    FU --> DB
    FU --> LA --> DB
    DB --> FE --> TR --> DB
    DB --> API --> UI
```

**How to read it:** data only flows **left to right**. Batch stages (CLI, run by you) do all the heavy work and write to the database. The API only **reads** precomputed results (the "thin API" rule from the design). Nothing on the right ever calls a paid external API.

---

## 2. What exists now: the three notebooks

Notebooks are **exploration**, not the app. Each one answers a design question. The useful functions in them (`fetch_transcript`, `sec_get_json`, `facts_df`, `first_reported`) are first drafts of 🔜 `src/spincheck/` modules.

```mermaid
flowchart TB
    N1["notebook 01<br/>explore Option A"] -->|"finding: A misses decliners,<br/>Gap, small caps"| D2a["Decision 2a"]
    N2["notebook 02<br/>test Alpha Vantage"] -->|"finding: AV fills the gaps,<br/>fiscal quarters, current tickers"| D2a
    N3["notebook 03<br/>explore SEC financials"] -->|"finding: tag switches,<br/>first-reported rule"| D2b["Decision 2b"]
    D2a --> S1["🔜 src/spincheck/ingest/transcripts.py"]
    D2b --> S2["🔜 src/spincheck/ingest/fundamentals.py"]
```

---

## 3. Function flows: "what happens if I run this?"

### 3.1 `fetch_transcript(symbol, quarter)`: notebook 02 ✅

**Called by:** the "Run the test calls" loop. **Calls:** `requests.get` (only on a cache miss). **Writes:** `data/raw/alphavantage/{SYMBOL}_{QUARTER}.json`.

```mermaid
flowchart TD
    A["fetch_transcript('GAP', '2019Q2')"] --> B{"data/raw/alphavantage/<br/>GAP_2019Q2.json exists?"}
    B -->|yes| C["read file → return (data, from_cache=True)<br/>⚡ no request spent"]
    B -->|no| D["requests.get(BASE_URL, params incl. apikey, timeout=30)"]
    D -->|network error| E["raise RuntimeError('network error')<br/>🔒 URL with key NOT shown"]
    D --> F{"response.ok?"}
    F -->|no| G["raise RuntimeError('HTTP 5xx')<br/>🔒 not raise_for_status()"]
    F -->|yes| H{"JSON has Note /<br/>Information / Error Message?"}
    H -->|yes| I["raise RuntimeError(...)<br/>❗ NOT saved: cache must never hold an error"]
    H -->|no| J["write JSON to disk"] --> K["return (data, from_cache=False)"]
```

**The loop that calls it:**

```mermaid
sequenceDiagram
    participant Loop as run-calls cell
    participant F as fetch_transcript()
    participant Disk as data/raw/alphavantage/
    participant AV as Alpha Vantage
    loop for each (symbol, quarter) in TEST_CALLS
        Loop->>F: fetch_transcript(symbol, quarter)
        F->>Disk: cached?
        alt cache hit
            Disk-->>F: JSON
        else cache miss
            F->>AV: GET ?function=EARNINGS_CALL_TRANSCRIPT
            AV-->>F: JSON (HTTP 200 even on errors)
            F->>Disk: save (only if no error key)
        end
        F-->>Loop: (data, from_cache)
        Loop->>Loop: results[(symbol, quarter)] = data
        Note over Loop: sleep 15s only after a REAL request
    end
```

**Downstream:** `results` feeds `describe()` (prints keys + first segments), the summary-table cell, and the Macy's Jaccard overlap cell (which also loads Option A from the Hugging Face cache).

### 3.2 `sec_get_json(url, cache_name)`: notebook 03 ✅

Same cache-first shape as `fetch_transcript`, but with a **User-Agent header** (name + email from `.env`) and a 0.2 s pause. **Writes:** `data/raw/sec/company_tickers.json`, `data/raw/sec/companyfacts_{CIK10}.json`.

```mermaid
flowchart TD
    A["sec_get_json(url, cache_name)"] --> B{"data/raw/sec/cache_name exists?"}
    B -->|yes| C["return cached JSON"]
    B -->|no| D["requests.get(url, headers={User-Agent})<br/>sleep 0.2s"]
    D --> E{"ok?"}
    E -->|no| F["raise RuntimeError(url → HTTP code)<br/>(safe: no secret in SEC URLs)"]
    E -->|yes| G["save → return JSON"]
```

### 3.3 Notebook 03, end to end: from ticker to first-reported quarterly revenue ✅

```mermaid
flowchart TD
    T["company_tickers.json<br/>via sec_get_json"] -->|"ticker 'M'"| CIK["CIK 0000794367<br/>(⚠️ current tickers only: BBBY missing)"]
    CIK --> CF["companyfacts_0000794367.json<br/>via sec_get_json: 1 request, full history"]
    CF --> UG["us_gaap = facts['us-gaap']<br/>539 tags"]
    UG --> TO["tag_overview(keywords)<br/>which tags exist + date ranges"]
    UG --> FD["facts_df(tag)<br/>one tag → DataFrame<br/>+ duration_days"]
    FD --> RV["revenue = concat(facts_df(t) for t in REVENUE_TAGS)"]
    RV --> QF["quarters = duration 80–100 days<br/>(drops year-to-date + annual)"]
    QF --> Q1["fiscal Q1 2018 deep dive<br/>all filings of one quarter"]
    RV --> FR["first_reported(df)<br/>earliest 'filed' per (start, end)"]
    FR --> Q4["annual vs Q1–Q4 check"]
```

**Key idea:** `facts_df` produces *every* report of a number; `first_reported` collapses those to **what was public first**. That's the leakage rule (Decision 4) in one function.

---

## 4. The data map: who writes and who reads

| Location | Written by | Read by | In git? |
|---|---|---|---|
| `.env` | you (by hand) | `load_dotenv` in notebooks 02, 03 | ❌ never |
| `data/raw/alphavantage/*.json` | `fetch_transcript()` | notebook 02 cells; 🔜 normalise stage | ❌ |
| `data/raw/sec/*.json` | `sec_get_json()` | notebook 03 cells; 🔜 fundamentals stage | ❌ |
| `~/.cache/huggingface/` | `load_dataset()` (notebooks 01, 02) | same | ❌ (outside the repo) |
| `docs/DESIGN.md` | design reviews | you, interviewers | ✅ |
| `docs/FLOW.md` | updated with every code change | you, before reading code | ✅ |
| `docs/NOTES.md` | updated at the end of each session | you, before interviews | ✅ |

---

## 5. The planned app: what one API request will trigger 🔜

Shown so you can see where today's functions end up. Nothing here exists yet.

```mermaid
sequenceDiagram
    participant U as Browser / curl
    participant R as FastAPI router<br/>api/routers/companies.py
    participant S as service layer
    participant DB as Postgres
    U->>R: GET /v1/companies/M/prediction?quarter=2018Q1
    R->>S: get_prediction("M", 2018, 1)
    S->>DB: SELECT ... FROM predictions JOIN companies (by CIK)
    DB-->>S: row (precomputed by the batch pipeline)
    S-->>R: Prediction model (Pydantic)
    R-->>U: 200 JSON {probability, top evidence quotes}
    Note over R,DB: No LLM call, no external API: the request only READS
```

The same **router → service → repository → database** layering as your webhook service.

---

## 6. Rule for keeping this file honest
When code is added or changed: update the matching section here **in the same commit**: new function → new flow in §3, new file location → new row in §4, notebook logic moved into `src/` → flip 🔜 to ✅ in §1.
