# AI-Driven Trading Strategy Builder & Backtesting System
## Five Functional Interface Design Prototypes — Specification

**Project:** Astral Trading Terminal
**Team:** Data Sentinels (Bruck Asefa, Kavin Manivannan, Chesed Peabody, Ricky Rouse)
**Course:** IS 436 Structured Systems Analysis and Design
**Date:** May 5, 2026

---

## Design System (Applied to All Five Prototypes)

To guarantee consistency across the five screens, every prototype shares the same design system. This addresses the rubric requirement that all prototypes be consistent in layout, navigation, terminology, colors, standards, and overall design style.

**Layout Grid.** Every screen uses a fixed top status bar (system online, market open, SEC compliant, user, EST clock), followed by an F1–F5 tab navigation, followed by a two- or three-column workspace. Validation rule strips appear at the bottom of every screen so the user always sees what controls are protecting their data.

**Color Standards.** Deep navy `#0a1628` background with cyan accent `#4dd0ff`, green `#4dff8c` for valid/positive states, red `#ff4d6d` for invalid/negative states, and amber `#ffcc4d` for warnings. The palette mirrors the Astral landing page brand from the reference imagery.

**Typography.** A single monospace family (`Courier New`) is used everywhere; uppercase labels for field names, mixed-case for values, and bracket markers (`▍`) for section headers. This matches the Bloomberg-terminal aesthetic of the reference dashboard screenshots.

**Navigation.** F1–F5 function-key tabs persist across all screens. The use case identifier (`TW-1` through `TW-6`) is displayed in the top-right of the navigation bar so the user always knows which workflow they are inside, tying the UI directly back to the use cases written in Deliverable 2.

**Terminology.** Field names are drawn directly from the ERD entity attribute names (StrategyName, ParameterName, StartDate, RuleDefinition, BrokerID, etc.) so the user-facing vocabulary matches the data model in Deliverable 4.

---

## Prototype 1 — F1: Strategy Builder (Use Case TW-1)

**Purpose.** First screen in the workflow: a user creates a new trading strategy by typing a plain-English description, which the NLP engine parses into structured TradingRules records.

### 1. Input Selection Controls

All inputs on this screen are React form components, matching the physical DFD tag on processes 1.1 Capture Strategy (React) and 1.3 Configure Parameters (React). The screen uses a compact set of high-signal input controls. The Strategy ID field is a text box with a 40-character maxlength attribute. The Asset/Ticker field is a grouped dropdown list (optgroups for Equities, Crypto, FX) so the user picks from a curated list rather than typing freely. The plain-English strategy description is a textarea with a live character counter and word counter underneath. Two buttons — a primary "Parse & Generate Rules" and a secondary "Clear" — submit or reset the form. The Parse button posts the description to the NLP service (physical Process 1.2 Parse Strategy, Python AI). After parsing, a right-hand panel renders read-only parsed-rule cards with Approve and Edit buttons. Approving the parsed rules persists the strategy through a FastAPI endpoint (physical Process 1.4 Save Strategy), assigns a sequential ST-XXX identifier, prepends a row to the Recent Strategies table, and propagates the new strategy into the F2 (Configure Parameters) and F3 (Backtest Runner) dropdowns. The strategy is deliberately not added to the F5 Deployment dropdown until at least one successful backtest exists, matching the validation rule documented for Prototype 5.

### 2. Input Validation Rules

The Strategy ID field is required, accepts only A–Z, 0–9, and underscore characters, requires a minimum of 3 and maximum of 40 characters, and is auto-uppercased on every keystroke. The Asset/Ticker field is required and rejects submission if the placeholder option is left selected. The strategy description is required, must contain at least 5 words after whitespace tokenization, and is hard-capped at 1,000 characters by the maxlength attribute. Cross-field check at submit: the (UserID, StrategyName) pair must be unique against the Strategy table to prevent duplicates. Live inline hints turn green on valid input and red on invalid input, and the Parse button performs a final guard check before submitting to the NLP service.

### 3. How These Controls Prevent Garbage Data

Each control eliminates a specific class of bad data at the point of entry. The auto-uppercase regex strip on Strategy ID guarantees the column ends up with consistent, indexable values, which is critical because StrategyName participates in the User-to-Strategy foreign key path. The grouped ticker dropdown prevents typo-driven garbage like "APPL" or "btc usd" from ever reaching the MarketData lookup. The minimum word count on the description blocks empty or trivially short descriptions ("buy stock") that the NLP parser cannot meaningfully convert into TradingRules. The character cap defends the database column from oversized payloads. The duplicate check at submit prevents two strategies with the same name from existing under one user, which would corrupt downstream backtest reporting. Together, these rules guarantee that every row written to the Strategy, StrategyParameters, and TradingRules tables in D1 (PostgreSQL: Strategy Parameters) is well-formed, parsable, and uniquely identifiable.

---

## Prototype 2 — F2: Parameters Configuration (Use Case TW-2)

**Purpose.** Configure the StrategyParameters key-value records that govern how a strategy will be sized and risk-managed during backtest and live deployment.

### 1. Input Selection Controls

All inputs on this screen are React form components and submit to a FastAPI endpoint that writes through to D1 (PostgreSQL: Strategy Parameters), matching physical Process 1.3 Configure Parameters. A target-strategy dropdown lists existing strategies the user has created. Numeric spin boxes capture Initial Capital, Risk Per Trade, Stop Loss, Take Profit, and Maximum Drawdown — each with explicit min, max, and step attributes. A range slider (with a paired numeric readout) controls Position Size as a percentage of the portfolio. A radio button group selects Time Horizon (Intraday, Swing, Position) so exactly one option is forced. A checkbox group selects which days of the week the strategy may trade (Mon–Sun, with crypto strategies allowed to enable weekends). Two buttons commit the parameter set: a primary "Save Parameters" and a secondary "Reset to Defaults".

### 2. Input Validation Rules

Every numeric input enforces a typed range and a step increment. Initial Capital accepts $1,000 to $10,000,000 in $1,000 increments. Risk Per Trade accepts 0.1 to 10 percent in 0.1 percent increments. Stop Loss accepts 0.1 to 50 percent. Take Profit accepts 0.1 to 1,000 percent. Maximum Drawdown accepts 1 to 50 percent. The Position Size slider is locked to 1–100 percent so out-of-range values are physically impossible to enter. A cross-field rule requires Take Profit to exceed Stop Loss, otherwise an inline error banner blocks submission. The Time Horizon radio group requires exactly one selection. The Trading Days checkbox group requires at least one day to be checked. All values write to the StrategyParameters entity as ParameterName/ParameterValue pairs only after the full validation suite passes.

### 3. How These Controls Prevent Garbage Data

Risk-management parameters are the highest-stakes data in the system because they directly govern how much real money can be lost in live deployment, so every control is selected to make a bad value impossible to enter rather than merely flagged after the fact. The slider for Position Size guarantees a value in 1–100 percent — the user cannot type "150" or paste in negative numbers. The min/max attributes on the spin boxes are enforced both client-side (browser rejects out-of-range typing) and server-side (the API rejects payloads that bypass the UI). The cross-field Take Profit > Stop Loss check prevents the logical contradiction of a strategy that would close out at a loss before reaching its profit target. Forced one-of-many radio selection on Time Horizon eliminates the ambiguity of "no value" or "multiple values" reaching the database. Required minimum-one-day check on Trading Days prevents the creation of a strategy that can never actually trade. The end result is that the StrategyParameters table is guaranteed to hold only internally consistent, executable, risk-bounded configurations.

---

## Prototype 3 — F3: Backtest Runner (Use Case TW-3)

**Purpose.** Configure and execute a Backtest record against historical MarketData, producing a Backtest row and a linked PerformanceReport.

### 1. Input Selection Controls

A strategy dropdown lists only strategies that have a saved StrategyParameters record (a referential filter). A time-resolution dropdown offers fixed options: 1 minute, 5 minutes, 15 minutes, 1 hour, 1 day. Two HTML5 date pickers capture Start Date and End Date with min/max attributes set to the available MarketData range. Three numeric inputs capture Commission per trade ($0–$100), Slippage (0–5%), and an optional Initial Capital override ($1,000–$10,000,000). A checkbox group toggles execution options (Include Commission, Include Slippage, Intrabar Fills, Walk-Forward Validation). A primary "Run Backtest" button starts execution, and a danger-styled "Abort" button stops it. A live progress bar reports execution status, and a recent-backtests table on the right lists prior runs with READY/RUNNING/FAILED status tags. The Run Backtest button posts the configuration to a FastAPI endpoint that triggers a Pandas/NumPy worker (physical processes 2.1 Run Backtest, 2.3 Calculate Performance, 2.4 Calculate Risk). On completion, the worker assigns a fresh BT-XXX identifier, writes a Backtest row plus a one-to-one PerformanceReport row to D2 (PostgreSQL: Performance Reports / Market Data), and adds the new BT-XXX to the F4 backtest dropdown so the user can immediately review the report.

### 2. Input Validation Rules

Both date pickers use the native HTML5 type-safe calendar, so syntactically invalid dates (Feb 30, etc.) cannot be typed. The min and max attributes restrict the selectable range to 2010-01-01 through today's date, mirroring the MarketData table's available range. A cross-field rule requires End Date to be greater than or equal to Start Date; if violated, the Run button is blocked and an inline error banner is shown. The strategy dropdown is referentially filtered: only strategies that have at least one StrategyParameters record can be selected, preventing an attempt to backtest an unconfigured strategy. The resolution dropdown only exposes intervals that are supported by the backtesting engine. Commission and Slippage have explicit min=0 caps, so negative cost values (which would silently inflate returns) are rejected. The Initial Capital override inherits the same $1K–$10M bounds as Prototype 2 for consistency.

### 3. How These Controls Prevent Garbage Data

The Backtest entity is the source of truth for every PerformanceReport that follows it, so any garbage that lands in this table propagates downstream into reports the user will rely on for live-deployment decisions. Native date pickers make malformed dates unrepresentable in the UI. The min/max date bounds prevent the user from requesting a backtest over a period for which no MarketData exists, which would otherwise produce an empty Backtest with misleading "0%" returns. The End ≥ Start rule prevents inverted ranges that would silently return zero trades. The referential filter on the strategy dropdown prevents the creation of a Backtest with a foreign key pointing to a strategy that has no parameters defined, eliminating a whole class of NULL-pointer failures in the execution engine. The fixed-resolution dropdown ensures the data pipeline only ever has to handle intervals it is engineered for. The non-negative commission and slippage rules prevent the user from accidentally running a "free trading" simulation that would massively overstate returns. Together, these controls ensure that every Backtest row is anchored to a valid Strategy, a valid date range, and economically realistic execution costs.

---

## Prototype 4 — F4: Performance Report (Use Case TW-5)

**Purpose.** Render the PerformanceReport entity for a completed Backtest, including equity curve, trade log, and risk metrics.

### 1. Input Selection Controls

This prototype is intentionally a low-input, high-display screen because the data is system-generated and must not be user-modifiable. The only input is a backtest dropdown that lists every completed BT-XXX run, ordered most-recent first, plus two export buttons (PDF, CSV). Selecting a different backtest re-renders all five top-line metric cards, the equity curve SVG chart, the trade log table, and the risk metrics table from the persisted PerformanceReport for that run — the Y-axis labels on the equity curve auto-scale to the actual min/max equity of the selected backtest, and the trade log entries reflect the date range that backtest was run over. Every visible element is read-only and rendered directly from persisted Backtest and PerformanceReport records, served by a FastAPI endpoint (physical Process 2.2 Build Report). A status tag in the header confirms that the report is ready and shows which BacktestID it was generated from.

### 2. Input Validation Rules

Because the user does not enter any business data on this screen, validation is enforced through display-layer constraints rather than form validation. The backtest dropdown is filtered: only Backtest rows with status READY appear, so a user cannot request a report that has not yet been computed or that failed mid-run. The metric cards, charts, and tables are read-only and contain no editable inputs at all, eliminating any possibility of user-introduced data corruption. The export buttons re-serialize the PerformanceReport record from the database every time they are clicked, so the exported PDF or CSV reflects the canonical stored values, not anything the user might have manipulated in the browser. P&L values are color-coded by sign (green positive, red negative) based on a pure render rule, not a user attribute.

### 3. How These Controls Prevent Garbage Data

This screen prevents data quality problems by removing the user's ability to introduce them in the first place. Because the PerformanceReport record is generated by the backtesting engine and is one-to-one with a Backtest row, any user-side editability would create an audit trail divergence between what was computed and what is displayed. By making the entire screen read-only and tying every visible element to a database record by primary key, the system guarantees that what the user sees is what was actually computed. The filtered dropdown prevents the user from being shown a half-finished or failed backtest as if it were a real result. The PDF and CSV exports re-pull from the database, so no stale or browser-modified intermediate values ever leak into deliverables that might be shared with regulators or clients. The net effect is that the PerformanceReport surface is a perfectly faithful read-only mirror of the data model, by construction.

---

## Prototype 5 — F5: Live Deployment (Use Case TW-6)

**Purpose.** Deploy a backtested strategy to a live or paper-trading broker, creating a Deployment record linked to a Broker entity and producing real-time Alerts.

### 1. Input Selection Controls

All access to this screen requires an authenticated session — the three acknowledgment checkboxes only carry audit value because the user has been identified through physical Diagram 5 (Verify Login → Issue Tokens → Check Permissions), and the JWT cookie issued there is what authorizes the FastAPI deployment endpoints. This is also the most safety-critical screen in the system, so it pairs strong input controls with explicit user acknowledgments. A strategy dropdown lists only strategies with a successful Backtest. A broker dropdown picks the licensed Broker entity the deployment will route through (orders are routed through physical Process 4.2 Place Orders, Broker SDK). A radio group with two options selects Deployment Mode (Paper Trading or Live Trading). Two password-masked text inputs capture the broker API key and API secret. Three numeric inputs cap Daily Loss Limit ($), Maximum Position Size ($), and configure execution thresholds. An email input captures the alert destination. Two HTML5 time pickers define the trading window. Three required acknowledgment checkboxes cover regulatory and risk disclosures. The primary Deploy button is disabled until all required acknowledgments are checked, and live mode triggers an additional confirmation modal. Once deployed, fills are streamed back through physical Process 4.3 Listen for Fills (WebSocket) and rendered live on the screen, while 4.4 Live Monitor (Redis) pushes live performance updates without re-querying the database.

### 2. Input Validation Rules

The strategy dropdown is referentially filtered to require a successful Backtest, preventing deployment of unvalidated strategies. The broker dropdown requires an explicit selection. API credentials are validated against the broker's authentication endpoint before saving and are never logged in plaintext; the secret field requires a minimum length of 16 characters. Daily Loss Limit accepts $100–$100,000 in $100 steps, and Maximum Position Size accepts $100–$500,000. The alert email input uses HTML5 email type validation enforcing the RFC 5322 format. The trading window enforces End > Start and falls within accepted market hours for the selected broker. All three acknowledgment checkboxes must be checked before the Deploy button enables; the button is wired to live state and re-disables if any checkbox is uncovered. Live Trading mode triggers a final confirmation modal that requires the user to type "OK" to proceed, which is logged as a regulatory consent event.

### 3. How These Controls Prevent Garbage Data

Live deployment is the single point in the system where bad data can produce real financial loss, so the controls are deliberately layered and redundant. Filtering the strategy dropdown to backtested strategies only prevents the deployment of an untested or unconfigured strategy — a class of failure that would otherwise corrupt the Deployment table with rows pointing to strategies that have no risk profile. Password masking and on-save broker validation of API credentials prevent typos or paste errors from creating broken Deployment records that would silently fail at order time. Numeric bounds on loss limit and position size prevent fat-finger errors (an extra zero on a position size could be catastrophic). HTML5 email validation prevents malformed alert addresses that would cause the Alerts table to accumulate undeliverable messages. The trading window time-picker rules prevent windows that wrap midnight or run during closed markets, which would otherwise generate orphaned Alert rows with no corresponding fills. The three-checkbox acknowledgment gate, combined with the live-mode confirmation modal, ensures that every live Deployment record carries an unambiguous, auditable user consent trail, satisfying the Cultural/Political Requirements (Trading Risk Disclaimer, Automated Trading Regulation) from the Deliverable 4 nonfunctional requirements matrix. Together these controls ensure that every row in the Deployment table is bound to a validated strategy, valid broker credentials, sane risk caps, a deliverable alert channel, a legal trading window, and a logged user consent — the most defensible deployment record the system can produce.

---

## Mapping to Physical DFDs (Deliverable 3.1)

Each prototype screen is the user-facing surface of one or more processes from the physical Level 1 DFDs. The table below makes that mapping explicit so a reader of D3.1 can move directly between the diagrams and the prototype.

| Prototype | Physical DFD process(es) | Data store(s) touched |
|---|---|---|
| F1 Strategy Builder | 1.1 Capture Strategy (React), 1.2 Parse Strategy (Python AI), 1.4 Save Strategy (FastAPI), 1.5 Send Alert (WebSocket) | D1 PostgreSQL: Strategy Parameters |
| F2 Parameters | 1.3 Configure Parameters (React) | D1 PostgreSQL: Strategy Parameters |
| F3 Backtest Runner | 2.1 Run Backtest (Pandas), 2.3 Calculate Performance (NumPy), 2.4 Calculate Risk (NumPy), 2.5 Store Results (SQL) | D1 (read), D2 PostgreSQL: Performance Reports / Market Data (write) |
| F4 Performance Report | 2.2 Build Report (FastAPI) — read-only consumer of Backtest + PerformanceReport rows produced by F3 | D2 PostgreSQL: Performance Reports / Market Data (read) |
| F5 Live Deployment | 4.1 Load Strategy (FastAPI), 4.2 Place Orders (Broker SDK), 4.3 Listen for Fills (WebSocket), 4.4 Live Monitor (Redis), 4.5 Confirm Deployment (FastAPI) | D1 (read), D3 PostgreSQL: Users / Alerts (write) |
| Implicit on every screen — top-bar "USER:" badge | 5.1 Verify Login (FastAPI), 5.2 Issue Tokens (JWT), 5.3 Check Permissions (Redis) | D3 PostgreSQL: Users / Alerts |
| Behind-the-scenes data freshness | 3.2 Load Market Data (Cron Job), plus 3.1 Strategy Repository, 3.3 Analytics Repository, 3.4 User Service (FastAPI) | D1, D2, D3 |

The human–machine boundary drawn on each physical DFD wraps around the Trader external entity. Every prototype screen sits exactly on that boundary: the React form is what the Trader manually fills in, and everything submitted from it crosses into the automated side of the system.

---

## Cross-Prototype Consistency Summary

All five prototypes draw from the same control vocabulary so the user only has to learn it once. Numeric bounds use spin boxes with visible min/max hints. Closed-set selections use dropdowns or radio groups, never free text. Dates and times use the native HTML5 pickers. Required fields are starred and enforced at submit. Live inline hints turn green on valid and red on invalid input. Cross-field rules surface as full-width banners above the action buttons. Read-only outputs are visually distinct from inputs (no border, muted label color). Action buttons are color-coded by destructiveness — primary cyan for normal saves, red for halt/abort/stop. This single vocabulary, applied consistently across every screen, is what allows the system to enforce a uniformly high data-quality bar from the first user input on F1 all the way to the live order on F5.
