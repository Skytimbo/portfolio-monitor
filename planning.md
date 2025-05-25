# Portfolio Monitor MVP - Planning Document

## Project Overview

**MVP Objective:** Build a working portfolio monitoring system that tracks custom triggers for US stocks and ETFs, generates real-time alerts, and provides a simple dashboard - all deployable on Replit within 4 sprints.

**Core Value Proposition:** Automated monitoring that catches portfolio risks before they show up in stock prices, with flexible thresholds for any investment style.

---

## MVP Scope (4 Modules Only)

### What's IN the MVP:
- ✅ US-listed stocks and major ETFs (ETFs: price-based triggers only)
- ✅ Core financial triggers (revenue, margins, price movements)
- ✅ Configurable thresholds per position
- ✅ Email/Slack alerts
- ✅ Simple monitoring dashboard
- ✅ CSV portfolio upload

### What's OUT of the MVP (Phase 2):
- ❌ Advanced backtesting engine
- ❌ Complex reporting system
- ❌ International securities
- ❌ Options/futures/crypto
- ❌ Multi-user/SaaS features
- ❌ ETF fundamental analysis (AUM, tracking error, expense ratios)

---

## Technical Architecture

### Technology Stack (Replit-Optimized)
- **Runtime:** Python 3.9+
- **Data Sources:** yfinance (primary), Alpha Vantage (fallback)
- **Storage:** DuckDB (embedded, persistent)
- **Web UI:** Streamlit (2 simple pages)
- **Alerts:** Email (SMTP) + Slack webhooks
- **Hosting:** Replit (with replit-db for persistence)
- **Scheduling:** Replit cron + external ping service

### Replit-Specific Constraints Addressed
- **Memory Limit:** 2GB max - batch data fetching, persist immediately
- **Process Restarts:** Use replit-db for state, no long-running background tasks
- **No Chrome/Selenium:** Use SEC EDGAR API directly, requests-only scraping
- **Secrets Management:** All API keys via Replit Secrets tab
- **Replit cron Limitations:** 1-minute minimum intervals, jobs can restart
- **Streamlit Production:** Disable watchdog reload (`run_on_save = false`) to avoid restarting during cron runs

---

## MVP Module Architecture

### Current Sprint Goal: Module 1 ✅
**Status:** Ready for development

### Module Dependencies (Linear)
```
Module 1: Core Data → Module 2: Asset Rules → Module 3: Alert Engine → Module 4: Simple UI
```

### **Module 1: Core Data Collection**
**Purpose:** Reliable data fetcher for US stocks and ETFs with fallback strategies
**Input:** List of ticker symbols
**Output:** Standardized financial data in DuckDB
**Replit Considerations:** Batch processing, memory management, API rate limits

**Key Functions:**
```python
def fetch_stock_fundamentals(ticker: str, period: str = "2y") -> Dict
def fetch_etf_metrics(ticker: str, period: str = "2y") -> Dict  # price/volume only
def validate_and_clean_data(raw_data: Dict) -> Dict
def cache_data_with_timestamp(ticker: str, data: Dict) -> bool
def setup_duckdb_wal_mode(db_path: str) -> None  # Prevent write corruption
```

**Data Fallback Strategy:**
1. yfinance (primary, free)
2. Alpha Vantage (5 calls/min + 500 calls/day limit - batch overnight if intraday refresh fails)
3. IEX Cloud or Polygon.io (optional paid fallback via Replit Secrets)
4. Cache-only mode: If all APIs fail, serve stale data + Level 3 "data stale" alert
5. Graceful degradation with partial data

### **Module 2: Asset Rules Engine**
**Purpose:** Apply appropriate monitoring rules based on asset type (stock vs ETF)
**Input:** Asset data + user threshold configuration
**Output:** Configured trigger thresholds per asset
**Dependencies:** Module 1

**Key Functions:**
```python
def classify_asset_type(ticker: str, market_data: Dict) -> str
def apply_stock_rules(stock_data: Dict, user_thresholds: Dict) -> Dict
def apply_etf_rules(etf_data: Dict, user_thresholds: Dict) -> Dict
def generate_smart_defaults(asset_type: str, historical_data: Dict) -> Dict
```

**Asset-Specific Logic:**
- **Stocks:** Revenue growth, gross margins, P/E ratios
- **ETFs:** Price-based triggers only (price movements, volume spikes) - no fundamental analysis
- **Smart Defaults:** Asset-type appropriate threshold suggestions (optional for MVP)

### **Module 3: Alert Generation Engine**
**Purpose:** Evaluate triggers and deliver prioritized alerts
**Input:** Current data + configured thresholds
**Output:** Formatted alerts via email/Slack
**Dependencies:** Module 2

**Key Functions:**
```python
def evaluate_all_triggers(portfolio_data: Dict, thresholds: Dict) -> List[Alert]
def prioritize_alerts(alerts: List[Alert]) -> List[Alert]
def format_alert_message(alert: Alert, delivery_method: str) -> str  # Jinja2 template
def deliver_alert(formatted_alert: str, delivery_config: Dict) -> bool
```

**Alert Priority Levels:**
1. **Level 1 (Immediate):** Existential risks, >20% single-day price moves (close vs previous close)
2. **Level 2 (Daily):** Threshold violations, earnings misses
3. **Level 3 (Weekly):** Valuation opportunities, trend changes, data staleness

**Alert Message Template (Jinja2):**
```jinja2
🚨 {{ alert.ticker }} Alert - Level {{ alert.alert_level }}
{{ alert.trigger_type }}: {{ alert.current_value }} (threshold: {{ alert.threshold_value }})
Recommended Action: {{ alert.recommended_action }}
Time: {{ alert.timestamp.strftime('%Y-%m-%d %H:%M') }}
{% if alert.link %}Details: {{ alert.link }}{% endif %}
```

### **Module 4: Simple Dashboard UI**
**Purpose:** Portfolio configuration and monitoring interface
**Input:** User interactions via Streamlit
**Output:** Portfolio setup + real-time status display
**Dependencies:** Modules 1-3

**Two-Page Streamlit App:**

**Page 1 - Portfolio Setup:**
```python
def portfolio_upload_interface() -> None  # CSV upload + manual entry
def threshold_configuration_panel() -> None  # Edit triggers per position
def test_data_connection() -> None  # Validate tickers and API access
```

**Page 2 - Live Monitor:**
```python
def current_status_dashboard() -> None  # Position status + recent alerts
def manual_refresh_button() -> None  # Force data update
def alert_history_panel() -> None  # Last 30 days of alerts
```

---

## Data Models (Simplified)

### Portfolio Configuration
```yaml
# config.yaml
config_version: 1  # For future compatibility
portfolio:
  name: "Tech Growth Portfolio"
  positions:
    - ticker: "NVDA"
      weight: 0.15
      thresholds:
        revenue_growth_floor: 50
        gross_margin_floor: 70
    - ticker: "QQQ" 
      weight: 0.20
      thresholds:
        price_change_alert: 15  # ETF price-based triggers only
        volume_spike_threshold: 2.0

settings:
  alert_email: "user@example.com"  # Recommend SendGrid/Mailgun for deliverability
  slack_webhook: "https://hooks.slack.com/..."
  refresh_frequency: "daily"
  email_provider: "sendgrid"  # authenticated transactional email API
```

### Alert Data Structure
```python
@dataclass
class Alert:
    ticker: str
    alert_level: int  # 1-3
    trigger_type: str  # "revenue_decline", "margin_compression", etc.
    current_value: float
    threshold_value: float
    timestamp: datetime
    message: str
    recommended_action: str
    link: str = ""  # Optional deep-link to details/EDGAR page
```

---

## Testing Strategy (MVP-Focused)

### **Module Testing Requirements:**
- **Coverage Target:** 85% (realistic for MVP)
- **Test Data:** Fixed JSON fixtures in `tests/fixtures/`
- **Mock Strategy:** All external API calls mocked
- **Performance:** <30 seconds total test suite runtime

### **Integration Testing:**
- **Happy Path:** CSV upload → data fetch → alert generation → dashboard display
- **API Failure:** Graceful fallback to cached data
- **Invalid Input:** Bad tickers, malformed CSV handling
- **Edge Cases:** Market holidays, data gaps, API rate limits

### **Critical Edge Cases (Must Test):**
- **Stock Split Handling:** Ticker with recent split (e.g., NVDA 10-for-1) - verify price adjustment
- **Missing ETF Data:** ETF with incomplete dividend/volume history - graceful degradation
- **API Empty Response:** External API returning empty JSON - error handling and fallback

### **Test Data Requirements:**
- Fixed JSON fixtures in `tests/fixtures/` for all edge cases
- Mock responses for API failure scenarios
- Sample portfolios with mixed stocks/ETFs
- Mock time fixtures for unit-testing cron-interval logic without sleep() calls

### **User Acceptance Criteria:**
- ✅ Upload 10-position portfolio in <2 minutes
- ✅ Generate first alert within 5 minutes of setup
- ✅ Dashboard loads in <10 seconds
- ✅ System recovers from API outages gracefully

---

## Development Sprint Plan

### **Sprint 1: Module 1 (Core Data Collection)**
- **Week 1:** Basic yfinance integration + data validation
- **Week 2:** DuckDB storage + fallback API + comprehensive testing

**Deliverables:**
- Working data fetcher with 85% test coverage
- DuckDB schema with sample data
- API fallback mechanism tested
- Performance benchmarks documented

### **Sprint 2: Module 2 (Asset Rules Engine)**
- **Week 1:** Asset classification + smart defaults
- **Week 2:** Threshold configuration + validation

**Deliverables:**
- Stock vs ETF rule differentiation
- YAML configuration loading
- Threshold validation and bounds checking
- Integration tests with Module 1

### **Sprint 3: Module 3 (Alert Generation Engine)**
- **Week 1:** Trigger evaluation logic
- **Week 2:** Alert formatting + delivery (email/Slack)

**Deliverables:**
- Complete alert evaluation pipeline
- Email and Slack delivery tested
- Alert prioritization working
- End-to-end alert flow functional

### **Sprint 4: Module 4 (Simple Dashboard UI)**
- **Week 1:** Portfolio setup page (Streamlit)
- **Week 2:** Monitoring dashboard + integration testing

**Deliverables:**
- Two-page Streamlit app deployed on Replit
- CSV upload working
- Real-time dashboard functional
- Complete MVP ready for production use

---

## Success Metrics (MVP)

### **Technical Performance:**
- **Data Refresh:** <5 minutes for 20-position portfolio
- **Alert Delivery:** ≤2 minutes from trigger to notification (Replit cron runs every 1 minute max)
- **Uptime:** >95% during market hours (9:30 AM - 4:00 PM EST)
- **Memory Usage:** <1.5GB peak on Replit

### **User Experience:**
- **Setup Time:** <10 minutes for new portfolio
- **Alert Relevance:** <30% false positive rate (measured via manual quarterly review + user feedback UI)
- **Dashboard Usability:** Key info visible without scrolling
- **Error Recovery:** Clear error messages, no silent failures

### **Business Value:**
- **Early Warning:** Detect risks 1+ days before major price moves
- **Actionable Alerts:** Each alert includes specific recommended action
- **Time Savings:** Automated monitoring vs. daily manual checks
- **Portfolio Protection:** Reduce maximum drawdown through early alerts

---

## Risk Mitigation

### **Technical Risks:**
- **API Rate Limits:** Multiple data sources + intelligent caching + cache-only mode
- **Replit Restarts:** State persistence in replit-db + DuckDB with WAL mode
- **Memory Constraints:** Batch processing + immediate persistence
- **Data Quality:** Validation at every step + fallback sources
- **Email Deliverability:** Use authenticated transactional email API (SendGrid/Mailgun recommended)
- **Slack Rate Limits:** 1 message/second limit for burst Level 1 alerts

### **Business Risks:**
- **False Alerts:** Conservative thresholds + user feedback loop
- **Missing Alerts:** Multiple trigger types + redundant monitoring
- **User Adoption:** Simple setup + clear value demonstration
- **Scope Creep:** Fixed 4-module scope with clear Phase 2 boundary

---

## Phase 2 Enhancement Framework

### **Immediate Extensions (Post-MVP):**
The MVP architecture naturally supports these additions without major refactoring:

**Enhanced Analytics Module:**
- Historical backtesting of threshold settings
- Portfolio correlation analysis and risk metrics
- Performance attribution and benchmark comparison

**Advanced Asset Support Module:**
- International securities (ADRs, foreign ETFs)
- Fixed income (bond ETFs, Treasury funds)
- Alternative assets (REITs, commodity funds)

**Professional Features Module:**
- Multi-portfolio management
- Client reporting and white-label dashboards
- Advanced notification channels (SMS, Teams, Discord)
- Options for paid data provider swap (if yfinance becomes unreliable)

### **Long-Term Vision (Phase 3+):**
**Machine Learning Module:** Predictive alerts based on pattern recognition
**Trading Integration Module:** Brokerage API connections for automated actions  
**Community Module:** Shared portfolio templates and crowd-sourced insights
**Enterprise Module:** Multi-user support, role-based access, audit trails

### **Architectural Considerations:**
- Current DuckDB design scales to Phase 2 multi-portfolio needs
- Modular alert system easily extends to new notification channels
- Asset classification framework accommodates new asset types
- Configuration system supports unlimited complexity growth

**Migration Path:** Each phase builds on MVP foundation without breaking changes. Users can upgrade incrementally based on their needs.

---

## Conclusion

This MVP delivers immediate value while establishing a solid foundation for future enhancements. The 4-module design is specifically optimized for Replit Agent development - each module is self-contained, testable, and builds logically on the previous one.

The system will be production-ready after Sprint 4, capable of monitoring real portfolios and generating actionable alerts. The architectural choices ensure that advanced features can be added incrementally without requiring a complete rewrite.

**Current Sprint Goal:** Begin Module 1 development with focus on reliable data collection and Replit deployment constraints.