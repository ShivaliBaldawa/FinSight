# FinSight — Financial Data Warehouse & NL Query Engine

> A cloud data warehouse that ingests SEC filings, stock prices, and macro indicators → models them into a star schema → lets you query it in plain English via an LLM layer.

---

## What This Project Does

FinSight is an end-to-end financial data platform built across three layers:

1. **Ingestion** — Python pipelines pull data from three public financial APIs and load it raw into Google BigQuery
2. **Transformation** — dbt Core models clean, normalize, and aggregate the raw data into a star schema with analytics-ready mart tables
3. **Query Layer** — A LangChain + Gemini interface lets you ask plain English questions and get SQL-powered answers back

**Example query:**
```
User: "What was the YoY revenue growth for the top 3 companies in 2023?"

Gemini generates →
  SELECT ticker, fiscal_year, yoy_growth_pct
  FROM marts.mart_revenue_trend
  WHERE fiscal_year = 2023
  ORDER BY yoy_growth_pct DESC
  LIMIT 3

BigQuery executes → Gemini narrates the result
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                         │
│   SEC EDGAR XBRL API   │   Yahoo Finance   │   FRED API     │
└────────────┬───────────┴────────┬──────────┴──────┬─────────┘
             │                   │                  │
             ▼                   ▼                  ▼
┌─────────────────────────────────────────────────────────────┐
│                     INGESTION LAYER                         │
│         Python 3.11+ · requests · yfinance · fredapi        │
│   sec_ingest.py   │  prices_ingest.py  │  macro_ingest.py   │
└─────────────────────────────────┬───────────────────────────┘
                                  │ Load to BigQuery
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│                      RAW LAYER (BigQuery)                   │
│   raw.sec_financials  │  raw.stock_prices  │ raw.macro_ind  │
└─────────────────────────────────┬───────────────────────────┘
                                  │ dbt staging models
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│                    STAGING LAYER (dbt)                      │
│   stg_financials.sql  │  stg_prices.sql  │  stg_macro.sql  │
│         Type casting · Renaming · Null filtering            │
└─────────────────────────────────┬───────────────────────────┘
                                  │ dbt mart models
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│                      MARTS LAYER (dbt)                      │
│                                                             │
│  fact_financials   dim_company   dim_date                   │
│  mart_revenue_trend  (YoY via LAG)                          │
│  mart_pe_ratio       (prices × financials join)             │
└─────────────────────────────────┬───────────────────────────┘
                                  │ LangChain SQL chain
                                  ▼
┌─────────────────────────────────────────────────────────────┐
│                      QUERY LAYER                            │
│            LangChain · Gemini 1.5 Flash                     │
│     Plain English → SQL → BigQuery → Narrated Answer        │
└─────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Language | Python 3.11+ | Ingestion scripts |
| Ingestion | `requests`, `yfinance`, `fredapi`, `pandas` | Pull from 3 APIs |
| Warehouse | Google BigQuery (free tier) | Analytical database |
| Transformation | dbt Core | SQL models, tests, lineage |
| LLM | LangChain + Gemini 1.5 Flash | NL → SQL query interface |
| IDE | VS Code | Python + SQL + dbt |

---

## Data Sources

### 1. SEC EDGAR XBRL API → `raw.sec_financials`
- **Source:** `data.sec.gov/api/xbrl/companyfacts`
- **Metrics:** Revenue, Net Income, EPS, Total Assets, Stockholders Equity, Operating Income
- **Coverage:** All 10-K and 10-Q filings ever submitted (no API key required)

### 2. Yahoo Finance → `raw.stock_prices`
- **Source:** `yfinance` library
- **Metrics:** Daily open, high, low, close, volume
- **Coverage:** 2019 to present

### 3. FRED API → `raw.macro_indicators`
- **Source:** Federal Reserve Economic Data
- **Metrics:** Federal Funds Rate, CPI Inflation, Real GDP, Unemployment Rate
- **Coverage:** Full historical series

### Companies Covered
10 S&P 500 companies across technology, finance, and consumer sectors.

---

## Star Schema Design

```
                        ┌──────────────┐
                        │  dim_company │
                        │  ticker (PK) │
                        │  company_name│
                        │  sector      │
                        │  exchange    │
                        └──────┬───────┘
                               │
┌──────────────┐    ┌──────────┴──────────┐    ┌──────────────┐
│   dim_date   │    │   fact_financials   │    │  (prices via │
│  date (PK)   ├────┤  ticker (FK)        │    │  mart layer) │
│  year        │    │  period_end (FK)    │    └──────────────┘
│  quarter     │    │  metric             │
│  month       │    │  value              │
│  weekday     │    │  fiscal_year        │
│  is_weekend  │    │  fiscal_period      │
└──────────────┘    │  form (10-K/10-Q)   │
                    └─────────────────────┘
```

**Mart models built on top:**
- `mart_revenue_trend` — YoY revenue growth using `LAG()` window function
- `mart_pe_ratio` — Price-to-earnings ratio joining `fact_financials` + `raw.stock_prices`

---

## dbt Model Structure

```
models/
├── staging/
│   ├── stg_financials.sql    # Cleans raw SEC data
│   ├── stg_prices.sql        # Cleans raw price data
│   └── stg_macro.sql         # Cleans raw macro data
└── marts/
    ├── fact_financials.sql   # Core fact table
    ├── dim_company.sql       # Company dimension
    ├── dim_date.sql          # Date dimension
    ├── mart_revenue_trend.sql # YoY growth analytics
    └── mart_pe_ratio.sql     # P/E ratio analytics
```

**dbt tests applied:** `not_null`, `unique`, `relationships` (referential integrity) across all mart models.

---

## NL Query Layer

The query interface uses LangChain's SQL chain with Gemini 1.5 Flash:

```python
# Simplified flow
chain = create_sql_query_chain(llm=gemini, db=bigquery_connection)
result = chain.invoke({"question": "What was revenue growth in 2023?"})
```

LangChain handles schema context injection and query execution in a single abstraction, keeping the query layer fully decoupled from the warehouse schema.

---

## Project Status

| Component | Status |
|---|---|
| Ingestion scripts (SEC, prices, macro) | 🔄 In Progress |
| Raw tables in BigQuery | 🔄 In Progress |
| dbt staging models | 🔄 In Progress |
| dbt mart models + tests | 🔄 In Progress |
| LangChain + Gemini query layer | 🔄 In Progress |
| README + documentation | ✅ Done |

---

## Setup & Usage

```bash
# Clone the repo
git clone https://github.com/ShivaliBaldawa/finsight.git
cd finsight

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export FRED_API_KEY=your_key_here
export GOOGLE_APPLICATION_CREDENTIALS=path/to/service_account.json

# Run ingestion
python ingestion/sec_ingest.py
python ingestion/prices_ingest.py
python ingestion/macro_ingest.py

# Run dbt transformations
dbt run
dbt test

# Launch query interface
python query/nl_query.py
```

---

## Why This Architecture

- **BigQuery over Postgres** — financial analytics on millions of rows of XBRL data needs columnar storage and BigQuery's partitioning; Postgres would be too slow for cross-company aggregations at this scale
- **dbt over raw SQL scripts** — lineage tracking, schema tests, and modular mart models make the transformation layer maintainable and auditable
- **LangChain SQL chain** — handles schema context injection and query execution in a single abstraction, keeping the NL layer decoupled from warehouse schema changes
- **Star schema over flat tables** — separating facts from dimensions allows mart models to be written once and reused across multiple analytics views

---

*Built by [Shivali Baldawa](https://linkedin.com/in/shivalibaldawa5)*