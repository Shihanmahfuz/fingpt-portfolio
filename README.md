# Veris — Portfolio Intelligence

Your portfolio. The truth.

Veris is the first portfolio intelligence platform built on the principle that investors deserve to know the math behind every number they are shown. Not marketing. Not approximations. The actual calculation, the actual source, the actual tax consequence.

Built with a FastAPI backend, vanilla JavaScript frontend, SQLAlchemy persistence, and an optional language model running on Google Colab. All quantitative analytics work offline with zero API cost.

---

## Contributors

| Contributor | Contributions |
|---|---|
| **Askar Kassimov** | Core platform: FastAPI backend, yfinance/FRED/SEC data pipeline, base portfolio analytics, Colab integration with ngrok tunneling, and the initial frontend dashboard. |
| **Shihan Mahfuz** | Institutional-grade analytics rebuild, risk management system, database persistence layer, Veris brand identity, and all features listed below. |

---

## What Veris Shows You

### Mathematics and Analytics Engine

- **Log returns** replacing simple returns across the entire pipeline, with Bessel-corrected standard deviations (ddof=1)
- **Full covariance portfolio volatility**: `sigma_p = sqrt(w^T * Sigma * w)` using the complete correlation structure
- **252-day OLS beta** against SPY as the single canonical beta source
- **Sortino ratio** using downside deviation only
- **Treynor ratio** (excess return per unit of systematic risk)
- **Calmar ratio** (annualized return divided by maximum drawdown)
- **Information ratio** (active return vs SPY divided by tracking error)
- **Jensen's alpha** via the CAPM formula
- **HHI concentration index**, **effective N**, and **diversification ratio**
- **Wilder-smoothed RSI-14** with labeled bands
- **Total return computation** including dividends, with dividend-adjusted cost basis
- **Tax liability engine** at 23.8% LTCG with tax loss harvesting identification
- **AAPL tariff exposure model** with constant P/E price translation
- **CAPM factor attribution** decomposing portfolio return into beta contribution and alpha

### Monte Carlo VaR/CVaR (10,000 simulations)

- **Student-t marginal distribution** fitted per asset via MLE
- **Cholesky decomposition** of the full covariance matrix for correlated multi-asset draws
- **Rescaling** by `sqrt((nu-2)/nu)` so the simulated covariance matches the sample covariance exactly
- **30-day horizon** with daily compounding of log returns
- VaR at 95% and 99%, CVaR (expected shortfall), percentile fan chart

### Efficient Frontier Optimization

- **Ledoit-Wolf shrinkage** on the covariance matrix for stability
- **Black-Litterman style expected returns**: blends implied equilibrium returns with historical means
- **Box constraints** (5% to 60% per name) to produce investable portfolios

### Intraday Minute Tape

- **Minute-resolution OHLCV** pulled live from Yahoo via yfinance
- Six intervals: **1m, 2m, 5m, 15m, 30m, 1h** (plus 60m, 90m)
- **Interval-aware default windows** so Yahoo's caps are never hit (1m→1d, 5m→5d, 30m→1mo, 1h→3mo)
- **UTC-normalized timestamps** on every bar, independent of server timezone
- **Freshness flag** on the newest bar: `age_seconds`, `is_stale` keyed to 3× the bar interval for stocks, 2× for crypto
- **Interval-tuned caching**: 30s TTL for 1m, 120s for 2–5m, 300s for coarser bars
- Per-holding tabs plus free-text symbol input (works for stocks, crypto like `BTC-USD`, indices)

### Options Chain with Black-Scholes Greeks

- Full Black-Scholes pricing with **delta, gamma, theta, vega, rho**
- **Put/call ratio** with positioning interpretation
- **IV skew** with market interpretation
- Risk-free rate sourced live from the 10-year Treasury via FRED

### Stress Testing

- Beta-adjusted losses under 5 historical crises
- **Reverse stress test**: solves for the market drop required to produce a given portfolio loss
- Per-holding beta breakdown cards

### Regime-Aware Risk Engine

- **21-day rolling realized volatility** classifying the market into four regimes
- **Regime-conditional VaR** using only returns observed during the current regime
- Regime transition probabilities and historical regime distribution

### What-If Trade Simulator

- Simulate adding or removing any position before executing
- Before/after/delta for Sharpe, volatility, beta, HHI, effective N, and VaR

### Legendary Investors (SEC 13F)

- Cross-references portfolio stocks against 25+ curated legendary investors
- Dynamic ticker search with cached results

### Event-Risk Calendar

- Upcoming earnings dates with historical earnings-day return distributions

### Data Quality Dashboard

- Per-ticker quality assessment with composite quality score (0-100)

### Database Persistence Layer

- **SQLAlchemy ORM** with 8 models: User, Portfolio, Holding, AnalysisSnapshot, TradeJournal, Watchlist, Alert, AuditLog
- **Multi-backend support**: SQLite (default), MySQL, PostgreSQL
- **User authentication** with session persistence (login, register, guest mode)
- **Auto-snapshot**: every analysis saves a full analytics snapshot for historical tracking
- **Admin dashboard** with credential-protected access to all database tables
- **PBKDF2-HMAC-SHA256** password hashing with bcrypt support
- **30 REST endpoints** under `/api/db/` for full CRUD

---

## Architecture

```
Google Colab (T4 GPU)                          FRED API
  Llama-2-7B + FinGPT LoRA                    (macro data)
  4-bit NF4 quantized                              |
  Exposed via ngrok tunnel                          |
         |                                          |
         | HTTP                                     |
         v                                          v
  +-----------------------------------------------------------+
  |  FastAPI Backend (backend/)                               |
  |                                                           |
  |  app.py ................. API server, 45 endpoints        |
  |  portfolio.py ........... Core analytics engine           |
  |  advanced_analytics.py .. Monte Carlo, Frontier, Stress,  |
  |                           What-If, Regime, Data Quality   |
  |  data_fetcher.py ........ yfinance, FRED, SEC EDGAR       |
  |  options_math.py ........ Black-Scholes + Greeks          |
  |  cache.py ............... In-memory TTL cache             |
  |  model_client.py ........ Colab LLM bridge               |
  |  database/ .............. SQLAlchemy ORM persistence      |
  |    engine.py ............ Connection pooling, multi-DB    |
  |    models.py ............ 8 ORM models                    |
  |    crud.py .............. 35+ CRUD functions + audit log  |
  |    schemas.py ........... Pydantic request/response DTOs  |
  +-----------------------------------------------------------+
         |                            |
         | HTML + JSON API :8000      | SQLAlchemy
         v                            v
  +---------------------------+  +---------------------------+
  |  Veris Dashboard          |  |  Database                 |
  |  (frontend/static/)       |  |  SQLite (default)         |
  |                           |  |  MySQL / PostgreSQL       |
  |  22 analytics panels      |  |  8 tables, audit log,     |
  |  Auth overlay             |  |  auto-snapshots,          |
  |  Admin dashboard          |  |  connection pooling       |
  |  Plotly.js, Chart.js      |  +---------------------------+
  +---------------------------+
```

---

## Quick Start

### 1. Clone and install

```bash
git clone https://github.com/Shihanmahfuz/fingpt-portfolio.git
cd fingpt-portfolio
pip install -r requirements.txt
```

### 2. Configure

```bash
cp config/.env.example config/.env
```

Open `config/.env` and add your FRED API key (free at [fred.stlouisfed.org](https://fred.stlouisfed.org/docs/api/api_key.html)).

### 3. Run

```bash
python3 backend/app.py
```

Open **http://localhost:8000**. Enter your holdings and click "Analyze Portfolio."

### 4. (Optional) Enable AI features

1. Open `colab/fingpt_server.ipynb` in [Google Colab](https://colab.research.google.com/)
2. Runtime > Change runtime type > **T4 GPU** > Save
3. Add `HF_TOKEN` and `NGROK_AUTH_TOKEN` to Colab Secrets
4. Runtime > Run all. Wait 2-4 minutes.
5. Copy the ngrok URL into `config/.env` as `COLAB_MODEL_URL`
6. Restart the backend.

---

## Dashboard Sections

| # | Section | Description |
|---|---------|-------------|
| 1 | **Portfolio Input** | Enter holdings: symbol, shares, avg cost, dividends per share |
| 2 | **Portfolio Summary** | 14 metric cards with disclosed methodology |
| 3 | **AI Insight** | Citation-backed research from the language model (optional) |
| 4 | **Holdings Detail** | 16-column table with per-stock metrics. CSV/JSON export |
| 5 | **Price Charts** | Interactive candlestick charts with SMA overlays |
| 5b | **Intraday Tape** | Minute-resolution OHLCV candles (1m–1h) with live/stale freshness flag |
| 6 | **Monte Carlo VaR** | 10,000 simulated paths, VaR/CVaR, fan chart |
| 7 | **Efficient Frontier** | Markowitz optimization with current vs optimal weights |
| 8 | **Correlation Matrix** | Pairwise correlation heatmap |
| 9 | **Stress Testing** | Beta-adjusted losses under 5 crises + reverse stress test |
| 10 | **Factor Attribution** | CAPM decomposition: beta contribution vs alpha |
| 11 | **Tariff Exposure** | AAPL China COGS shock model |
| 12 | **Tax Liability** | Per-position LTCG at 23.8%, harvestable losses |
| 13 | **Options Chain** | Black-Scholes Greeks, put/call ratio, IV skew |
| 14 | **Allocation** | Doughnut chart of portfolio weights |
| 15 | **Macro Environment** | Live FRED data: Fed Funds, CPI YoY, 10Y Treasury |
| 16 | **News & Sentiment** | Deduplicated headlines with sentiment classification |
| 17 | **Legendary Investors** | SEC 13F cross-reference against 25+ investors |
| 18 | **What-If Simulator** | Preview metric changes before executing a trade |
| 19 | **Regime Risk Engine** | Rolling volatility regime detection with conditional VaR |
| 20 | **Event Calendar** | Upcoming earnings with historical return distributions |
| 21 | **Data Quality** | Per-ticker completeness, freshness, and quality score |
| 22 | **Admin Dashboard** | Credential-protected view of all database tables |

Every metric card discloses its computation parameters. Every info button shows the exact formula.

---

## API Endpoints

### Analytics (16 endpoints)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health` | GET | Backend + model server status |
| `/api/quote/{symbol}` | GET | Real-time stock quote |
| `/api/daily/{symbol}` | GET | Daily OHLCV data |
| `/api/intraday/{symbol}` | GET | Intraday OHLCV bars (1m, 2m, 5m, 15m, 30m, 60m, 90m, 1h) |
| `/api/news` | GET | Financial news for tickers |
| `/api/macro` | GET | FRED macro snapshot |
| `/api/filings/{ticker}` | GET | SEC EDGAR filings |
| `/api/options/{symbol}` | GET | Options chain with Greeks and IV skew |
| `/api/earnings` | GET | Upcoming earnings dates |
| `/api/holders` | GET | Institutional holders and legendary investor matches |
| `/api/portfolio/analyze` | POST | Full portfolio analysis (auto-saves snapshot if logged in) |
| `/api/whatif` | POST | What-if trade simulation |
| `/api/regime` | POST | Regime detection and conditional VaR |
| `/api/data-quality` | POST | Per-ticker data quality report |
| `/api/model/analyze` | POST | Direct LLM access |
| `/api/cache/clear` | POST | Clear the in-memory data cache |

### Database CRUD (30 endpoints)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/db/register` | POST | Create new user account |
| `/api/db/login` | POST | Authenticate user |
| `/api/db/users/{id}` | GET/PUT | User profile |
| `/api/db/users/{id}/dashboard` | GET | User dashboard stats |
| `/api/db/portfolios` | POST | Create portfolio |
| `/api/db/portfolios/{id}` | GET/PUT/DELETE | Portfolio CRUD |
| `/api/db/portfolios/{id}/holdings` | GET/POST | Holdings (auto-merges duplicates) |
| `/api/db/holdings/{id}` | PUT/DELETE | Update or remove holding |
| `/api/db/portfolios/{id}/snapshots` | GET | Paginated snapshot history |
| `/api/db/snapshots/{id}` | GET | Full snapshot detail |
| `/api/db/portfolios/{id}/trades` | GET/POST | Trade journal |
| `/api/db/users/{id}/watchlist` | GET/POST | Watchlist management |
| `/api/db/users/{id}/alerts` | GET/POST | Price/metric alerts |
| `/api/db/admin/login` | POST | Admin authentication |
| `/api/db/admin/overview` | GET | All tables (admin only) |

Full interactive docs at **http://localhost:8000/docs** (Swagger UI).

---

## Database

Veris uses **SQLite by default** — zero configuration, created automatically at `data/veris.db`.

For production, configure MySQL or PostgreSQL via `config/.env`:

```bash
DATABASE_URL=mysql+pymysql://user:password@host:3306/veris
```

The database is completely optional — guest mode works without it.

---

## Project Structure

```
veris/
├── backend/
│   ├── app.py                 # FastAPI server, 45 endpoints
│   ├── portfolio.py           # Core analytics engine
│   ├── advanced_analytics.py  # Monte Carlo, Frontier, Stress, Regime
│   ├── data_fetcher.py        # yfinance, FRED, SEC EDGAR
│   ├── options_math.py        # Black-Scholes pricing and Greeks
│   ├── cache.py               # In-memory TTL cache
│   ├── model_client.py        # Colab LLM bridge
│   └── database/              # SQLAlchemy persistence layer
│       ├── engine.py          # Connection pooling, multi-backend
│       ├── models.py          # 8 ORM models
│       ├── crud.py            # 35+ CRUD functions with audit logging
│       └── schemas.py         # 22 Pydantic DTOs
├── frontend/
│   └── static/
│       └── index.html         # Veris dashboard
├── colab/
│   └── fingpt_server.ipynb    # LLM on Colab T4 GPU
├── config/
│   ├── .env.example           # Template
│   └── .env                   # Your secrets (git-ignored)
├── data/                      # Database file (git-ignored)
├── tests/
│   └── test_spec_validation.py
├── requirements.txt
├── CHANGELOG.md
├── LICENSE
└── README.md
```

---

## Requirements

- Python 3.9+
- Dependencies: `fastapi`, `uvicorn`, `httpx`, `pandas`, `numpy`, `yfinance`, `scipy`, `PyPortfolioOpt`, `SQLAlchemy`
- Browser: Any modern browser
- Optional: Google Colab with T4 GPU for AI features

---

## License

MIT License. See [LICENSE](LICENSE) for details.
