# Physical Level 0 DFD Conversion — Design Decisions
## AI-Driven Trading Strategy Builder & Backtesting System
**Team Data Sentinels** · IS 436 · Deliverable 3.2 (Physical)

This document records the decisions made at each of the five steps used to convert the Level 0 logical Data Flow Diagram (Deliverable 3.2 logical) into the physical Level 0 DFD, following the Dennis/Wixom textbook procedure. Decisions made here are consistent with — and aggregated from — the Level 1 physical DFDs already documented in Deliverable 3.1.

---

## Step 1 — Add Implementation References

**Goal of this step.** State *how* each process, data store, and data flow will be implemented at the Level 0 abstraction, so the system-context view describes the actual architecture rather than the abstract business logic.

### Decisions made for processes
At Level 0, every process is a parent that decomposes into multiple Level 1 subprocesses. Each Level 1 subprocess already has its own technology tag (React, FastAPI, Pandas, etc.). For the Level 0 physical diagram, we chose to *aggregate* the child tags into a compact composite tag rather than invent new ones. The rule we applied was: list the dominant technologies that survive the aggregation, in the order they appear in the typical request path (UI → API → compute → persistence).

The mapping we settled on was:
- **Process 1 Strategy Management → "(React + FastAPI + Python AI)"** — aggregates 1.1 Capture Strategy (React), 1.2 Parse Strategy (Python AI), 1.3 Configure Parameters (React), 1.4 Save Strategy (FastAPI), 1.5 Send Alert (WebSocket). React and FastAPI cover the form-and-API path; Python AI is called out because the NLP parser is the single most architecturally significant child.
- **Process 2 Backtesting → "(Pandas + NumPy + FastAPI)"** — aggregates 2.1 Run Backtest (Pandas), 2.2 Build Report (FastAPI), 2.3 Calculate Performance (NumPy), 2.4 Calculate Risk (NumPy), 2.5 Store Results (SQL). Pandas leads the tag because the backtest engine is the dominant compute workload; NumPy supports the metrics; FastAPI is the request boundary.
- **Process 3 Performance Analysis → "(FastAPI + NumPy)"** — at this abstraction the process is a read-mostly service: FastAPI serves the report, NumPy formats the metrics on demand.
- **Process 4 Strategy Deployment → "(FastAPI + Broker SDK + Redis)"** — aggregates 4.1 Load Strategy (FastAPI), 4.2 Place Orders (Broker SDK), 4.3 Listen for Fills (WebSocket), 4.4 Live Monitor (Redis), 4.5 Confirm Deployment (FastAPI). Redis is included at this level because the live-monitor cache is the architectural feature that makes deployment latency acceptable.
- **Process 5 Security & Authentication → "(FastAPI + JWT + bcrypt)"** — aggregates 5.1 Verify Login (FastAPI), 5.2 Issue Tokens (JWT), 5.3 Check Permissions (Redis). bcrypt is included because hashing is what makes the password-comparison flow safe; Redis was omitted from this tag at Level 0 because it appears more prominently inside Process 4's tag.

### Decisions made for data stores
We used the exact same data-store labels at Level 0 that we use at Level 1, so the two diagrams can be cross-referenced without translation:
- D1 → "PostgreSQL: Strategy / Parameters / Rules"
- D2 → "PostgreSQL: Backtests / Reports / Market Data"
- D3 → "PostgreSQL: Users / Alerts"

The only change from the Level 1 labels is that the table list at Level 0 spells out all three primary tables for each store (rather than the dominant table) because the Level 0 view is the place where the reader first encounters each store and should see its full contents at a glance.

### Decisions made for data flows
We applied the same `Data Name (Medium)` convention used at Level 1, but renamed several flows because the Level 0 abstraction allowed a more compact name. Examples:
- "Natural Language Strategy" → "NL Strategy (Web Form)" — shorter form fits inside the available space without losing meaning.
- "Backtest Results" → "Backtest Result (JSON)" — reflects that the response is a single JSON document.
- "Strategy execution requests" → "Order (Broker API)" — uses the canonical industry term for what is actually being sent.
- "Execution feedback" → "Fill (WebSocket)" — names the actual broker event type.
- "Login Credentials" → "Credentials (HTTPS)" — same content, explicit about the security medium.
- "Strategy Rules" → "Rules Select (SQL)" — the SQL operation is part of the name so a reader can see whether the flow reads or writes.
- "Strategy and Parameter Store" → "Strategy Insert (SQL)" — same disambiguation.

### Decisions made for external entities
External-entity names were preserved (per the team decision recorded in D3.1) and a short interface tag was appended where it was useful:
- Trader → "Trader (React Web UI · HTTPS)"
- Market Data Provider → "Market Data Provider (HTTPS, JSON · WebSocket)"
- Brokerage API → "Brokerage API (REST + WebSocket)"
- Authentication Service → "Authentication Service (OAuth / OIDC, HTTPS)"
- Cloud Infrastructure → "Cloud Infrastructure (AWS / GCP, Auto-scale API)"

The Authentication Service tag deserves a specific note. At Level 1 (Diagram 5), we modeled authentication entirely inside Process 5 using FastAPI + JWT + bcrypt. At Level 0, the original logical diagram showed an "Authentication Service" external entity, so at the physical level we interpreted that as a third-party identity provider (OAuth/OIDC) that Process 5 talks to. This preserves the logical architecture, while making clear that internal session and permission state still lives inside Process 5.

---

## Step 2 — Draw the Human–Machine Boundary

**Goal of this step.** Separate the manual parts of the workflow from the automated parts with a single visible line, so it is clear what the system actually does versus what the user does.

### Decisions made
- The boundary is drawn as a dashed rectangle around the Trader external entity, with an italic "Human–Machine Boundary (Manual)" label on the outside. This is the same convention used in every Level 1 physical DFD, so the two abstraction levels read consistently.
- The Trader is the only entity inside the dashed rectangle. Every process and data store sits outside the boundary, on the automated side.
- Market Data Provider, Brokerage API, Authentication Service, and Cloud Infrastructure are placed *outside* the boundary as well, even though they are external systems. They are themselves automated (third-party APIs and cloud services), so they belong on the machine side of the diagram even though they are external to ours.

### Why this placement was chosen
We considered drawing the boundary inside the system to separate the React frontend (running in the user's browser) from the Python backend (running on the cloud). We rejected that interpretation because the textbook definition of human-machine boundary is human-vs-automated, not client-vs-server. Both the React UI and the FastAPI backend are automated; the only human in the loop is the Trader. Putting the boundary at the Trader gives a single clear cut at every level of abstraction (Level 0 and all five Level 1 diagrams).

---

## Step 3 — Add System-Related Components

**Goal of this step.** Add the housekeeping components — authentication, infrastructure, caching, error handling — that have nothing to do with the trading business logic but that any real system needs.

### Decisions made
- **Authentication Service was retained at Level 0 as an external entity.** It already appeared on the logical Level 0 (D3.2 logical), so we chose to keep it visible at the physical level and tag it as a third-party OAuth/OIDC provider. This is consistent with the Level 1 view, where the *internal* parts of authentication (verify, issue, check) sit inside Process 5; at Level 0 we make it explicit that there is an *external* identity service that Process 5 hands off to. This is a system-related component because it has nothing to do with trading, but it is essential to running the system.
- **Cloud Infrastructure was retained at Level 0 as an external entity.** It also appeared on the logical Level 0 and represents the cloud platform on which the application runs. At the physical level we tagged it (AWS / GCP, Auto-scale API) and renamed its incoming/outgoing flows to indicate the cloud-management API medium. This component is purely system-related — it has zero business meaning — but at the Level 0 abstraction it is appropriate to show because the choice of cloud platform is an architectural decision documented in D4.
- **The Cloud Infrastructure flows are connected to Process 5.** This was inherited from the logical diagram and we chose not to re-route them. The reasoning is defensible: Process 5 is the entry point that authenticates every request, and it is also the natural place for the system to enforce resource quotas, request additional storage, and respond to auto-scaling triggers. Routing the cloud flows through Process 5 keeps the diagram readable without lying about the architecture.

### Decisions deliberately *not* made at Level 0
- We chose not to add separate Audit Log or Error Log data stores at Level 0. Audit and error logging are cross-cutting concerns that fire on every state-changing process; representing them at Level 0 would require parallel arrows from every process and would not survive the readability test. They are recorded in the CASE metadata (Step 5) instead.
- We chose not to add a separate Redis data store at Level 0, even though Redis is used inside Processes 4 and 5. Redis content is derived/transient (cached permissions, cached live prices); at Level 0 it is more honest to mention Redis in the tech tag of the process that owns it than to add a fourth data store that contains nothing of business record.
- We chose not to add observability components (metrics, tracing, log aggregation). They are infrastructure concerns sitting underneath the Cloud Infrastructure entity and will be documented in the deployment plan (D5).

---

## Step 4 — Update the Data Elements in the Data Flows

**Goal of this step.** Update the existing data flows so their data definitions reflect any new system-level fields needed for the physical implementation, and rename flows whose data content has changed under the new tech stack.

### Decisions made
- **Token cookie added explicitly to the Trader–Process 5 flow.** The logical diagram showed only `Login Credentials` flowing into Process 5 from the Trader. At the physical level, the response also has to be on the diagram: we added `Token Cookie (HTTPS)` flowing from Process 5 back to the Trader. This is the same JWT cookie shown on Level 1 Diagram 5; surfacing it at Level 0 makes the security loop visible at the system-context view.
- **All cross-boundary flows now carry their security medium.** Anything crossing the human–machine boundary or going to a third-party (Authentication Service, Brokerage API, Market Data Provider, Cloud Infrastructure) now ends in `(HTTPS)`, `(WebSocket)`, `(REST API)`, `(Broker API)`, `(Cloud API)`, or `(OAuth/JWT)`. This makes the encryption and protocol expectations visible without having to read the CASE metadata.
- **Database flows now name the SQL operation.** Every D1, D2, D3 flow uses an `Insert` or `Select` verb so a reader can see at a glance whether the flow modifies the store or reads from it (e.g., Strategy Insert (SQL), Strategies Select (SQL), Rules Select (SQL), Backtest Insert (SQL), Performance Insert (SQL), Hash Select (SQL), User Insert/Select (SQL)).
- **Cloud Infrastructure flows were renamed for accuracy.** "Storage Request" became `Storage Provision (Cloud API)`, "Resource Availability" became `Resource Status (Cloud API)`, and "System Scalability" became `Auto-Scale Trigger (Cloud API)`. The renames make explicit that these are calls into a cloud-management API (rather than vague abstract requests) and that they follow a consistent medium tag.

### Decisions on what *not* to include
- Plaintext passwords do not appear on any Level 0 flow. The login flow carries `Credentials (HTTPS)` once across the wire and is hashed for comparison inside Process 5. The hash itself never crosses the human-machine boundary.
- Broker API keys do not appear on any Level 0 flow. They are stored encrypted in D3 (or in D4 Redis at Level 1) and are looked up inside Process 4 just before each order. Keeping secrets off the diagram is itself a design decision: it makes the diagram safe to share and prevents accidental leakage in screenshots.
- We chose not to expand each flow's data dictionary on the Level 0 diagram itself. Per-flow field lists (`request_id`, `user_id`, `timestamp`, `jwt_token`, `status_code`, etc., as documented in the Level 1 design decisions) are recorded in the CASE metadata so the Level 0 view stays uncluttered.

---

## Step 5 — Update Metadata in the CASE Repository

**Goal of this step.** The CASE tool (Lucidchart) keeps a metadata entry for every process, data store, data flow, and external entity in the diagram. We updated those entries with physical characteristics that are not visible in the drawing itself but are needed for downstream design and implementation work.

### Decisions made for processes
At Level 0, each process metadata record refers downstream to its Level 1 children for fine-grained values, and at the Level 0 entry itself we record:
- **Aggregate implementation stack** (e.g., Process 1 → "Python 3.11 / FastAPI 0.110 / React 18 / OpenAI or Anthropic LLM API").
- **Deployment target** matching the hardware specification in D4 — Standard Web Server, Standard Application Server, or Standard Database Server.
- **Decomposition pointer** — the metadata for each Level 0 process explicitly lists its Level 1 children (e.g., Process 1 decomposes into 1.1, 1.2, 1.3, 1.4, 1.5).
- **Aggregate frequency** computed from the Level 1 children (e.g., Process 4 ≈ 5,000 calls/day during market hours, dominated by 4.3 Listen for Fills).
- **Aggregate latency budget** computed from the slowest critical-path child (e.g., Process 2 backtest report delivery ≤ 5 seconds because 2.1 Run Backtest dominates the latency).

### Decisions made for data stores
Data-store metadata at Level 0 is identical to the Level 1 metadata — the same engine, table list, indexes, retention policy, and growth estimates apply. We did this on purpose so the two abstraction levels never disagree about the same physical store.

### Decisions made for data flows
Each Level 0 flow's metadata records:
- **Medium** (the parenthetical tag from Step 1).
- **Format** — JSON schema reference for Web Form / JSON / WebSocket flows; SQL query template for Insert / Select flows; cloud-API contract reference for Cloud API flows; OAuth/OIDC token format for the Authentication Service flows.
- **Aggregate volume** at Level 0 (e.g., Trader→Process 1 NL Strategy ≈ 50/day; Process 2→D2 Backtest Insert ≈ 30/day; Process 4→Brokerage API Order peak ≈ 50/minute).
- **Latency budget** for cross-boundary flows (e.g., login response ≤ 300 ms, fill notification ≤ 250 ms).
- **Encryption requirement** — TLS in transit for everything that crosses the human–machine boundary or talks to a broker; field-level AES-256 at rest for API credentials and bcrypt at rest for passwords.

### Decisions made for external entities
External-entity records preserved their original names (per the team decision) but now carry physical metadata for system documentation:
- Trader → "React 18 SPA running in modern desktop browsers (Chrome 120+, Safari 17+, Firefox 121+); communicates with the backend over HTTPS and WebSocket."
- Market Data Provider → "Financial Modeling Prep REST API; rate limit 300 requests/minute on our plan; JSON over HTTPS for historical data; WebSocket for live prices."
- Brokerage API → "Multiple supported: Alpaca, Interactive Brokers, Binance, Coinbase Pro; per-broker SDK adapter pattern; REST for orders, WebSocket for fills."
- Authentication Service → "Third-party OAuth 2.0 / OIDC provider; tokens validated against a public JWKS endpoint; vendor TBD (Auth0 or AWS Cognito as primary candidates)."
- Cloud Infrastructure → "AWS or GCP; managed Postgres for D1/D2/D3; managed Redis for the cache layer; auto-scaling group for the FastAPI tier; vendor finalized in D5."

### Decisions deliberately *not* made
- Per-developer ownership was not added to the CASE metadata at this stage. Ownership will be assigned in Deliverable 5 alongside the deployment plan.
- Specific server hostnames or cloud regions were not recorded; those are deployment-time concerns, not analysis-time concerns.

---

## Summary

The biggest design decision at the Level 0 conversion was that the physical Level 0 diagram should *describe the same business processes the logical Level 0 describes*, only with implementation references added. We did not redraw the diagram around layers (frontend / backend / DB), and we did not promote any Level 1 subprocess up to Level 0. Every Level 0 process keeps its original name and decomposes cleanly into the Level 1 physical DFD that already exists for it.

Two components that had no Level 1 representation — Authentication Service and Cloud Infrastructure — were retained at Level 0 because they appeared on the logical Level 0 and represent architecturally significant external systems. Tagging them at the physical level (OAuth/OIDC for Authentication Service, AWS/GCP for Cloud Infrastructure) makes the system-context view honest about the third-party dependencies the system relies on, while keeping the Level 1 view focused on the parts of the system we are building ourselves.
