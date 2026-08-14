# Stock Market Intelligence Agent

An institutional-grade long/short equity analysis agent that gathers, validates, and synthesizes market data into a structured investment report. Designed for a long/short equity strategy with strict data integrity rules — it never fabricates data, labels every inference, and halts rather than force a thesis.

---

## Project Structure

```
Stock Agent/
├── agent.py                  ← Main orchestrator — run this
├── config.py                 ← API keys, thresholds, watchlists, constants
├── scoring.py                ← Confidence / Risk / Opportunity scoring engine
├── requirements.txt
├── .env.example              ← Copy to .env and fill in your keys
│
├── data/
│   ├── market_data.py        ← Price, OHLCV, macro indices, sector ETFs (yfinance)
│   ├── fundamentals.py       ← SEC EDGAR, FMP, yfinance earnings & estimates
│   ├── sentiment.py          ← Finnhub, NewsAPI, Reddit, institutional holders
│   └── validator.py          ← DataFlag system: freshness, plausibility, cross-source checks
│
├── analysis/
│   ├── technical.py          ← RSI, MACD, ATR, SMAs, setup verdict (BULLISH/NEUTRAL/BEARISH)
│   ├── fundamental.py        ← Revenue growth, margins, FCF, balance sheet, valuation
│   └── macro.py              ← Market regime classification, sector rotation, macro risks
│
├── screening/
│   └── screener.py           ← 9-signal long screen / 10-signal short screen
│
├── report/
│   └── generator.py          ← Full 11-section report (terminal + .txt file)
│
└── output/                   ← Generated reports saved here (auto-created)
```

---

## Setup

### 1. Install dependencies

```bash
cd "Stock Agent"
pip install -r requirements.txt
```

### 2. Configure API keys

```bash
cp .env.example .env
```

Then edit `.env` with your keys:

| Key | Source | Free Tier |
|-----|--------|-----------|
| `FMP_API_KEY` | [financialmodelingprep.com](https://financialmodelingprep.com/developer/docs) | 250 req/day |
| `FINNHUB_API_KEY` | [finnhub.io](https://finnhub.io/register) | 60 req/min |
| `NEWS_API_KEY` | [newsapi.org](https://newsapi.org/register) | 100 req/day |
| `TWELVE_DATA_API_KEY` | [twelvedata.com](https://twelvedata.com/register) | Optional |
| `REDDIT_CLIENT_ID/SECRET` | [reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) | Optional |

> **Minimum viable setup:** `FMP_API_KEY` + `FINNHUB_API_KEY`. Everything else degrades gracefully with `[DATA UNAVAILABLE]` flags.

---

## Usage

### Run with defaults

```bash
python agent.py
```

Scans the default universe (~30 tickers), generates the full report, saves it to `output/report_YYYYMMDD_HHMM.txt`.

### Custom watchlist + universe

```bash
python agent.py --watchlist AAPL MSFT NVDA --universe AAPL MSFT NVDA TSLA AMD SMCI PLTR
```

### Tune output size

```bash
python agent.py --gems 2 --hicv 1 --shorts 2
```

| Flag | Default | Description |
|------|---------|-------------|
| `--watchlist` | Config default | Tickers always included |
| `--universe` | Config default | Full scan universe |
| `--gems` | 3 | Max hidden gem long candidates |
| `--hicv` | 2 | Max high-conviction long candidates |
| `--shorts` | 3 | Max short candidates |

---

## How It Works — 9-Step Daily Workflow

```
Step 1  Macro environment scan        → Classify: RISK-ON / RISK-OFF / TRANSITIONAL / RANGE-BOUND
Step 2  Data collection               → Price, fundamentals, sentiment for every ticker
Step 3  Data validation               → Flag stale, missing, or conflicting data
Step 4  Long candidate screening      → 9 signals, need ≥5 to qualify
Step 5  Short candidate screening     → 10 signals, need ≥5 to qualify
Step 6  Technical confirmation        → Setup must align with thesis direction
Step 7  Fundamental verification      → Cross-check against SEC EDGAR where possible
Step 8  Scoring                       → Confidence + Risk + Opportunity scores
Step 9  Report generation             → Full 11-section report
```

The agent **halts** (rather than fabricating) if core market data (S&P 500, VIX, Treasury yields) is unavailable.

---

## Screening Signals

### Long (need ≥ 5 of 9)

| # | Signal | Type |
|---|--------|------|
| 1 | Revenue growth accelerating QoQ | Fundamental |
| 2 | Gross margin expansion >100bps | Fundamental |
| 3 | FCF positive or turning positive | Fundamental |
| 4 | Earnings estimate revisions trending up | Fundamental |
| 5 | Undervalued vs peers >20% on EV/EBITDA or P/FCF | Fundamental |
| 6 | Insider buying (open-market, Form 4, last 90 days) | Behavioral |
| 7 | Institutional accumulation | Behavioral |
| 8 | Analyst upgrade or initiation | Behavioral |
| 9 | RS line improving / making new highs | Technical |

### Short (need ≥ 5 of 10)

| # | Signal | Type |
|---|--------|------|
| 1 | Revenue decelerating 2+ consecutive quarters | Fundamental |
| 2 | Margin compression with no recovery path | Fundamental |
| 3 | FCF negative and worsening | Fundamental |
| 4 | Earnings estimate revisions trending down | Fundamental |
| 5 | Net Debt/EBITDA >4x with limited refinancing | Fundamental |
| 6 | Aggressive accounting practices | Fundamental |
| 7 | Cluster insider selling | Behavioral |
| 8 | Institutional distribution | Behavioral |
| 9 | Analyst downgrade or PT cut | Behavioral |
| 10 | Volume-confirmed breakdown below support | Technical |

> Short candidates with elevated squeeze risk (SI >20% float or DTC >5 days) are automatically disqualified unless signal count ≥7.

---

## Scoring System

### Confidence Score (0–100)

| Component | Max | Criteria |
|-----------|-----|----------|
| Fundamental data quality | 25 | SEC=25, FMP=15, estimated=5 |
| Technical confirmation | 20 | Aligns=20, neutral=10, diverges=0 |
| Macro alignment | 15 | Supports=15, neutral=8, headwind=0 |
| Institutional/insider signal | 20 | Both=20, one=10, none=0 |
| Catalyst quality | 20 | Dated+specific=20, general=10, none=0 |

Deductions: -10 per missing critical data point, -10 conflicting sources, -15 macro forecast dependency, -15 short squeeze risk.  
**Capped at 85** unless all components are fully verified.

### Risk Score (0–100)
Valuation risk + Balance sheet risk + Volatility + Execution risk + Macro sensitivity + Liquidity risk.

### Opportunity Score (0–100)
`(Upside% × Confidence%) − (Downside% × Risk%)` — normalized across all candidates.

---

## Report Structure (11 Sections)

1. **Executive Summary** — 3–5 sentence overview
2. **Market Overview** — Macro classification, index levels, sector rotation heatmap
3. **Macro Risks** — Top risks rated LOW/MEDIUM/HIGH/CRITICAL
4. **Top Hidden Gems (Long)** — Small/mid-cap underfollowed longs with full stock card
5. **High-Conviction Longs** — Larger-cap longs with strong evidence base
6. **Short Candidates** — Full card + mandatory squeeze risk assessment
7. **Sector Analysis** — Bullish / Bearish / Transitional sector deep-dives
8. **Technical Watchlist** — 5 stocks near actionable levels
9. **Insider Activity** — Top transactions; cluster flags
10. **Data Quality Footnotes** — Every `[DATA UNAVAILABLE]`, `[STALE]`, `[ESTIMATED]` flag
11. **Final Recommendations** — Ranked by Opportunity Score, one monitor note each

---

## Data Integrity Rules

- **Never fabricate** a price, ratio, or estimate — missing data = `[DATA UNAVAILABLE]`
- **Label every inference** — `[ASSUMPTION]`, `[ESTIMATED]`, `[INFERRED FROM X]`
- **Stale data** (>24h price, >72h fundamentals) reduces confidence scores
- **Conflicting sources** flagged and penalized
- **Chain of assumptions** disqualifies a thesis
- **Reddit / social sentiment** treated as contrarian signal only — never confirmation

---

## Customisation

Edit `config.py` to change:
- `DEFAULT_WATCHLIST` / `DEFAULT_SCAN_UNIVERSE` — tickers to scan
- `LONG_MIN_SIGNALS` / `SHORT_MIN_SIGNALS` — signal thresholds (default 5)
- `MARKET_CAP_MIN_B` / `MARKET_CAP_MAX_B` — hidden gem size filter
- `MAX_ANALYST_COUNT` — underfollowed threshold (default 8)
- `SHORT_INTEREST_SQUEEZE_THRESHOLD` — % float (default 20%)
- All technical indicator periods (RSI, MACD, ATR, SMAs)
