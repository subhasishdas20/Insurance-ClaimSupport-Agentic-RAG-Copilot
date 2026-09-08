# Enterprise Insurance Claims Policy & Coverage Copilot

An end-to-end Forward Deployed Engineer (FDE) project that converts an Agentic RAG workflow into a deployable internal insurance claims support product using LangGraph, FastAPI, Pinecone, OpenAI, Tavily, HTML, CSS, and JavaScript.

## 1. Business Problem

### Customer
ShieldWave Insurance, a fictional multi-state home & auto insurer with ~1,200 claims agents and adjusters.

### Problem
The claims team maintains many internal documents: homeowner and auto policy wordings, coverage exclusions, deductible schedules, claims intake procedures, escalation rules, and HR-adjacent operations runbooks.

Agents and policyholders still ask repetitive coverage questions because they do not know which clause applies, keyword search returns too many loosely related policy sections, generic chatbots may confidently state a coverage determination even when the evidence is weak, internal documents may not reflect current state regulations, and some questions depend on fresh external information (disaster declarations, filing deadline extensions).

### Example
An employee/agent asks:

> "Is water damage from a burst pipe covered under a standard homeowner policy?"

The answer exists in ShieldWave's private policy wording, so the system should answer from internal policy — citing the specific clause — without searching the public internet.

Another agent asks:

> "What's the extended claims filing deadline after the recent Texas flooding declaration?"

The internal KB may not contain a disaster declared last week. The system should recognize weak private evidence, use external search, grade the evidence, and clearly identify the answer as external information requiring adjuster/compliance validation before it's used in an actual claim decision.

### Business Goal
Build a secure Claims Policy Copilot that:

1. Searches trusted private policy/claims knowledge first.
2. Checks whether retrieved evidence names a specific clause or exclusion.
3. Uses web search only when private knowledge is insufficient.
4. Rewrites weak/ambiguous queries (missing policy line, state, or peril) and retries.
5. Generates grounded, clause-cited answers.
6. Flags any externally-sourced answer as requiring adjuster review.
7. Shows the LangGraph decision path for transparency and auditability.
8. Lets authorized claims/compliance staff add new policy documents.

## 2. Why This Is an FDE Project

A Forward Deployed Engineer does more than build an LLM notebook. The FDE translates a customer problem into a usable product:

```text
Customer Problem
      ↓
Discovery & Requirements
      ↓
Solution Architecture
      ↓
Data / Knowledge Integration
      ↓
Agentic RAG Development
      ↓
API Development
      ↓
User Interface
      ↓
Security + Audit + Testing
      ↓
Deployment
      ↓
Observe + Improve
```

## 3. Simple Architecture

![System architecture](docs/architecture.png)

```text
Policyholder / Claims Agent
        ↓
HTML/CSS/JavaScript Web UI
        ↓ POST /api/chat
FastAPI
        ↓
LangGraph Agentic RAG Controller
        ↓
 ┌───────────────┬─────────────────┐
 ↓               ↓
Private Policy KB    Tavily Web Search
Pinecone              (regulatory fallback only)
 └───────┬───────┘
         ↓
OpenAI LLM
Grounded, Clause-Cited Answer
```

## 4. Agentic RAG Workflow

```text
Question
   ↓
[1] Route Question
   ├── Greeting / simple chat ─────────→ Direct Answer
   │
   └── Claims / coverage question
                ↓
[2] Retrieve from Private Pinecone KB
                ↓
[3] Grade Private Evidence
       ┌────────┴────────┐
       │                 │
     GOOD               WEAK
       │                 │
       ▼                 ▼
Generate from KB   [4] Tavily Web Search
                         ↓
                  [5] Grade Web Evidence
                    ┌────┴─────┐
                    │          │
                  GOOD        WEAK
                    │          │
                    ▼          ▼
       Generate Web (flag:  [6] Rewrite Query
       adjuster review)         ↓
                         Retry Private KB
                               ↓
                        Max retry reached?
                               ↓
                    Insufficient Evidence
                    (flag: adjuster review)
```

## 5. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent workflow | LangGraph | Stateful routing and conditional decisions |
| LLM | OpenAI | Routing, grading, rewriting, answer generation |
| Embeddings | OpenAI `text-embedding-3-small` | Vector embeddings |
| Vector DB | Pinecone | Private enterprise policy/claims knowledge base |
| External search | Tavily | Fallback for state regulations & disaster declarations |
| API | FastAPI | Backend and REST endpoints |
| Frontend | HTML/CSS/JavaScript | Agent- and policyholder-facing interface |
| Audit | SQLite | Decision-path logging + adjuster-review flag |
| Packaging | Docker | Reproducible deployment |

## 6. Project Structure

```text
Enterprise-Insurance-Claims-Agentic-RAG-Copilot/
├── app/
│   ├── api/routes.py
│   ├── core/config.py
│   ├── core/logging.py
│   ├── rag/state.py
│   ├── rag/vectorstore.py
│   ├── rag/workflow.py
│   ├── services/audit.py
│   ├── services/ingestion.py
│   └── main.py
├── data/sample_kb/
│   ├── homeowner_policy_wording.md
│   └── claims_handling_runbook.md
├── static/
│   ├── css/style.css
│   └── js/app.js
├── templates/index.html
├── uploads/
├── docs/
│   └── architecture.png
├── Dockerfile
├── ingest_sample_kb.py
├── requirements.txt
├── run.py
└── README.md
```

## 7. Setup

### Step 1 — Create and activate a virtual environment

```bash
python -m venv venv
```

Windows:

```bash
venv\Scripts\activate
```

macOS/Linux:

```bash
source venv/bin/activate
```

### Step 2 — Install dependencies

```bash
pip install -r requirements.txt
```

### Step 3 — Configure environment

Copy `.env.example` to `.env` and add your keys.

```env
OPENAI_API_KEY=your_openai_api_key_here
TAVILY_API_KEY=your_tavily_api_key_here
PINECONE_API_KEY=your_pinecone_api_key_here
PINECONE_INDEX_NAME=shieldwave-claims-rag
PINECONE_NAMESPACE=company-policy-kb
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small
ADMIN_API_KEY=change-me-in-production
APP_ENV=development
MAX_REWRITES=2
```

### Step 4 — Load sample policy knowledge

```bash
python ingest_sample_kb.py
```

### Step 5 — Run the application

```bash
python run.py
```

Open `http://127.0.0.1:8080` and FastAPI docs at `http://127.0.0.1:8080/docs`.

## 8. Classroom Demo Scenarios

### Demo A — Private KB Success
Ask: **Is water damage from a burst pipe covered under a standard homeowner policy?**

Expected path:

```text
Router → Claims question
Private KB Retrieval
KB Grade → GOOD
Generate from Private KB (cites Section 3.1, $1,000 deductible)
```

### Demo B — Company Policy Question
Ask: **What's the deductible for wind and hail damage?**

Expected result: answer from the internal policy wording table, without web search.

### Demo C — External / Current Information
Ask: **What's the extended claims filing deadline after the recent Texas flooding declaration?**

Expected path when internal policy documents are insufficient:

```text
Router → Claims question
Private KB Retrieval
KB Grade → WEAK
Tavily Search
Web Grade → GOOD
Web Answer (flagged: requires adjuster review)
```

### Demo D — Weak Query Rewrite
Ask an ambiguous claims question such as: **What happens if my claim is wrong?**

If neither private nor web evidence is sufficient, the workflow rewrites the query to clarify policy line / state / peril, retries the KB, and eventually stops with an insufficient-evidence response (also flagged for adjuster review) rather than hallucinating a coverage determination.

## 9. What Changed From the HR Reference

The application structure, graph topology, API shape, retrieval logic, ingestion layer, audit layer, Docker setup, and frontend behavior remain the same. What changed is domain-specific: coverage/claims prompts, Pinecone index and namespace names, UI wording, example questions, sample policy documents, the addition of a `requires_adjuster_review` compliance flag not present in the HR version, and this documentation.

## Attribution

This project's architecture pattern (LangGraph agentic RAG workflow, FastAPI + Pinecone + Tavily stack, ingestion/audit layers) is adapted from [entbappy's Enterprise HR Policy & Employee Support Agentic RAG Copilot](https://github.com/entbappy/Enterprise-HR-Policy-Employee-Support-Agentic-RAG-Copilot), licensed under Apache-2.0. Domain logic, prompts, sample data, and the adjuster-review compliance flag are original to this insurance adaptation.
