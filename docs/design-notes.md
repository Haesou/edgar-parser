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
