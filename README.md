# RouteZero

**Autonomous incident routing pipeline. Four agents, medallion architecture, zero invented claims.**

> Built for the AMD Developer Hackathon ACT II (Track 3). Recognized by hackathon mentors as a strong real-world application of agent orchestration for production engineering workflows.

---

## What it does

When a production incident hits, an engineer pastes a stack trace. RouteZero runs it through a four-agent pipeline that classifies the failure, routes it to the owning team with an explained priority decision, and drafts a full Jira ticket plus role-appropriate messages for every stakeholder — full technical ticket for the engineer, leadership-framed summary for the team lead, five-sentence plain-English digest for the manager. A fourth agent mines accumulated incident history for architectural patterns and files proactive flags before the next incident happens.

One governing rule: **if a piece of information was not provided as input, it does not appear in any output.**

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │       Streamlit Dashboard (:8501)        │
                    │  New Incident | History | Arch Intel     │
                    └──────────────────┬──────────────────────┘
                                       │ HTTP
                    ┌──────────────────▼──────────────────────┐
                    │        FastAPI Backend (:8000)           │
                    └──────────────────┬──────────────────────┘
                                       │
  raw error text + context             ▼
 ──────────────────► [A1: Classifier] ──► [A2: Router] ──► [A3: Ticket Writer]
                      regex + rules        rules-first,       Fireworks prose,
                      ZERO LLM calls       LLM only if        hallucination-
                                           conf < 0.65        validated
                           │                    │                  │
                           ▼                    ▼                  ▼
                    ┌──────────────────────────────────────────────────┐
                    │             DuckDB — Medallion Layout            │
                    │  bronze_raw_incidents                            │
                    │  silver_classified_incidents                     │
                    │  silver_routing_decisions                        │
                    │  gold_created_tickets                            │
                    │  gold_incident_intelligence                      │
                    │  gold_audit_runs                                 │
                    │  graph_nodes / graph_edges                       │
                    └───────────────────┬──────────────────────────────┘
                                        │ reads history
                                        ▼
                        [A4: Architectural Auditor]
                    pattern detection + code graph traversal
                                        │
                                        ▼
                      PLM tickets + red/orange/green graph viz
```

### Data pipeline

Raw unstructured error text enters at bronze, gets classified and validated at silver, and exits as structured gold-layer intelligence — the same ELT pattern used in production data platforms, with agent-based transformations in place of dbt models. Pydantic v2 schemas enforce strict data contracts at every stage boundary; malformed payloads are blocked before they reach downstream consumers.

### Agent 1 — Classifier

Fully deterministic. Zero LLM calls. Failure type via regex patterns, service detection via keyword and file-path scoring against the org config, stack trace extraction (Python and Java formats), environment inference, blast radius from affected-user counts. Every classification traces to a specific rule — no model involved, fully auditable.

### Agent 2 — Router

Rules-first. Team ownership, priority with explicit reasoning, stakeholder assembly, and deployment-based probable cause are all deterministic. Fireworks (Gemma2-9b-it) is consulted in exactly one case: when Agent 1 service confidence falls below 0.65. Every routing decision includes a plain-English `routing_reasoning` field.

### Agent 3 — Ticket Writer

Fireworks assembles prose from the verified facts in the routing decision — it is never asked to invent anything. Hallucination validation: any number or file-like token in AI output not present in the input facts causes that text to be discarded and replaced with a deterministic template. Three stakeholder versions per incident: full technical ticket, leadership-framed summary, five-sentence manager digest.

### Agent 4 — Architectural Auditor

On-demand from the dashboard. Reads gold-layer incident intelligence and detects three patterns:

- **Recurring location** — same file:line appearing in 2+ incidents
- **Service stress** — 3+ incidents across 2+ failure types in one service within 7 days
- **Cascading failure** — cross-service incidents within 30 minutes of each other

For each finding, it fetches the actual source code, traverses the two-hop neighborhood in the code knowledge graph, and asks Fireworks to assess the structural weakness — citing real code from real incidents. Flags above 0.70 confidence become PLM Jira tickets. When evidence is thin, it stays silent.

### Code knowledge graph

The StreamCo demo codebase in `demo_repo/` is parsed with Python's stdlib `ast` module into a `networkx` directed graph (nodes: modules, functions, classes; edges: contains, imports, calls). The graph is persisted to DuckDB's `graph_nodes` / `graph_edges` tables and rendered with `pyvis` in the dashboard. Red nodes appeared in 2+ incidents. Orange nodes neighbor red ones. Green nodes are clean.

---

## Tech stack

| Layer | Technology |
|---|---|
| Backend API | FastAPI + uvicorn |
| Frontend | Streamlit |
| Data warehouse | DuckDB (medallion layout) |
| LLM | Fireworks AI — Gemma2-9b-it (`accounts/fireworks/models/gemma2-9b-it`) |
| Code graph | networkx + stdlib ast + pyvis |
| Schema contracts | Pydantic v2 |
| Orchestration | Pure Python — no LangChain, no AutoGen |
| Integrations | Jira, Slack, GitHub REST (all DEMO_MODE-aware) |

All LLM calls use Gemma2-9b-it via Fireworks AI (qualifies for the AMD Hackathon Gemma prize track).

---

## Setup

### Docker (recommended)

```bash
cp .env.example .env        # DEMO_MODE=true works with zero credentials
docker-compose up
```

Backend available at `http://localhost:8000` — Dashboard at `http://localhost:8501`.

The database initializes automatically, five historical incidents seed on first run, and the code knowledge graph builds on the first audit. No manual steps.

### Local development

```bash
python3.11 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env

uvicorn main:app --port 8000 --reload    # terminal 1
streamlit run frontend/app.py            # terminal 2
```

### Environment variables

| Variable | Default | Notes |
|---|---|---|
| `DEMO_MODE` | `true` | When true, Jira and Slack render as previews — no external calls, no credentials needed. |
| `FIREWORKS_API_KEY` | *(empty)* | Without it, deterministic fallback templates are used. Add a real key for AI-written prose. |
| `FIREWORKS_MODEL` | `accounts/fireworks/models/gemma2-9b-it` | Used for every LLM call. |
| `DUCKDB_PATH` | `./database/routezero.db` | Database file location. |
| `BACKEND_URL` | `http://localhost:8000` | Set to the deployed backend URL on Streamlit Cloud. |
| `JIRA_API_TOKEN` / `JIRA_BASE_URL` / `JIRA_EMAIL` | *(empty)* | Only needed with `DEMO_MODE=false`. |
| `SLACK_WEBHOOK_URL` | *(empty)* | Only needed with `DEMO_MODE=false`. |

**`DEMO_MODE=true` demos the entire system with zero credentials.**

---

## Demo walkthrough

Four scenarios in order, all using the dashboard.

**Scenario 1 — Payment failure, rich context.**
Paste `data/sample_errors/payment_error.txt`. In *Add context*: 847 affected users, enterprise tier, production, deployment 37 minutes ago (any commit hash and message), SLA breach in 40 minutes.
Expected: **P1 → payments-team**, manager digest triggered, probable cause linked to the deployment, routing confidence above 0.85.

**Scenario 2 — Auth failure, minimal context.**
Paste `data/sample_errors/auth_error.txt` with no context filled in.
Expected: **P1 → security-team** (auth is critical-path), confidence around 0.75, and a missing-context list showing every field that was not provided.

**Scenario 3 — ML timeout, medium context.**
Paste `data/sample_errors/ml_timeout_error.txt` with affected users under 100 and environment set to staging.
Expected: **P2 → ml-platform-team**, no manager digest, lower blast radius.

**Scenario 4 — The Auditor.**
Open the *Architectural Intelligence* tab and click **Run Audit**. Using the five pre-seeded historical incidents, Agent 4 flags the recurring location at `payment_service/processor.py:31` — a null-reference bug appearing in three separate incidents — with confidence 0.85, files a PLM ticket listing contributing incident IDs, and renders that node red in the code graph.

---

## Running the tests

```bash
python -m pytest tests/ -v
```

80+ tests covering schemas, database layer, all four agents, integration clients (demo mode), code graph, and every API endpoint.

---

## Repository structure

```
agents/          classifier.py, router.py, ticket_writer.py, auditor.py
core/            schemas (Pydantic v2), DuckDB manager, Fireworks client, graph builder
integrations/    Jira, Slack, GitHub clients — all DEMO_MODE-aware
data/            StreamCo org config, sample errors, 5 seeded historical incidents
demo_repo/       Fictional StreamCo codebase — the bug at processor.py:31 is real in the graph
frontend/        app.py — Streamlit dashboard
tests/           Pytest suite
docker/          Backend and frontend Dockerfiles
main.py          FastAPI app — POST /incidents, /audit, /approve, /resolve, /graph/nodes, /stats
```

---

## Live demo

- Frontend: https://routezero-hackathon.streamlit.app
- Backend: https://routezero-hackathon.onrender.com (free tier — open `/docs` first and wait 30 seconds for it to wake)

---

## License

MIT
