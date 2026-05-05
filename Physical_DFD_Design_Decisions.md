# Physical DFD Conversion — Design Decisions
## AI-Driven Trading Strategy Builder & Backtesting System
**Team Data Sentinels** · IS 436 · Deliverable 3.1

This document records the decisions made at each of the five steps used to convert our Level 1 logical Data Flow Diagrams into physical DFDs, following the Dennis/Wixom textbook procedure. Each step references the actual changes that appear on the five diagrams in the deliverable.

---

## Step 1 — Add Implementation References

**Goal of this step.** State *how* each process, data store, and data flow will be implemented, so the diagram describes the actual system rather than the abstract business logic.

### Decisions made for processes
We chose to keep the original verb–noun action names from the logical DFD and append a parenthetical technology tag to every process box. This was done so that a reader can compare a logical box to its physical counterpart and see at a glance what changed. The technology tags were drawn from the stack already justified in Deliverable 4 (Alternative 1: Custom Full-Stack Python FastAPI + React).

The mapping we settled on for each layer was:
- Browser-facing processes the user fills in or sees → **React** (e.g., 1.1 Capture Strategy, 1.3 Configure Parameters).
- The NLP parser → **Python AI**, since it is implemented as a Python service that calls a language-model API (1.2 Parse Strategy).
- Synchronous backend processes → **FastAPI** (e.g., 1.4 Save Strategy, 2.2 Build Report, 3.1 Strategy Repository, 3.3 Analytics Repository, 4.1 Load Strategy, 4.5 Confirm Deployment, 5.1 Verify Login).
- Compute-heavy processes → **Pandas** for the backtest engine and **NumPy** for the metrics and risk calculations (2.1, 2.3, 2.4).
- Persistence-only processes → **SQL** (2.5 Store Results).
- Real-time, server-pushed processes → **WebSocket** (1.5 Send Alert, 4.3 Listen for Fills).
- Scheduled/system processes → **Cron Job** (3.2 Load Market Data).
- Authentication processes → **bcrypt** for password hashing (3.4 User Service), **JWT** for token issuance (5.2 Issue Tokens), and **Redis** for permission/cache lookups (4.4 Live Monitor, 5.3 Check Permissions).
- Order-routing process → **Broker SDK** (4.2 Place Orders), since the implementation is a per-broker adapter pattern rather than a single technology.

### Decisions made for data stores
We standardized on PostgreSQL as the primary persistence layer (chosen in D4) and renamed each data store to make the engine and the dominant table group explicit:
- D1 → "PostgreSQL: Strategy Parameters" (holds Strategy, StrategyParameters, and TradingRules tables).
- D2 → "PostgreSQL: Performance Reports / Market Data" (holds Backtests, PerformanceReports, and MarketData tables).
- D3 → "PostgreSQL: Users / Alerts tables" (holds Users and Alerts tables).

We chose not to introduce an additional Redis data store on the diagrams, even though Redis is referenced as the implementation technology of two processes (4.4 Live Monitor and 5.3 Check Permissions). The reasoning was that the Redis cache holds derived/transient data, not a primary data store of record, so showing it on the DFD would have added clutter without changing the meaning of any flow.

### Decisions made for data flows
We adopted a single naming convention across all five diagrams: `Data Name (Medium)`, with the medium tag indicating what kind of pipe the data is flowing through. We chose the following set of medium tags and used them consistently:
- `(Web Form)` for browser submissions from the Trader (e.g., Natural Language Strategy Web Form, Backtest Request, Deploy Request, Account Form).
- `(JSON)` for backend response payloads delivered to the browser (e.g., Confirmation, Report).
- `(SQL)` for database reads and writes, with the operation as part of the name (e.g., Strategy Insert, Strategies Select, Rules Select, Market Data Select, Analytics Insert, User Select, Hash Select, Password Check, Market Insert).
- `(WebSocket)` for live, server-pushed streams (e.g., Performance, Risk, Live Update, Live Prices, Feedback).
- `(REST API)` for outgoing calls to the market-data provider (e.g., Market Data).
- `(HTTPS)` for security-critical credential and token flows (e.g., Credentials, Token Cookie).
- `Internal …` prefix for in-memory passes between two backend modules (e.g., Internal Strategy Description, Internal Backtest Results, Internal Performance Data, Internal Trade Result, Internal Status, Internal User Object, Internal JWT Token). This label was important because it tells the reader the data is not crossing a network boundary, which has implications for performance and security.

### Other decisions in this step
- We chose not to rename the external entities (Trader, Market Data Provider, Brokerage API). They are interfaces to actors outside our system, and renaming them would obscure the business meaning. We did, however, append a short interface tag where it was useful — Trader is annotated as "Trader (React Web UI)" and Market Data Provider as "Market Data Provider (HTTPS, JSON)" — so a reader knows the medium of the boundary.
- We kept parenthetical tech tags short (one to three words) so that every label still fits inside a standard DFD process box without forcing a wider drawing.

---

## Step 2 — Draw the Human–Machine Boundary

**Goal of this step.** Separate the manual parts of the workflow from the automated parts with a single visible line, so it is clear what the system actually does versus what the user does.

### Decisions made
- The boundary is drawn as a dashed rectangle around the Trader external entity on every diagram where the Trader appears (Diagrams 1, 2, 3, 4, and 5). The dashed enclosure represents the manual side of the boundary; everything outside the dashed rectangle is automated.
- This placement was chosen because the Trader is the only manual actor in our system. The act of typing a strategy description, configuring parameters, submitting a backtest request, issuing a deployment command, or entering login credentials is performed by a human, but the moment those values are submitted through the React Web UI, the rest of the work is automated.
- Market Data Provider and Brokerage API are placed *outside* the dashed boundary. They are external systems but they are themselves automated, so they belong on the machine side of the diagram even though they are external to ours.

### Why this placement was chosen
We considered drawing the boundary inside the system to separate "frontend (React, runs in the user's browser)" from "backend (Python, runs on our server)." We rejected that interpretation because the textbook definition of human–machine boundary is human-vs-automated, not client-vs-server. Both the React UI and the FastAPI backend are automated; the only human in the loop is the Trader. Putting the boundary at the Trader gives a single clear cut and avoids implying that the React UI is somehow "manual."

---

## Step 3 — Add System-Related Components

**Goal of this step.** Add the housekeeping components — authentication, caching, error handling — that have nothing to do with the trading business logic but that any real system needs.

### Decisions made
- **Authentication was promoted to its own diagram (Diagram 5).** In the logical DFDs, authentication was an implicit assumption ("the user is logged in"). At the physical level it is too important to leave invisible: regulatory, security, and audit requirements all depend on a real authentication path. We therefore made Diagram 5 — Authentication — a first-class part of the deliverable, with three processes (5.1 Verify Login (FastAPI), 5.2 Issue Tokens (JWT), 5.3 Check Permissions (Redis)) and explicit credential, token, and hash flows.
- **Session and permission caching were added inside existing processes rather than as separate processes.** Process 5.3 Check Permissions and 4.4 Live Monitor are tagged "(Redis)" to make explicit that they read from a hot in-memory cache, but no new Redis data store was added to the diagram. The decision was that Redis content is derived data (cached permission lookups, cached live prices) rather than a separate system of record, so putting a fourth data store on the diagram would have inflated the level-1 view without adding business meaning.
- **The cron-driven market-data loader was added as Process 3.2.** This is a system-related process (no Trader interaction) that keeps D2 fresh by pulling from the Market Data Provider on a schedule. Showing it as its own process at level 1 was important because the freshness of D2 directly bounds the validity of every backtest in D2.

### Decisions deliberately *not* made
- We chose not to add separate Audit Log or Error Log data stores at this stage. The justification was that the level-1 physical DFD should still be readable on a single page, and audit/error logging is most accurately modeled as a cross-cutting concern that fires on every state-changing process. We will document audit/error logging in the CASE metadata (Step 5) and in the deployment plan (D5) instead of cluttering the level-1 diagrams with parallel arrows on every process.
- We chose not to add Backup or Observability components, because backups are a database-level operation handled by PostgreSQL's own tooling and observability (metrics, log aggregation) is an infrastructure concern outside the application boundary.

---

## Step 4 — Update the Data Elements in the Data Flows

**Goal of this step.** Update the existing data flows so their data definitions reflect any new system-level fields needed for the physical implementation, and rename flows whose data content has changed under the new tech stack.

### Decisions made
- **Credentials and tokens are explicit on Diagram 5.** The login flow now carries `Credentials (HTTPS)` from the Trader to 5.1 Verify Login, and the response carries `Token Cookie (HTTPS)` back to the Trader. The token is then `Internal JWT Token` between 5.2 Issue Tokens and 5.3 Check Permissions. This is a deliberate change at the data-element level: the logical flow only carried "Login Credentials" and "Authentication Status," whereas the physical flow carries the actual security artifacts (HTTPS-encrypted credentials on the way in, an HttpOnly cookie carrying a signed JWT on the way out). The internal flow between processes now carries a token rather than a status flag.
- **Hash-based password verification is shown explicitly.** The flow from D3 to 5.1 is `Hash Select (SQL)`, returning the bcrypt hash, and the flow back is `Password Check (SQL)`, the comparison of the provided password against the stored hash. This makes it visible that we never compare plaintext to plaintext.
- **Internal flows carry structured objects rather than free text.** Internal Strategy Description (between 1.1 and 1.2), Internal Backtest Results (between 2.1 and 2.2), Internal Performance Data (between 2.2 and 2.3), Internal Passing Calculated Metrics (between 2.3 and 2.4), Internal Analysis Results (between 2.4 and 2.5), Internal Trade Result (between 4.3 and 4.4), Internal Status (between 4.4 and 4.5), Internal User Object (between 5.1 and 5.2), and Internal JWT Token (between 5.2 and 5.3) each represent a Python object passed in memory, not a serialized message. The "Internal" prefix is the convention we adopted to make this distinction visible without adding a new flow style.
- **External boundary flows carry serialized formats.** Where a flow crosses the Trader boundary or talks to an external API, the medium and format are spelled out — `(Web Form)` on submission, `(JSON)` on response, `(WebSocket)` on streamed updates, `(REST API)` on outgoing data-provider calls.
- **SQL flows name the operation.** Every database flow uses an `Insert` or `Select` verb (Strategy Insert, Strategies Select, Rules Select, Market Data Select, Market Insert, Analytics Insert, User Select, Hash Select, Password Check) so that a reader can see at a glance whether the flow modifies the store or reads from it.

### Decisions on what *not* to include
- Plaintext passwords do not appear on any flow after 5.1. They cross the wire as `Credentials (HTTPS)` once, are hashed for comparison inside 5.1, and are then discarded.
- Broker API keys are not shown on any flow on Diagram 4. The credentials live in D4 Redis (encrypted at rest) and are looked up by the Order Router process internally; they are never carried across a flow that the diagram shows. This is itself a design decision — keeping secrets off the diagram makes the diagram safe to share and prevents accidental leakage in screenshots or printouts.

### Open items still to address
The following data-flow updates are still pending in our diagrams and should be reconciled before final submission:
- **Diagram 2's Analytics Insert arrow points to D3 (Users/Alerts).** The data being written is performance and risk analytics, which belongs in D2. The arrow should be redirected from D3 to D2.
- **Diagram 3's Analytics Insert from Process 3.3 also points to D3.** Same correction as above — analytics data belongs in D2 (Performance Reports / Market Data).
- **Diagram 4 has an unlabeled flow named "Text" between 4.1 Load Strategy and 4.2 Place Orders.** This is a placeholder that needs to be renamed; we propose `Internal Strategy Object` to match the naming convention used elsewhere.
- **Diagram 1's flow "Sending Trading Alerts Input Form" reads as an input but is actually an output of 1.5.** It should be renamed `Trading Alerts (WebSocket)` or `Alert Notification (JSON)` so the direction and content are clear.
- **Diagram 4's flow "Submitting Trade Execution Requests" lacks a medium tag.** It should be renamed `Order Request (Broker API)` for consistency with all other physical flow names.

---

## Step 5 — Update Metadata in the CASE Repository

**Goal of this step.** Our CASE tool (Lucidchart) keeps a metadata entry for every process, data store, data flow, and external entity in the diagram. We updated those entries with physical characteristics that are not visible in the drawing itself but are needed for downstream design and implementation work.

### Decisions made for processes
Every process metadata record now includes:
- **Implementation language and framework** (e.g., 1.4 Save Strategy → "Python 3.11 / FastAPI 0.110 / SQLAlchemy 2.0"; 2.1 Run Backtest → "Python 3.11 / Pandas 2.x / NumPy 1.26").
- **Deployment target** matching the hardware specification in D4 — Standard Web Server for Nginx-served React assets, Standard Application Server for FastAPI and the Python compute workers, Standard Database Server for SQL operations.
- **Trigger type** — synchronous request (most FastAPI endpoints), background worker (2.1 Run Backtest), scheduled job (3.2 Load Market Data, runs daily after market close), WebSocket subscriber (4.3 Listen for Fills, 4.4 Live Monitor).
- **Estimated execution frequency** (e.g., 1.1 Capture Strategy ≈ 50/day; 2.1 Run Backtest ≈ 30/day; 4.3 Listen for Fills ≈ 5,000/day during market hours; 5.1 Verify Login ≈ 200/day).
- **Estimated execution time** (e.g., 2.1 Run Backtest ≈ 5–60 seconds depending on date range and resolution; 5.1 Verify Login ≤ 300 ms).

### Decisions made for data stores
Every data-store record now lists:
- **Engine and version** — PostgreSQL 16 for D1, D2, D3.
- **Schema name** and full **table list**, drawn from the ERD in Deliverable 4.
- **Estimated row count and growth rate** (e.g., MarketData starts at ≈ 5M rows and grows ≈ 10K rows/day per ticker; Backtests grows ≈ 30 rows/day; Users ≈ 1K rows total at launch).
- **Indexes** required for the queries shown on the DFDs (e.g., `(strategy_id, created_at)` on Backtest, `(symbol, datetime)` on MarketData, `(user_id)` on Strategy).
- **Retention policy** — MarketData kept indefinitely (data of record), Audit and Error logs rotated separately (see Step 3).

### Decisions made for data flows
Every data-flow record now includes:
- **Medium** (the parenthetical tag from Step 1).
- **Format** — JSON schema reference for Web Form / JSON / WebSocket flows; SQL query template for Insert / Select flows.
- **Average and peak volume** (e.g., Live Update (WebSocket) ≈ 1 message/second per active deployment, peak 10/second; Order Request peak 50/minute).
- **Latency budget** (e.g., login response ≤ 300 ms, backtest report delivery ≤ 5 seconds, fill notification ≤ 250 ms).
- **Encryption requirement** — TLS in transit for everything that crosses the human–machine boundary or talks to a broker; field-level AES-256 at rest for API credentials and bcrypt at rest for passwords.

### Decisions made for external entities
External-entity records were left unchanged in name (preserving business meaning) but now carry physical metadata for system documentation:
- Trader → "React 18 SPA running in modern desktop browsers (Chrome 120+, Safari 17+, Firefox 121+); communicates with the backend over HTTPS and WebSocket."
- Market Data Provider → "Financial Modeling Prep REST API; rate limit 300 requests/minute on our plan; JSON over HTTPS."
- Brokerage API → "Multiple supported: Alpaca, Interactive Brokers, Binance, Coinbase Pro; per-broker SDK adapter pattern; REST for orders, WebSocket for fill notifications."

### Decisions deliberately *not* made
- Per-developer ownership was not added to the CASE metadata at this stage. Process ownership will be assigned in Deliverable 5 alongside the deployment plan.
- Specific server hostnames or cloud regions were not recorded; those are deployment-time concerns, not analysis-time concerns.

---

## Summary

Across the five steps, the biggest single design decision was that the physical DFDs would *describe the same business processes the logical DFDs describe*, rather than redrawing them in terms of layers (frontend / backend / DB). We wanted a reader to be able to compare a logical box to its physical counterpart and immediately see what changed. That is why every process keeps its original action verb, every data flow keeps its original direction and meaning, and the only added artifacts are authentication (Diagram 5) and the cron-driven market-data loader (Process 3.2) — components that exist in any production system but were not visible at the logical level.

The five remaining open items called out in Step 4 (Diagram 2 and Diagram 3 analytics arrows redirected to D2; Diagram 4 "Text" flow renamed; Diagram 1 alert-output flow renamed; Diagram 4 broker-request flow given a medium tag) are minor cleanup items that do not change any of the design decisions captured here.
