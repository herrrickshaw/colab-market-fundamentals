# Colab market fundamentals collector

A single Google Colab notebook that pulls roughly five years of dated annual fundamentals (the
Piotroski inputs: net income, revenue, gross profit, total assets, current assets and liabilities,
long-term debt, shares, equity, operating cash flow) for a 32,590-ticker, 20-market universe via
yfinance, and writes one parquet file per market plus a zip you download from Colab. It exists
because Yahoo blocks plain requests from datacenter IPs; the notebook works around that with
`curl_cffi` Chrome impersonation and fails fast when an IP is blocked outright.

> Status: notebook-only drop (last code change 2026-07-22). Research and education only; data comes
> from Yahoo Finance via yfinance and is subject to Yahoo's terms. Not investment advice.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/herrrickshaw/colab-market-fundamentals/blob/main/collect.ipynb)

> **New here?** See [How to use this repo with AI (Claude, Gemini, ChatGPT)](HOW_TO_USE_WITH_AI.md).

## What it does
- Fetches `tickers.csv` (32,590 rows, columns `market,yf_ticker`) straight from this repo — plain CSV, no LFS.
- Probes three blue chips (`AMZN`, `MSFT`, `VOLV-B.ST`) first; if none return data the Colab IP is blocked and the run stops with a "restart runtime" message instead of churning for 40 minutes.
- Fetches income statement, balance sheet and cash flow per ticker through a per-thread `curl_cffi` session (`impersonate='chrome'`), 8 workers, a 0.10 s global throttle and 3 retries with exponential back-off.
- Checkpoints every 3 minutes (and after every market) to `/content/fundamentals/<market>.parquet` and `/content/fundamentals.zip`, so a Colab disconnect loses at most 3 minutes.
- Resumes from a previously downloaded `fundamentals.zip`: already-collected tickers are skipped.

## Quick start
1. Click the Colab badge above (opens `collect.ipynb`).
2. Runtime → Run all. Cell 2 asks for an optional `fundamentals.zip` to resume from; cancel to start fresh.
3. If cell 4 prints `IP probe: 0/3`, Runtime → Restart runtime (new IP) and Run all again.
4. Download `/content/fundamentals.zip` from cell 5 at any time — it is rewritten every 3 minutes.

To run locally instead of Colab:

```bash
git clone https://github.com/herrrickshaw/colab-market-fundamentals.git
cd colab-market-fundamentals
pip install yfinance curl_cffi pandas pyarrow jupyter
jupyter notebook collect.ipynb   # cells 2 and 5 use google.colab.files and will be skipped / fail locally
```

## How it works
| Stage | Where | Output |
|---|---|---|
| 1. Install + load universe | `collect.ipynb` cell 1 | `U` DataFrame, per-market counts |
| 2. Optional resume | cell 2 | extracted parquet files in `/content/fundamentals/` |
| 3. Collector definitions | cell 3 — `fetch()`, `collect_market()`, `flush()`, `probe()` | — |
| 4. Probe + run all markets | cell 4 | `<market>.parquet` per market, `fundamentals.zip` |
| 5. Download | cell 5 | browser download of `fundamentals.zip` |

Output rows are one per ticker per fiscal-year end: `ticker, fy_end, net_income, revenue, gross_profit,
total_assets, current_assets, current_liabilities, long_term_debt, shares, equity, cfo, filed`.

Design decisions visible in the code:
- `filed` is approximated as `fy_end + 90 days`; yfinance does not expose a real filing date.
- India (`BARE = {'IN'}`) tickers are stored without their `.NS` / `.BO` suffix; every other market keeps the yfinance suffix.
- The US market is in `SKIP` and is **not collected** by default (7,441 tickers) — remove `'US'` from `SKIP` in cell 3 to include it.
- Field names are matched against a list of yfinance row labels (e.g. `Ordinary Shares Number` or `Share Issued`) because labels differ by market.

## Repository layout
| Path | Purpose |
|---|---|
| `collect.ipynb` | The collector (5 cells, described above) |
| `tickers.csv` | Ticker universe, 32,590 rows × `market,yf_ticker` — tracked, plain CSV |
| `notebooks/colab_test.ipynb` | Generated smoke-test scaffold (clone + byte-compile); the repo has no `.py` files, so it compiles nothing useful |
| `HOW_TO_USE_WITH_AI.md` | How to hand this repo to an AI assistant |
| `.github/workflows/quality-gates.yml` | gitleaks secret scan on push / PR |

## Ticker universe
| Market | Tickers | Market | Tickers |
|---|--:|---|--:|
| US | 7,441 | DE | 449 |
| JP | 3,709 | SA | 374 |
| CN | 2,895 | BR | 243 |
| HK | 2,793 | CH | 192 |
| KR | 2,768 | ZA | 180 |
| IN | 2,368 | FI | 171 |
| TW | 2,204 | DK | 104 |
| CA | 2,091 | SE | 564 |
| AU | 1,509 | SG | 614 |
| EU | 1,067 | UK | 854 |

## Data sources
| Source | What is taken | Access route | Licence / caveat |
|---|---|---|---|
| Yahoo Finance via [yfinance](https://github.com/ranaroussi/yfinance) | Annual income statement, balance sheet, cash flow | Unofficial API; `curl_cffi` browser impersonation to avoid datacenter-IP blocks | Yahoo terms of use; personal / research use only; may break without notice |
| `tickers.csv` (this repo) | Universe of 32,590 tickers across 20 markets | Raw GitHub URL fetched in cell 1 | Built 2026-07-09; not refreshed |

## Configuration
All settings are constants at the top of cell 3: `OUT` (output dir), `BARE`, `SKIP`, `MIN_INTERVAL`
(0.10 s), `workers=8` in `collect_market()`, 180 s checkpoint interval in `_flusher()`. No secrets
or API keys are needed.

## Limitations and known issues
- `curl_cffi` impersonation is a workaround for Yahoo's IP blocking, not a guarantee; a 0/3 probe means the Colab IP is blocked and only a runtime restart helps.
- US is skipped by default; the full run of the other 19 markets takes well over an hour on Colab.
- yfinance returns only the most recent ~4–5 fiscal years; `filed` dates are estimates (+90 days), so point-in-time backtests built on them are approximate.
- Failed tickers are silently dropped (`fetch()` returns `[]` after 3 retries); the per-market "names with data" count is the only signal.
- Collected parquet files are not committed to this repo.

## Related repositories
- Ticker universe and downstream Piotroski scoring live in the owner's market-pipeline repositories in this account; this notebook is the Colab offload for the fundamentals-collection step.

## Licence
No licence file yet — all rights reserved.
