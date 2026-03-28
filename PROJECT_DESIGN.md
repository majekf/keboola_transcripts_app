# Keboola Transcripts App — Project Design Document

> **Document Type**: Multi-Phase Project Analysis & Design  
> **Version**: 1.0  
> **Date**: 2026-03-28  
> **Status**: Design Phase — Ready for Stakeholder Review  

---

## Table of Contents

1. [Phase 1: UNDERSTAND](#phase-1-understand)
2. [Phase 2: QUESTIONING](#phase-2-questioning)
3. [Phase 3: RESEARCH](#phase-3-research)
4. [Phase 4: SYNTHESIS](#phase-4-synthesis)
5. [Phase 5: SOLUTION DESIGN](#phase-5-solution-design)
6. [Phase 6: IMPLEMENTATION PLAN](#phase-6-implementation-plan)

---

## Phase 1: UNDERSTAND

**Dominant Agent**: Understanding Agent + Analysis Agent

### Core Use Case

A web application that enables users to **search, open, analyze, and evaluate call recording transcripts** using configurable LLM prompts, with full evaluation pipeline management.

### Explicit Goals

1. **Transcript Search** — Search transcripts by metadata (from an accompanying document/sprievodka) and by text content (regex-based initially)
2. **Transcript Viewing** — Open and read individual recording transcripts
3. **Prompt Execution** — Run LLM prompts against transcripts (e.g., sentiment analysis, objection extraction, client need summarization)
4. **Result Persistence** — Store prompt outputs as new attributes linked to recordings in the database
5. **Prompt Management** — Create, edit, and version prompts (v1, v2, v3...) with full audit trail
6. **Custom Fields** — Allow users to define custom output fields (e.g., "Agreement with offer: Yes/No")
7. **LLM Model Selection** — Support multiple LLM models with ability to switch between them
8. **Evaluation Pipeline** — Maintain evaluation sets (100+ pairs of "required info" → "correct answer"), add/remove examples at any time, evaluate prompt+model accuracy
9. **Dashboard** — Visualize results and evaluation metrics

### Implicit Goals

- **Traceability**: Every result must be traceable to the specific prompt version + LLM model that produced it
- **Iterative Improvement**: The system must support rapid prompt iteration and comparison
- **Scalability**: The evaluation set and prompt library will grow over time
- **Multi-user**: Multiple users will likely interact with the system (⚠️ ASSUMPTION)

### Constraints

| Category | Constraint |
|----------|-----------|
| **Platform** | Keboola ecosystem integration implied by the name |
| **Search** | Text search is regex-based initially (not semantic/vector) |
| **Language** | Application context is Slovak (SK) — transcripts and UI labels are in Slovak |
| **Evaluation Set** | Minimum 100 pairs, must be editable at any time |
| **Prompt Versioning** | Must maintain version history and link results to specific versions |
| **LLM Models** | Must be model-agnostic and future-proof for new models |

### Actors

| Actor | Role |
|-------|------|
| **Analyst/User** | Searches transcripts, runs prompts, reviews results |
| **Prompt Engineer** | Creates/edits prompts, defines custom fields, runs evaluations |
| **System Admin** | Manages LLM model configurations, database (⚠️ ASSUMPTION) |
| **LLM Service** | External AI model that processes prompts against transcripts |
| **Database** | Stores transcripts, metadata, prompts, results, evaluation sets |
| **Keboola Platform** | Source of transcript data and metadata (⚠️ ASSUMPTION) |

---

## Phase 2: QUESTIONING

**Dominant Agent**: Understanding Agent (Clarifier)

> ⚠️ **Note**: Since this is a design document rather than a live stakeholder session, questions are listed with **reasonable assumptions** where answers were not provided. All assumptions are clearly labeled and should be validated with the stakeholder.

### Iteration 1 — Scope & Data Questions

| # | Question | Assumed Answer | Confidence |
|---|----------|---------------|------------|
| Q1 | What is the source of transcripts? Are they already in Keboola Storage, or do they come from an external transcription service (e.g., Whisper, Google STT)? | ⚠️ **ASSUMPTION**: Transcripts are already available in a Keboola Storage table or external database, pre-transcribed. | Medium |
| Q2 | What metadata is available in the "sprievodka" (accompanying document)? Specific fields like date, agent name, customer ID, call duration, etc.? | ⚠️ **ASSUMPTION**: Metadata includes at minimum: call ID, date/time, agent ID/name, customer ID, call duration, call type/category. | Medium |
| Q3 | How many transcripts are in the database currently, and what is the expected growth rate? | ⚠️ **ASSUMPTION**: Thousands to tens of thousands of transcripts, growing by hundreds per week. | Low |
| Q4 | What is the average transcript length (tokens/words)? This affects LLM cost and context window requirements. | ⚠️ **ASSUMPTION**: Average call transcript is 2,000–5,000 words (fits within most LLM context windows). | Medium |
| Q5 | Which LLM providers should be supported initially? OpenAI GPT-4, Azure OpenAI, Anthropic Claude, local models? | ⚠️ **ASSUMPTION**: OpenAI (GPT-4/GPT-4o) as primary, with architecture supporting additional providers. | Medium |
| Q6 | Is there an existing authentication/authorization system, or does this need to be built? | ⚠️ **ASSUMPTION**: Basic authentication is needed; can leverage Keboola's auth if deployed as a Keboola component. | Low |
| Q7 | Should prompt execution happen synchronously (user waits) or asynchronously (background processing with notifications)? | ⚠️ **ASSUMPTION**: Both — synchronous for single transcript, asynchronous for batch processing. | Medium |
| Q8 | What is the expected format of evaluation set pairs? Is "correct answer" a free-text string, a categorical value, or structured data? | ⚠️ **ASSUMPTION**: Mix of categorical (Yes/No, sentiment labels) and short free-text answers depending on the prompt type. | Medium |

### Iteration 2 — Technical & Integration Questions

| # | Question | Assumed Answer | Confidence |
|---|----------|---------------|------------|
| Q9 | Should the application be deployed as a standalone web app, a Keboola component/app, or both? | ⚠️ **ASSUMPTION**: Standalone web application that reads/writes to a database also accessible by Keboola. | Medium |
| Q10 | What database is preferred? PostgreSQL, Snowflake (Keboola's default), MySQL, or other? | ⚠️ **ASSUMPTION**: PostgreSQL for the application database, with potential Snowflake integration for Keboola Storage. | Medium |
| Q11 | How should prompt accuracy be measured? Exact match, fuzzy match, semantic similarity, or human review? | ⚠️ **ASSUMPTION**: Exact match for categorical outputs; for free-text, a combination of LLM-as-judge and human review. | Medium |
| Q12 | Is there a budget constraint for LLM API calls? Should the app track and display cost per prompt execution? | ⚠️ **ASSUMPTION**: Cost awareness is important but not a blocker; tracking cost per execution is a nice-to-have. | Low |
| Q13 | How many concurrent users are expected? | ⚠️ **ASSUMPTION**: Small team (5–20 users), not high-concurrency. | Medium |
| Q14 | Should the dashboard be real-time or is periodic refresh acceptable? | ⚠️ **ASSUMPTION**: Near real-time (refresh on demand), not streaming. | High |
| Q15 | Is there a preference for frontend framework (React, Vue, Streamlit, etc.)? | ⚠️ **ASSUMPTION**: No strong preference; choose based on speed of development and maintainability. | Low |
| Q16 | Should prompt versioning include diff/comparison views between versions? | ⚠️ **ASSUMPTION**: Yes — being able to compare two prompt versions side by side is valuable for iterative improvement. | Medium |

### Context Re-evaluation After Questioning

After two iterations of questioning, the following **key insights** emerged:

1. **Data pipeline is external**: Transcripts arrive pre-processed; the app focuses on analysis, not transcription.
2. **Evaluation is core**: The evaluation pipeline is not a secondary feature — it's central to the prompt engineering workflow.
3. **Versioning is critical**: Every result must be linkable to a specific prompt version + LLM model combination.
4. **Flexibility is paramount**: Users need to create arbitrary prompts with arbitrary output fields — the schema must be dynamic.
5. **Keboola integration** scope is unclear — this is the biggest risk area.

**Readiness Assessment**: Sufficient clarity to proceed to Research phase, with assumptions documented. ✅

---

## Phase 3: RESEARCH

**Dominant Agent**: Search Agent + Skeptic Agent

### Relevant Architectures & Patterns

#### 1. Prompt Management Systems

**Industry practice**: Tools like LangSmith, PromptLayer, Humanloop, and Weights & Biases Prompts provide:
- Prompt versioning with git-like semantics
- A/B testing of prompt variants
- Execution logging with full input/output capture
- Evaluation pipelines with custom metrics

**Relevance**: The application needs a simplified version of these capabilities, focused on the transcript analysis domain.

#### 2. LLM Abstraction Layer

**Best practice**: Use an abstraction layer (e.g., LiteLLM, LangChain's LLM interface) to decouple prompt logic from specific LLM providers.

**Benefits**:
- Easy model switching
- Unified API for different providers
- Cost tracking across models
- Fallback/retry logic

**Risk**: Over-abstraction can introduce complexity. LangChain, while popular, is often criticized for being overly complex for simple use cases.

**Recommendation**: Use a lightweight abstraction — either LiteLLM or a custom thin wrapper over provider SDKs.

#### 3. Dynamic Schema / EAV Pattern

**Challenge**: Users can create custom output fields at runtime. This requires either:

| Approach | Pros | Cons |
|----------|------|------|
| **Entity-Attribute-Value (EAV)** | Maximum flexibility, no schema changes | Query complexity, poor performance at scale |
| **JSONB columns** (PostgreSQL) | Flexible, queryable, good performance | Less strict typing, harder to enforce constraints |
| **Dynamic table creation** | Clean schema per field | Migration complexity, maintenance burden |

**Recommendation**: **JSONB columns** in PostgreSQL — best balance of flexibility and queryability for this scale.

#### 4. Evaluation Pipeline Design

**Industry best practices** (from OpenAI Evals, LangSmith, RAGAS):

- **Evaluation set** = curated pairs of (input, expected_output)
- **Evaluation run** = executing a specific prompt version + model against the evaluation set
- **Metrics**: accuracy, precision, recall (for categorical), BLEU/ROUGE (for text), LLM-as-judge (for semantic)
- **Versioning**: Each evaluation run is immutable and linked to specific prompt version + model + evaluation set snapshot

**Critical insight**: The evaluation set itself should be versioned. If examples are added/removed, previous evaluation results should still reference the set version they were run against.

#### 5. Transcript Search

**Regex-based search** (as specified) is feasible for the expected scale (thousands to tens of thousands of transcripts).

**PostgreSQL capabilities**:
- `~` operator for regex matching
- `GIN` index on `tsvector` for full-text search (future upgrade path)
- `ILIKE` for simple pattern matching

**Future upgrade path**: If search quality becomes insufficient, add pgvector + embeddings for semantic search.

### Technology Comparison

| Component | Option A | Option B | Option C | Recommendation |
|-----------|----------|----------|----------|---------------|
| **Backend** | Python + FastAPI | Python + Django | Node.js + Express | **Python + FastAPI** — async support, fast, great for LLM integration |
| **Frontend** | React + TypeScript | Streamlit | Vue.js | **React + TypeScript** — most flexible, best for complex UIs |
| **Database** | PostgreSQL | Snowflake | SQLite | **PostgreSQL** — JSONB support, mature, great for this scale |
| **LLM Abstraction** | LiteLLM | LangChain | Custom wrapper | **LiteLLM** — lightweight, supports 100+ models |
| **Deployment** | Docker + Docker Compose | Kubernetes | Serverless | **Docker Compose** — appropriate for team size and complexity |

### Known Risks & Failure Modes

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| LLM API latency causing poor UX | High | Medium | Async processing + progress indicators |
| LLM API cost overrun during batch evaluation | Medium | High | Cost estimation before execution, rate limiting |
| Prompt injection via transcript content | Medium | Medium | Input sanitization, output validation |
| Evaluation set bias (not representative) | Medium | High | Minimum set size requirements, distribution analysis |
| Schema migration complexity with custom fields | Low | High | JSONB approach avoids schema migrations |
| Keboola integration breaking changes | Medium | Medium | Abstraction layer for data source |

---

## Phase 4: SYNTHESIS

**Dominant Agent**: Supervisor + Analysis Agent

### Problem Summary

Build a **web-based transcript analysis workbench** that enables quality assurance and analytics teams to:
1. Search and browse call recording transcripts
2. Define and version LLM prompts for information extraction
3. Execute prompts against transcripts with configurable LLM models
4. Persist results as structured data linked to recordings
5. Maintain evaluation sets and measure prompt+model accuracy
6. Visualize results through dashboards

### Constraints Summary

- **Search**: Regex-based initially (not semantic)
- **Language**: Slovak-language transcripts and UI context
- **Evaluation**: Minimum 100 pairs, mutable at any time
- **Versioning**: All prompts versioned; results traceable to prompt version + model
- **LLM**: Model-agnostic, future-proof architecture
- **Scale**: Thousands of transcripts, small team of users (5–20)
- **Custom fields**: Users can define arbitrary output fields at runtime

### Key Insights

1. **This is fundamentally a prompt engineering workbench** — not just a transcript viewer. The evaluation pipeline is the highest-value feature.
2. **Dynamic schema is the core technical challenge** — custom fields require flexible data modeling (JSONB).
3. **Traceability is non-negotiable** — every result must link to: transcript ID + prompt ID + prompt version + LLM model + timestamp.
4. **Evaluation set versioning is often overlooked** but critical — changing the eval set changes the meaning of accuracy scores.
5. **Batch vs. single execution** needs clear UX separation — running a prompt on one transcript is interactive; running on 100+ is a background job.

### Identified Risks (Prioritized)

1. 🔴 **LLM cost management** during batch evaluation (100+ transcripts × multiple prompts)
2. 🟡 **Evaluation methodology** — how to fairly compare results across prompt versions when the eval set changes
3. 🟡 **Keboola integration scope** — unclear how tightly coupled the app should be
4. 🟢 **Prompt injection** via transcript content — manageable with proper design
5. 🟢 **Regex search limitations** — acceptable for MVP, clear upgrade path exists

---

## Phase 5: SOLUTION DESIGN

**Dominant Agent**: Code Agent + Analysis Agent

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND (React + TypeScript)            │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │ Search   │ │Transcript│ │ Prompt   │ │Dashboard │          │
│  │ & Browse │ │ Viewer   │ │ Manager  │ │& Reports │          │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘          │
│       │             │            │             │                │
│  ┌────┴─────┐ ┌────┴─────┐ ┌────┴──────┐                      │
│  │Eval Set  │ │ Prompt   │ │ Custom    │                      │
│  │ Manager  │ │ Runner   │ │ Field Mgr │                      │
│  └────┬─────┘ └────┬─────┘ └────┬──────┘                      │
└───────┼────────────┼────────────┼──────────────────────────────┘
        │            │            │
        ▼            ▼            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (FastAPI)                         │
│                                                                 │
│  /api/transcripts    — Search, get, list                        │
│  /api/prompts        — CRUD, versioning                         │
│  /api/executions     — Run prompts, get results                 │
│  /api/evaluations    — Manage eval sets, run evaluations        │
│  /api/custom-fields  — CRUD custom output fields                │
│  /api/models         — List/configure LLM models                │
│  /api/dashboard      — Aggregated metrics & stats               │
└────────────┬────────────────────────────────────────────────────┘
             │
     ┌───────┴────────┐
     │                 │
     ▼                 ▼
┌─────────┐    ┌──────────────┐
│PostgreSQL│    │ LLM Gateway  │
│   DB     │    │ (LiteLLM)    │
│          │    │              │
│•Transcr. │    │ ┌──────────┐ │
│•Metadata │    │ │ OpenAI   │ │
│•Prompts  │    │ ├──────────┤ │
│•Versions │    │ │ Azure    │ │
│•Results  │    │ ├──────────┤ │
│ (JSONB)  │    │ │ Anthropic│ │
│•Eval Sets│    │ ├──────────┤ │
│•Eval Runs│    │ │ Local/   │ │
│•Custom   │    │ │ Other    │ │
│ Fields   │    │ └──────────┘ │
└─────────┘    └──────────────┘
```

### Key Components & Responsibilities

#### 1. Transcript Service
- **Responsibility**: Search, retrieve, and manage transcripts
- **Search capabilities**: Metadata filtering (date, agent, customer, type) + regex text search
- **Data source**: PostgreSQL (imported from Keboola Storage or external source)
- **Indexing**: GIN index on metadata JSONB, regex-capable text columns

#### 2. Prompt Management Service
- **Responsibility**: Full lifecycle management of prompts
- **Versioning**: Immutable versions — editing creates a new version (v1 → v2 → v3)
- **Structure**: Each prompt has: name, description/purpose, template text, output field definition, version number, created_at, author
- **Version comparison**: Ability to diff two versions of the same prompt

#### 3. Execution Engine
- **Responsibility**: Run prompts against transcripts via LLM
- **Single execution**: Synchronous — user selects transcript + prompt → sees result immediately
- **Batch execution**: Asynchronous — user selects prompt + filter criteria → background job processes matching transcripts
- **Result storage**: Results stored as JSONB with full provenance (prompt_id, prompt_version, model_id, timestamp)
- **LLM abstraction**: Via LiteLLM — supports model switching without code changes

#### 4. Custom Field Manager
- **Responsibility**: Allow users to define new output fields
- **Field types**: Text, Categorical (enum), Boolean, Numeric
- **Linkage**: Each custom field is associated with a prompt
- **Storage**: Field definitions in a dedicated table; field values in JSONB on the results table

#### 5. Evaluation Pipeline
- **Responsibility**: Manage evaluation sets and measure prompt+model accuracy
- **Eval set management**: Add/remove/edit examples at any time
- **Eval set versioning**: Snapshot the eval set state before each evaluation run
- **Evaluation run**: Execute prompt version + model against eval set snapshot → compute metrics
- **Metrics**: Accuracy, precision, recall (categorical); exact match ratio; optional LLM-as-judge scoring
- **History**: Full history of evaluation runs with comparable metrics

#### 6. Dashboard & Reporting
- **Responsibility**: Visualize results, trends, and evaluation metrics
- **Views**: 
  - Prompt accuracy over versions
  - Model comparison per prompt
  - Result distribution per custom field
  - Evaluation history timeline

### Data Model (Conceptual)

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  transcripts │     │     prompts      │     │   llm_models    │
├──────────────┤     ├──────────────────┤     ├─────────────────┤
│ id (PK)      │     │ id (PK)          │     │ id (PK)         │
│ external_id  │     │ name             │     │ name            │
│ text         │     │ description      │     │ provider        │
│ metadata     │     │ created_at       │     │ model_id        │
│  (JSONB)     │     │ created_by       │     │ config (JSONB)  │
│ created_at   │     │ is_active        │     │ is_active       │
└──────┬───────┘     └────────┬─────────┘     └────────┬────────┘
       │                      │                         │
       │             ┌────────┴─────────┐               │
       │             │ prompt_versions  │               │
       │             ├──────────────────┤               │
       │             │ id (PK)          │               │
       │             │ prompt_id (FK)   │               │
       │             │ version_number   │               │
       │             │ template_text    │               │
       │             │ output_schema    │               │
       │             │   (JSONB)        │               │
       │             │ created_at       │               │
       │             └────────┬─────────┘               │
       │                      │                         │
       │    ┌─────────────────┴──────────────────┐      │
       │    │                                    │      │
       ▼    ▼                                    ▼      ▼
┌──────────────────────┐            ┌──────────────────────────┐
│   execution_results  │            │    evaluation_runs       │
├──────────────────────┤            ├──────────────────────────┤
│ id (PK)              │            │ id (PK)                  │
│ transcript_id (FK)   │            │ prompt_version_id (FK)   │
│ prompt_version_id(FK)│            │ model_id (FK)            │
│ model_id (FK)        │            │ eval_set_snapshot_id(FK) │
│ result_data (JSONB)  │            │ metrics (JSONB)          │
│ execution_time_ms    │            │ started_at               │
│ token_count          │            │ completed_at             │
│ cost_estimate        │            │ status                   │
│ created_at           │            └──────────┬───────────────┘
└──────────────────────┘                       │
                                               │
┌──────────────────────┐            ┌──────────┴───────────────┐
│   eval_set_items     │            │  eval_set_snapshots      │
├──────────────────────┤            ├──────────────────────────┤
│ id (PK)              │            │ id (PK)                  │
│ eval_set_id (FK)     │            │ eval_set_id (FK)         │
│ transcript_id (FK)   │            │ snapshot_data (JSONB)    │
│ expected_output      │            │ item_count               │
│   (JSONB)            │            │ created_at               │
│ is_active            │            └──────────────────────────┘
│ created_at           │
│ updated_at           │     ┌──────────────────────┐
└──────────────────────┘     │   custom_fields      │
                             ├──────────────────────┤
┌──────────────────────┐     │ id (PK)              │
│   eval_sets          │     │ prompt_id (FK)       │
├──────────────────────┤     │ field_name           │
│ id (PK)              │     │ field_type           │
│ name                 │     │ field_options (JSONB) │
│ description          │     │ is_active            │
│ created_at           │     │ created_at           │
│ updated_at           │     └──────────────────────┘
└──────────────────────┘
```

### Data Flow — Single Prompt Execution

```
Step 1: User searches transcripts by metadata/regex
         → Frontend sends GET /api/transcripts?metadata_filter=...&text_regex=...
         → Backend queries PostgreSQL with filters
         → Returns matching transcript list

Step 2: User opens a transcript
         → Frontend sends GET /api/transcripts/{id}
         → Backend returns full transcript text + metadata + previous results

Step 3: User selects a prompt and model
         → Frontend loads prompt list from GET /api/prompts
         → Frontend loads model list from GET /api/models
         → User selects prompt version + model

Step 4: User executes prompt
         → Frontend sends POST /api/executions
           { transcript_id, prompt_version_id, model_id }
         → Backend constructs full prompt (template + transcript text)
         → Backend calls LLM via LiteLLM
         → Backend parses and validates response against output schema
         → Backend stores result in execution_results table
         → Returns result to frontend

Step 5: Result is displayed and persisted
         → Frontend shows formatted result
         → Result is now queryable as an attribute of the transcript
```

### Data Flow — Evaluation Run

```
Step 1: User selects prompt version + model for evaluation
         → Frontend sends POST /api/evaluations/run
           { prompt_version_id, model_id, eval_set_id }

Step 2: Backend snapshots the evaluation set
         → Copies current eval_set_items to eval_set_snapshots
         → Creates evaluation_run record with status "running"

Step 3: Backend processes each eval set item (async)
         → For each (transcript_id, expected_output) pair:
           → Execute prompt against transcript via LLM
           → Compare actual output to expected output
           → Record individual result

Step 4: Backend computes metrics
         → Accuracy = correct / total
         → Per-field metrics (if multiple output fields)
         → Stores aggregated metrics in evaluation_run.metrics

Step 5: User views evaluation results
         → Frontend fetches GET /api/evaluations/{run_id}
         → Shows overall metrics + individual results + comparison to previous runs
```

### Interfaces Between Components

| Interface | Protocol | Format | Notes |
|-----------|----------|--------|-------|
| Frontend ↔ API | REST (HTTP) | JSON | Standard REST with pagination |
| API ↔ Database | SQL | PostgreSQL protocol | Via async ORM (SQLAlchemy + asyncpg) |
| API ↔ LLM | HTTP | Provider-specific (via LiteLLM) | Async calls with timeout handling |
| Keboola ↔ Database | SQL/CSV | Keboola Storage API | Data import/export pipeline |

### Trade-offs & Justifications

| Decision | Alternative | Why This Choice |
|----------|------------|----------------|
| **JSONB for results** vs. separate columns | Dynamic table creation | JSONB provides flexibility for custom fields without schema migrations; queryable via PostgreSQL operators; sufficient performance at this scale |
| **LiteLLM** vs. LangChain | LangChain provides more features | LiteLLM is lighter, simpler, and sufficient for the use case; LangChain would add unnecessary complexity |
| **FastAPI** vs. Django | Django has built-in admin | FastAPI's async support is better for LLM calls; type hints provide auto-documentation; Django's ORM would add overhead |
| **React** vs. Streamlit | Streamlit is faster to prototype | React provides the flexibility needed for complex UIs (prompt editor, eval set management, dashboards); Streamlit would become limiting |
| **PostgreSQL** vs. Snowflake | Snowflake is Keboola-native | PostgreSQL provides better JSONB support, lower latency for interactive queries, and simpler operational model; Snowflake can be used as a secondary analytics store |
| **Eval set snapshots** vs. no versioning | Simpler implementation | Without snapshots, adding/removing examples invalidates historical accuracy comparisons; snapshots ensure evaluation integrity |
| **Regex search** vs. full-text/semantic | Better search quality | Regex is explicitly requested for MVP; full-text search is a clear upgrade path (add `tsvector` + GIN index); semantic search can be added later with pgvector |

---

## Phase 6: IMPLEMENTATION PLAN

**Dominant Agent**: Code Agent + Supervisor

### Suggested Tech Stack

| Layer | Technology | Version | Reasoning |
|-------|-----------|---------|-----------|
| **Backend** | Python 3.11+ | 3.11+ | Async support, LLM ecosystem |
| **Web Framework** | FastAPI | 0.100+ | Async, auto-docs, type safety |
| **ORM** | SQLAlchemy 2.0 + asyncpg | 2.0+ | Async PostgreSQL support |
| **Database** | PostgreSQL 15+ | 15+ | JSONB, regex, GIN indexes |
| **LLM Abstraction** | LiteLLM | latest | Multi-provider support |
| **Frontend** | React 18 + TypeScript | 18+ | Component-based, typed |
| **UI Library** | Shadcn/UI or Ant Design | latest | Rich component library |
| **State Management** | TanStack Query (React Query) | v5 | Server state management |
| **Charts** | Recharts or Apache ECharts | latest | Dashboard visualizations |
| **Task Queue** | Celery + Redis | 5.3+ | Background job processing for batch operations |
| **Containerization** | Docker + Docker Compose | latest | Development and deployment |
| **Migrations** | Alembic | latest | Database schema management |

### MVP Definition (Smallest Viable Version)

The MVP should demonstrate the core value loop: **Search → View → Prompt → Result → Evaluate**

**MVP Includes:**
- ✅ Transcript list view with metadata filtering
- ✅ Transcript detail view (full text)
- ✅ Regex text search across transcripts
- ✅ 3 pre-built prompts (sentiment, objections, client need)
- ✅ Single transcript prompt execution with one LLM model (e.g., GPT-4o)
- ✅ Result storage and display on transcript detail
- ✅ Basic prompt CRUD (create, edit, view)
- ✅ Prompt versioning (automatic version increment on edit)
- ✅ Evaluation set CRUD (add/remove/edit pairs)
- ✅ Basic evaluation run (accuracy metric)
- ✅ Simple results dashboard (prompt accuracy per version)

**MVP Excludes (Post-MVP):**
- ❌ Batch prompt execution across multiple transcripts
- ❌ Multiple LLM model support (MVP uses one model)
- ❌ Advanced evaluation metrics (precision, recall, F1)
- ❌ Prompt version diff/comparison view
- ❌ Cost tracking per execution
- ❌ Custom field management UI (use hardcoded fields in MVP)
- ❌ Keboola Storage integration (use direct DB import)
- ❌ User authentication/authorization
- ❌ Real-time dashboard updates

### Step-by-Step Roadmap

#### Phase A: Foundation (Week 1–2)

| Step | Task | Deliverable |
|------|------|-------------|
| A1 | Project scaffolding — backend (FastAPI + Poetry) | Working `/health` endpoint |
| A2 | Project scaffolding — frontend (React + Vite + TypeScript) | Working development server |
| A3 | Docker Compose setup (API + DB + Frontend) | `docker-compose up` starts full stack |
| A4 | Database schema design + Alembic migrations | All tables created |
| A5 | Seed data — import sample transcripts + metadata | 50+ transcripts in DB |
| A6 | API scaffolding — all route groups with stub endpoints | OpenAPI docs at `/docs` |

#### Phase B: Core Transcript Features (Week 2–3)

| Step | Task | Deliverable |
|------|------|-------------|
| B1 | Transcript list API with metadata filtering | `GET /api/transcripts` with query params |
| B2 | Transcript detail API | `GET /api/transcripts/{id}` |
| B3 | Regex text search in transcripts | `GET /api/transcripts?text_regex=...` |
| B4 | Frontend — transcript list view with search/filter | Functional search UI |
| B5 | Frontend — transcript detail view | Readable transcript display |

#### Phase C: Prompt Management (Week 3–4)

| Step | Task | Deliverable |
|------|------|-------------|
| C1 | Prompt CRUD API | `POST/GET/PUT /api/prompts` |
| C2 | Prompt versioning logic | Auto-increment on edit, version history |
| C3 | Frontend — prompt list & editor | CRUD UI for prompts |
| C4 | Frontend — prompt version history view | Version list with template text |
| C5 | Custom field definition API | `POST/GET /api/custom-fields` |
| C6 | Frontend — custom field definition UI | Add/edit field definitions |

#### Phase D: Execution Engine (Week 4–5)

| Step | Task | Deliverable |
|------|------|-------------|
| D1 | LiteLLM integration + model config | LLM calls working |
| D2 | Single execution API | `POST /api/executions` |
| D3 | Result storage with full provenance | Results in DB with JSONB |
| D4 | Frontend — prompt runner on transcript detail | Run button + result display |
| D5 | Frontend — execution history on transcript | List of past results per transcript |
| D6 | LLM model selection API | `GET /api/models`, model parameter in execution |

#### Phase E: Evaluation Pipeline (Week 5–7)

| Step | Task | Deliverable |
|------|------|-------------|
| E1 | Evaluation set CRUD API | `POST/GET/PUT/DELETE /api/eval-sets` |
| E2 | Eval set item management | Add/remove/edit individual pairs |
| E3 | Eval set snapshot mechanism | Snapshot creation before evaluation |
| E4 | Evaluation run execution (async via Celery) | Background evaluation processing |
| E5 | Accuracy metric computation | Exact match + categorical metrics |
| E6 | Frontend — eval set management UI | CRUD for eval sets and items |
| E7 | Frontend — evaluation run trigger + progress | Start evaluation, see progress |
| E8 | Frontend — evaluation results view | Metrics display + individual results |

#### Phase F: Dashboard & Polish (Week 7–8)

| Step | Task | Deliverable |
|------|------|-------------|
| F1 | Dashboard API — aggregated metrics | `GET /api/dashboard/*` endpoints |
| F2 | Frontend — prompt accuracy over versions chart | Line chart |
| F3 | Frontend — model comparison view | Comparison table/chart |
| F4 | Frontend — result distribution charts | Bar/pie charts per field |
| F5 | Error handling & input validation polish | Consistent error responses |
| F6 | API documentation cleanup | Complete OpenAPI spec |

#### Phase G: Post-MVP Enhancements (Week 8+)

| Step | Task | Priority |
|------|------|----------|
| G1 | Batch prompt execution with Celery | High |
| G2 | Multi-model support + cost tracking | High |
| G3 | Advanced eval metrics (precision, recall, F1) | Medium |
| G4 | Prompt version diff viewer | Medium |
| G5 | Keboola Storage integration | Medium |
| G6 | User authentication (OAuth/JWT) | Medium |
| G7 | Semantic search upgrade (pgvector) | Low |
| G8 | Export results to CSV/Excel | Low |
| G9 | Webhook/notification on batch completion | Low |

### Task Breakdown (Ready for Coding Agents)

Each task above is sized for **1–3 days of work** by a single developer. Tasks within the same phase can be partially parallelized (backend tasks can start before frontend tasks of the same phase, but frontend needs API endpoints).

**Parallelization opportunities:**
- A1 + A2 can run in parallel (backend + frontend scaffolding)
- Within each phase, backend and frontend tasks can overlap
- Phase E (Evaluation) is the largest — consider 2 developers

**Dependencies:**
```
A (Foundation) → B (Transcripts) → C (Prompts) → D (Execution) → E (Evaluation) → F (Dashboard)
                                                                    ↑
                                                          C (Prompts) ──┘ (Eval needs prompts)
```

### Risks & Mitigation Strategies

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| **LLM API costs during development** | High | Medium | Use GPT-4o-mini for development; set budget alerts; cache responses during testing |
| **LLM response format inconsistency** | High | Medium | Structured output (JSON mode); retry with re-prompt on parse failure; output schema validation |
| **JSONB query performance at scale** | Low | High | GIN indexes on JSONB columns; monitor query plans; optimize hot paths |
| **Evaluation set management complexity** | Medium | Medium | Start with simple exact match; add complexity incrementally |
| **Keboola integration scope creep** | Medium | High | Defer to post-MVP; use direct DB import initially |
| **Slovak language handling in LLM** | Medium | Medium | Test with Slovak prompts early; ensure model supports SK; consider prompt language (EN template with SK content) |
| **Frontend complexity for prompt editor** | Medium | Medium | Use Monaco Editor (VS Code editor component) for prompt template editing |
| **Background job reliability** | Medium | Medium | Celery with Redis broker; dead letter queue; job status tracking in DB |

### Directory Structure (Recommended)

```
keboola_transcripts_app/
├── docker-compose.yml
├── README.md
├── PROJECT_DESIGN.md
│
├── backend/
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── alembic/
│   │   ├── alembic.ini
│   │   └── versions/
│   ├── app/
│   │   ├── main.py              # FastAPI app entry
│   │   ├── config.py            # Settings/env config
│   │   ├── database.py          # DB connection
│   │   ├── models/              # SQLAlchemy models
│   │   │   ├── transcript.py
│   │   │   ├── prompt.py
│   │   │   ├── execution.py
│   │   │   ├── evaluation.py
│   │   │   └── custom_field.py
│   │   ├── schemas/             # Pydantic schemas
│   │   │   ├── transcript.py
│   │   │   ├── prompt.py
│   │   │   ├── execution.py
│   │   │   └── evaluation.py
│   │   ├── api/                 # Route handlers
│   │   │   ├── transcripts.py
│   │   │   ├── prompts.py
│   │   │   ├── executions.py
│   │   │   ├── evaluations.py
│   │   │   ├── custom_fields.py
│   │   │   ├── models.py
│   │   │   └── dashboard.py
│   │   ├── services/            # Business logic
│   │   │   ├── transcript_service.py
│   │   │   ├── prompt_service.py
│   │   │   ├── execution_service.py
│   │   │   ├── evaluation_service.py
│   │   │   └── llm_service.py
│   │   └── tasks/               # Celery tasks
│   │       ├── batch_execution.py
│   │       └── evaluation_runner.py
│   └── tests/
│       ├── conftest.py
│       ├── test_transcripts.py
│       ├── test_prompts.py
│       ├── test_executions.py
│       └── test_evaluations.py
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── api/                 # API client
│   │   │   └── client.ts
│   │   ├── components/          # Reusable components
│   │   │   ├── TranscriptList.tsx
│   │   │   ├── TranscriptDetail.tsx
│   │   │   ├── PromptEditor.tsx
│   │   │   ├── PromptRunner.tsx
│   │   │   ├── EvalSetManager.tsx
│   │   │   ├── EvalRunResults.tsx
│   │   │   ├── CustomFieldEditor.tsx
│   │   │   └── Dashboard.tsx
│   │   ├── pages/               # Page-level components
│   │   │   ├── SearchPage.tsx
│   │   │   ├── TranscriptPage.tsx
│   │   │   ├── PromptsPage.tsx
│   │   │   ├── EvaluationPage.tsx
│   │   │   └── DashboardPage.tsx
│   │   ├── hooks/               # Custom React hooks
│   │   └── types/               # TypeScript type definitions
│   └── tests/
│
└── scripts/
    ├── seed_data.py             # Import sample transcripts
    └── run_migration.sh         # DB migration helper
```

---

## Appendix: Open Questions for Stakeholder

The following questions should be answered before implementation begins:

1. **Data source confirmation**: Where exactly are transcripts stored today? What is the import mechanism?
2. **Authentication requirements**: Is multi-user access needed? What auth provider?
3. **Keboola integration depth**: Is this a Keboola component or a standalone app that happens to use Keboola data?
4. **LLM provider**: Which LLM provider(s) are already contracted/available?
5. **Deployment target**: Where will this run? Cloud VM, Keboola infrastructure, customer premises?
6. **Budget for LLM API calls**: Is there a monthly budget for LLM API usage?
7. **Compliance**: Any data protection requirements (GDPR, call recording regulations)?
8. **Timeline**: What is the expected delivery timeline for MVP?
9. **Team size**: How many developers are available?
10. **Existing tooling**: Are there any existing tools or databases this needs to integrate with beyond Keboola?

---

*This document was generated following a structured multi-agent analysis process. All assumptions are clearly labeled and should be validated with stakeholders before implementation begins.*
