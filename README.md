# edgar-parser

A Python library for pulling clean, **point-in-time** financial data out of SEC EDGAR filings:
normalized XBRL financials first, then filing text (footnotes and 10-K sections).

It is the data foundation for a filings-based market pricer (DCF and comparable-company valuation).

## Status

Early development. Nothing usable yet.

## Roadmap

- [ ] Fetcher: ticker → CIK, `submissions` + `companyfacts` download, rate limiting, retries, disk cache
- [ ] Data model for facts and canonical fields
- [ ] XBRL normalizer: 5 core fields, annual (10-K) data
- [ ] Expand to ~25 fields and ~10 companies across sectors
- [ ] Quarterly data (derive standalone quarters from YTD figures)
- [ ] Point-in-time queries (`as_of=` filing date)
- [ ] Validation suite (accounting identities) and coverage stats
- [ ] Inline XBRL parser: dimensions (segments, geographies) and footnote text blocks
- [ ] 10-K section extraction (Items 1, 1A, 7)

## Development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
pytest
```

Copy `.env.example` to `.env` and set `SEC_USER_AGENT`. The SEC requires contact info in the
User-Agent and limits clients to 10 requests per second.

## Design notes

See [docs/design-notes.md](docs/design-notes.md).
