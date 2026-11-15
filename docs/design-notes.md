# Design notes

Working notes on how EDGAR data behaves and the decisions made because of it.
Findings below come from exploring Nike (CIK 0000320187) in November 2026.

## Data sources

**`submissions`** (`data.sec.gov/submissions/CIK##########.json`): company metadata and filing history.

```
subs
├── name, cik, tickers, sic, fiscalYearEnd, ...     ← company info
└── filings
    ├── recent                                       ← dict of PARALLEL LISTS
    │   ├── accessionNumber: [ ... ]
    │   ├── form:            [ ... ]
    │   ├── filingDate:      [ ... ]
    │   ├── reportDate:      [ ... ]
    │   └── primaryDocument: [ ... ]
    └── files                                        ← older filings, in extra JSON files
```

**`companyfacts`** (`data.sec.gov/api/xbrl/companyfacts/CIK##########.json`): every XBRL fact
the company has reported, across all filings.

```
facts
└── facts
    ├── us-gaap                      ← taxonomy
    │   └── NetIncomeLoss            ← concept (tag)
    │       ├── label, description
    │       └── units
    │           └── USD              ← unit
    │               └── [ {start, end, val, accn, fy, fp, form, filed, frame}, ... ]
    └── dei
        └── EntityCommonStockSharesOutstanding → units → shares → [ ... ]
```

Path to any number: **taxonomy → concept → `units` → unit → list of facts.**

## Exploration log

Format: what I observed, then what the parser has to do about it.

### Filings

**Most filings are not financial reports.** Of Nike's ~1,000 recent filings, 649 are Form 4
(insider trades), 116 are 8-Ks, 48 are Form 144; only 34 are 10-Qs.
→ For now, the only ones that I should parse are:
1. 10-K - Annual Financial Report
2. 10-Q - Quarterly Financial Report
3. 10-K/A, 10-Q/A - Corrections to earlier 10-K/10-Q
4. 10-KT, 10-QT - Transitional Reports When Company Changes Fiscal Year End

Out of scope for now: DEF 14A (proxy statements), registration statements (S-1, S-4, 424B),
ownership forms (3, 4, 5, 144, 13G), and foreign filers (20-F, 40-F, 6-K, which often use IFRS).
8-K Item 2.02 (earnings releases) is noted for later as the earliest public date of results.

**`recent` is not the full history.** It holds the latest ~1,000 filings. Older ones are in extra
files listed under `filings.files` (Nike has two, going back to 1995). 24 of the 70 filings that
reported net income are not in `recent`.
→ The fetcher must also download the files in `filings.files` to see the full filing list.

### Facts and filings

**Facts are already linked to filings.** Every fact carries `accn` (accession number) and `filed`
(filing date), plus copies of `form`, `fy`, `fp`. `accn` joins to `accessionNumber` in
`submissions` and also builds the filing's folder URL.
→ Store `accn` and `filed` on every fact row. `accn` makes the data traceable to the actual
filing; `filed` enables point-in-time queries (only using facts that were public on a given date,
which avoids look-ahead bias).

**`companyfacts` gives all facts at once, but not everything.** One request returns every fact
from every filing, so no per-filing parsing is needed for core numbers. However, `companyfacts` excludes
some information such as segments, geographies, and footnotes. Furthermore, `companyfacts` only covers
2009+ filings.
→ Parser needs two components: a facts normalizer over `companyfacts` (phase 1) and a filing parser for
segments, text, and older data (phase 2).

### Duplicates and revisions

**The same period is reported many times.** `NetIncomeLoss` has 236 entries but only ~110 unique
periods, because each 10-K and 10-Q repeats prior-period comparatives.
→ Store every entry; don't deduplicate at ingestion. Choosing a single value per period happens
at query time (see below).

**Repeated values are not always identical.** Some periods have different values across filings,
e.g. FY2009 net income is 1,486.7M in one filing and 1,487M in a later one. Periods with more than
one value: operating cash flow 17, `SalesRevenueNet` 12, assets 4. Many look like rounding/precision
changes. However, some may be real restatements or reclassifications.
→ Keep every reported value with its `filed` date rather than overwriting. Selection rule at query
time: filter to facts with `filed <= as_of`, then take the most recently filed value for the
period ("latest known as of X"). Optionally support "first reported" too. Flag differences above
a small relative threshold as possible restatements rather than rounding.

**Storage vs. query.** Storage keeps everything; the query layer picks one value per period based
on the `as_of` date. That way the same data answers both "what's the best number today?" and
"what was known on date X?"

### Concepts and tags

**Revenue uses different tags over time.** Nike's 10-Ks used `SalesRevenueNet` through July 2018,
then `RevenueFromContractWithCustomerExcludingAssessedTax` from July 2019 (ASC 606 adoption).
→ Each canonical field needs an ordered list of candidate tags, and the resolved tag can change
from year to year for the same company.

**Keyword matching on tag names is a trap.** Searching for "Revenue" also returns
`CostOfRevenue` (a cost) and `DisposalGroupIncludingDiscontinuedOperationRevenue` (revenue of a
sold business).
→ Tag mappings must be explicit and hand-curated, not substring searches.

### Periods and time

**`fy`/`fp` describe the filing, not the number.** The FY2010 10-K's entry for the period
June 2007 to May 2008 has `fy: 2010`, `fp: FY`. Every comparative in a filing inherits the
filing's `fy`/`fp`.
→ Derive the fiscal period from `start`/`end` (and the company's `fiscalYearEnd`), never from
`fy`/`fp`.

**Fiscal and calendar periods differ.** Nike's fiscal year ends May 31. The quarter ending
2026-08-31 is fiscal `Q1 FY2027` but `frame: CY2026Q3`.
→ Decide the primary period key (fiscal vs. calendar). Comparing companies needs calendar
alignment; reporting per company is natural in fiscal terms. Probably store both.

**Quarterly filings mix 3-month and year-to-date figures.** Income statement 10-Q facts come as
both 3-month and YTD (Q2: 3 and 6 months; Q3: 3 and 9 months). Cash flow 10-Q facts are YTD
only (3, 6, 9 months). A handful of odd durations (e.g. 12-month periods in a Q3 10-Q) also exist.
→ Classify each fact by duration. Derive standalone quarters by subtraction where only YTD exists.

**There is no Q4.** No fact has `fp: Q4`; the fourth quarter is never filed on its own.
→ Q4 = full year (10-K) minus Q1–Q3, or FY minus 9-month YTD.

**Balance-sheet facts are instants.** `Assets` has an `end` but no `start`, and its frame ends in
`I` (`CY2026Q3I`).
→ The fact model needs both duration facts (start + end) and instant facts (end only).

**Shares outstanding is dated by the cover page, not the period.** `dei:EntityCommonStockSharesOutstanding`
in the FY2015 10-K is dated 2015-07-17, weeks after the May 31 fiscal year end; units are `shares`.
→ For per-share values, use the share count with its own date, and don't assume it lines up with
the period end.

### The `frame` field

**`frame` marks one representative fact per calendar period.** Only 74 of 236 net income entries
have it, and it can land on a DEF 14A entry rather than the 10-K for the same period.
→ Useful as a hint or for cross-company `frames` queries, but don't rely on it for picking the
"right" value.

## Data model

<!-- Step 3: what one fact row looks like, canonical fields, tag mapping, how duplicates are resolved. -->

## Open questions

- When two filings report different values for the same period, which wins: first reported or
  latest? Probably both, depending on the query.
- How to tell a rounding difference from a real restatement?
- Primary period key: fiscal, calendar, or both?
- Include 10-K/A and 10-Q/A amendments, and how do they override the originals?
- What to do with odd-duration facts (e.g. 12-month values in a 10-Q)?
- How to handle companies whose fiscal year end changes?
- Should point-in-time logic eventually use the 8-K earnings release date (Item 2.02) instead of
  the 10-K/10-Q filing date, since the market learns the numbers weeks earlier?

## Roadmap

Each phase leaves the repo in a working, showable state.

1. **Core numbers.** Fetcher + `companyfacts`. ~5 canonical fields (revenue, net income,
   operating cash flow, capex, shares outstanding). Annual 10-K data, a handful of companies.
   Storage keeps every version; `as_of` queries.
2. **Complicated numbers.** Still `companyfacts`: quarterly data (YTD subtraction, derived Q4,
   TTM), restatements and amendments, fiscal vs. calendar alignment, ~25 fields (debt, cash, ...),
   more companies including odd ones (banks, unusual fiscal years). Validation suite and coverage
   stats.
3. **First valuation (MVP).** Derived metrics (FCF, margins, growth, net debt, per-share values),
   a simple DCF and peer multiples, a market price source. Output: fair value vs. market price.
4. **Footnotes and filing parsing.** Inline XBRL and `FilingSummary.xml`. Numbers `companyfacts`
   lacks (segments, geographies, dimensional footnote details). Text: footnote TextBlocks and
   10-K sections (1A, 7), stored at sentence/paragraph level with section labels. Use footnote
   data to improve valuation (leases, stock comp, debt schedule, segments).
5. **Text intelligence.** LLM layer, text signals, and point-in-time backtests of the full pricer.

## How the data will be used

```
                ┌─ numbers (companyfacts) ─→ derived metrics ─→ DCF / comps ─┐
filings ─ parser┤                                                             ├→ fair value + signals + explanation
                └─ text (footnotes, 1A, MD&A) ─→ adjustments, flags, NLP features, citations ┘
```

**Numbers → valuation.** Derived metrics: margins, growth, free cash flow (operating cash flow −
capex), net debt (debt − cash), per-share values, TTM figures. Two valuation methods:

- **DCF:** project FCF, discount at cost of capital, add terminal value, subtract net debt, divide
  by shares.
- **Comps:** peer multiples (P/E, EV/EBITDA, EV/Sales, P/FCF) applied to the company's numbers.
  Peers by SIC code, size, or business-description similarity.

What the valuation needs from the parser: operating cash flow and capex, debt and cash, shares
outstanding with its own date, standalone quarters for TTM, calendar-aligned periods for comps,
SIC codes, and `as_of` queries for backtests. Market prices come from a separate source.

**Text → adjustments, signals, explanations.** Footnotes explain why numbers moved, how they
were measured, and what isn't on the statements yet. Facts and text share `accn` + `filed` as
the common key. Text table sketch: `(accn, filed, section_or_note, tag, text, offsets)`.

## Feature ideas (Phases 4–5)

### Hard numbers from footnotes (Phase 4, feed the pricer directly)

Check `companyfacts` first; some of these are already tagged (lease liabilities, remaining
performance obligations, unrecognized tax benefits).

| Value | Footnote | Why it matters |
|---|---|---|
| Debt maturity schedule | Debt | Refinancing risk |
| Average interest rate on debt | Debt | Cost of debt → discount rate |
| Lease obligations | Leases | Debt-like; add to net debt |
| Purchase commitments | Commitments | Locked-in future outflows |
| Litigation accruals, possible loss ranges | Contingencies | Potential future costs |
| Stock-based comp, unvested awards | Stock comp | True cost of pay; future dilution |
| Effective tax rate, unrecognized tax benefits | Income taxes | Forecast tax rate; tax risk |
| Pension funded status | Pensions | Hidden debt |
| Remaining performance obligations / backlog | Revenue | Contracted future revenue |
| Customer concentration | Segments / concentrations | Dependence risk |
| Segment revenue and margins, geographic mix | Segments | Per-segment forecasts; currency exposure |
| Goodwill, impairments | Goodwill | Overpaid acquisitions; write-down risk |
| Inventory breakdown | Inventories | Demand signals |

### LLM layer (Phase 5)

A pipeline component that sends clean text to an LLM for a specific job and stores the result
with the filing's `accn` and `filed` date. Jobs:

1. **Extraction:** prose → structured rows (e.g. "issued $1.0B of 4.25% notes due 2033" →
   debt table row).
2. **Summarization / change detection:** e.g. what changed in Risk Factors since last year.
3. **Classification / flags:** policy change, material lawsuit, going-concern doubt.
4. **Q&A and explanation (RAG):** answer questions about a valuation using retrieved footnote
   chunks, citing each claim to a note in a filing.

Trust rules: validate against XBRL where they overlap and report accuracy; require citations
(`accn` + note + offsets); run once per filing and store outputs (repeatable, cheap,
point-in-time); never use the LLM for numbers XBRL already has.

### Embedding-based scoring (Phase 5)

Original idea: embed sentences and compare (cosine similarity) to an anchor like "high risk".

Problems with the naive version: embeddings capture topic more than tone (every risk-factor
sentence is "about risk"); cosine scores bunch in a narrow positive range with no natural zero;
negation ("we do not expect material losses"); a single short anchor is noisy; boilerplate
dominates.

Better versions:

- **Two-pole axis:** sets of high-risk and low-risk anchor sentences; score by projection onto the
  difference of their averages. Gives a signed score with a meaningful zero.
- **Supervised classifier:** label a few hundred to a few thousand sentences (by hand or with an
  LLM, spot-checked) and train a classifier on the embeddings.
- **Changes over levels:** compare each filing with the company's previous one (*Lazy Prices*,
  Cohen, Malloy & Nguyen).
- **Baselines:** Loughran–McDonald word lists, FinBERT.

Embeddings vs. LLM rating: they overlap on scoring. LLMs handle nuance and negation better;
embeddings are cheaper, faster, deterministic, and also do similarity jobs (retrieval, year-over-year
comparison, peer finding). Combined pattern: LLM labels a sample → spot-check → train an
embedding classifier → run it cheaply on all filings.

### Model choice by label type

| Labels | Model |
|---|---|
| Binary (0/1) | Logistic regression |
| Unordered categories | Multinomial (softmax) logistic regression |
| Multiple topics per sentence | Multi-label: one binary classifier per label |
| Ordered ratings (e.g. risk 1–5) | Ordinal logistic regression (`statsmodels` `OrderedModel`, `mord`); or collapse to binary; or ridge regression |
| Continuous values | Ridge regression; fractional logit for values in [0, 1] |

Evaluate ordered labels with Spearman rank correlation; rare events with precision/recall, not
accuracy.

### Categories to score

**Tone (ordinal/binary):** risk severity, sentiment, uncertainty, confidence (weak vs. strong
modal words), forward-looking optimism, specificity (boilerplate ↔ concrete).

**Topics (multi-label):** liquidity and debt, litigation, regulation, supply chain and inventory,
competition, macro (rates, inflation, currency), geopolitics and regions, cybersecurity, labor
and management, technology / AI, climate and ESG. Aggregated per filing → topic exposure profile.

**Events / red flags (binary, rare):** accounting policy change, restatement, going-concern doubt,
material weakness in internal controls, impairment, major lawsuit or investigation, covenant
breach or waiver, executive or auditor departure, customer loss.

**Statement type:** forward-looking vs. historical; boilerplate vs. company-specific (use as a
filter before other scores); quantitative vs. qualitative (flags candidates for extraction).

**Outcome-based labels (quant):** label text by what happened next (high vs. low next-year
volatility, out- vs. underperformance). Strongest signal if it works, easiest to fool yourself;
needs strict time-based train/test splits.

### Structural features

Year-over-year section similarity, new footnotes or risk factors added, length growth, count of
accounting policy changes, critical audit matters (count, topics), filing timing (days after
period end, late-filing notices), number of restatements or amendments. Readability (e.g. Fog
index): Feng Li found harder-to-read annual reports go with weaker, less persistent earnings.

### Validation plan

Test whether per-filing scores, and their year-over-year changes, predict something measurable
after the filing date: next-year volatility (cleaner) or returns (harder). Point-in-time inputs
only. Compare: embedding anchors vs. LLM ratings vs. embedding classifier on LLM labels vs.
Loughran–McDonald vs. FinBERT vs. no text. Report results honestly, including negative ones.

Suggested first set: boilerplate vs. specific (filter), risk severity, uncertainty, and two or
three topic labels.
