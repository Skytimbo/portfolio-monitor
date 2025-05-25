# tasks.md

## Sprint 1A – Module 1: Core Data Collection (Part 1)

### **Sprint Goal**
Build basic data fetching and storage for stocks and ETFs with simple fallback.

---

### **Task 1: Setup DuckDB Connection**
**File:** `src/m1_core_data/db.py`

**Requirements:**
- Import `DEFAULT_DB_PATH` from `src.m1_core_data` module
- Function `get_connection(db_path: str = DEFAULT_DB_PATH)` returns DuckDB connection with WAL mode
- Function `init_database(db_path: str = DEFAULT_DB_PATH)` creates fundamentals table only
- Handle basic connection errors

**Simple Schema:**
```sql
CREATE TABLE IF NOT EXISTS fundamentals (
    ticker VARCHAR PRIMARY KEY,
    data JSON,
    last_updated TIMESTAMP,
    data_source VARCHAR
);
```

**Unit Tests (use pytest-mock):**
- `test_connection_with_wal()` - Assert `con.execute("PRAGMA journal_mode").fetchone()[0] == 'wal'`
- `test_table_creation()` - fundamentals table exists after init

**Test Fixtures to Create:**
- `tests/fixtures/empty_db_state.json`

---

### **Task 2: Basic Stock Data Fetcher**
**File:** `src/m1_core_data/stock_fetcher.py`

**Requirements:**
- Function `fetch_stock_fundamentals(ticker: str) -> Dict`
- Use yfinance, extract: revenue, grossMargin, currentPrice only
- Return standardized format, handle errors with try/catch

**Simple Output Format:**
```python
{
    "ticker": "NVDA",
    "asset_type": "stock", 
    "fundamentals": {
        "revenue": 50000000000,  # Most recent quarterly
        "gross_margin": 0.75,
        "current_price": 875.50
    },
    "last_updated": "2025-01-15T10:30:00Z",
    "data_source": "yfinance"
}
```

**Unit Tests (mark live tests with @pytest.mark.live):**
- `test_fetch_valid_stock()` - Mock yfinance, return expected format
- `test_fetch_invalid_ticker()` - Handle ValueError gracefully

**Test Fixtures to Create:**
- `tests/fixtures/yfinance/nvda_response.json`
- `tests/fixtures/yfinance/invalid_ticker_error.json`

---

### **Task 3: Basic ETF Data Fetcher**
**File:** `src/m1_core_data/etf_fetcher.py`

**Requirements:**
- Function `fetch_etf_metrics(ticker: str) -> Dict`
- Use yfinance, extract: currentPrice, volume (dividendYield optional - mark as optional if missing)
- Return same format as stocks but with asset_type="etf"

**Unit Tests:**
- `test_fetch_valid_etf()` - Mock yfinance for QQQ
- `test_etf_vs_stock_detection()` - Correctly sets asset_type

**Test Fixtures to Create:**
- `tests/fixtures/yfinance/qqq_response.json`

---

### **Task 4: Alpha Vantage Fallback**
**File:** `src/m1_core_data/fallback_fetcher.py`

**Requirements:**
- Function `fetch_fallback_data(ticker: str) -> Dict`
- Use Alpha Vantage API with simple rate limiting (time.sleep(12))
- Convert to same format as yfinance fetcher
- Use requests library
- If `ALPHA_VANTAGE_KEY` not set, fallback tests are skipped (mark skip)

**Simple Rate Limiting:**
```python
import time

def fetch_fallback_data(ticker: str) -> Dict:
    time.sleep(12)  # Simple 5-calls/min throttle
    # Make API call...
```

**Unit Tests (use responses library for mocking):**
- `test_alpha_vantage_success()` - Mock successful API response
- `test_alpha_vantage_rate_limit()` - Use monkeypatch to stub time.sleep, verify called

**Test Fixtures to Create:**
- `tests/fixtures/alpha_vantage/nvda_quote.json`

---

## Sprint 1B – Module 1: Core Data Collection (Part 2)

### **Task 5: Simple Cache Manager**
**File:** `src/m1_core_data/cache_manager.py`

**Requirements:**
- Function `cache_data(ticker: str, data: Dict) -> bool` - Store in DuckDB
- Function `get_cached_data(ticker: str) -> Optional[Dict]` - Retrieve from DuckDB
- Function `is_cache_stale(ticker: str, hours: int = 24) -> bool` - Check age

**Unit Tests:**
- `test_cache_round_trip()` - Store and retrieve data
- `test_cache_staleness()` - Age calculation works

---

### **Task 6: Basic CLI Script**
**File:** `src/m1_core_data/batch_fetch.py`

**Requirements:**
- Simple argparse: `python batch_fetch.py --tickers NVDA,MSFT`
- Loop through tickers, try yfinance first, fallback to Alpha Vantage
- Store results in DuckDB via cache_manager
- Print simple progress messages
- Return `sys.exit(0)` on success, `sys.exit(1)` on any failures

**Basic CLI (no fancy features yet):**
```python
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--tickers', required=True)
    args = parser.parse_args()
    
    for ticker in args.tickers.split(','):
        # Fetch and store data
        print(f"Processing {ticker}...")
    
    sys.exit(0)  # Success
```

**Unit Tests:**
- `test_cli_basic_functionality()` - Mock fetchers, verify DB storage

---

## **Definition of Done - Sprint 1A**

### **Functional Requirements ✅**
- [ ] Tasks 1-4 completed with passing unit tests
- [ ] `pytest` coverage ≥80% for completed files
- [ ] Can fetch NVDA data via yfinance and Alpha Vantage
- [ ] Data stored in DuckDB successfully

### **Quality Requirements ✅**
- [ ] All external APIs mocked in tests (use pytest.mark.live for actual calls)
- [ ] Error handling for invalid tickers and API failures
- [ ] Consistent data format across all fetchers

### **Integration Requirements ✅**
- [ ] Code runs on Replit without external dependencies
- [ ] DuckDB file created and readable
- [ ] No crashes on malformed input

---

## **Definition of Done - Sprint 1B**

### **Additional Requirements ✅**
- [ ] Tasks 5-6 completed
- [ ] CLI can process comma-separated ticker list
- [ ] Cache system prevents unnecessary API calls
- [ ] End-to-end: CLI → Fetchers → Cache → DuckDB

---

## **Testing Notes**

**Mock Libraries:**
- Use `pytest-mock` for function mocking
- Use `responses` for HTTP API mocking
- Mark live API tests with `@pytest.mark.live`

**Run Tests:**
```bash
pytest -v -m "not live"           # Skip live API calls
pytest -v tests/test_stock_fetcher.py  # Single file
```

**Test Coverage:**
```bash
pytest --cov=src/m1_core_data --cov-report=html
```

---

## **Stretch Goals (Optional)**

If Sprint 1A completes quickly, add these to Sprint 1B:
- Progress bar for CLI processing
- `--dry-run` flag for CLI
- Data validation schema with pydantic

**Performance Benchmarks (document, don't test):**
- Target: 10 tickers in <3 minutes with live APIs
- Memory: Peak <1GB during processing
- Database: <50MB for 20 tickers

---

## **Next Sprint Preview**

**Sprint 2: Module 2 (Asset Rules Engine)**
- Load portfolio configuration from YAML
- Apply stock vs ETF specific rules
- Generate threshold violations
- Prepare data for alert engine

**Current Sprint Focus:** Get data collection working reliably with good test coverage. Keep it simple and functional.