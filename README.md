# Keboola Transcripts App

A web-based transcript analysis workbench that enables quality assurance and analytics teams to search, analyze, and evaluate call recording transcripts using configurable LLM prompts.

## Project Status

🟡 **Design Phase** — See [PROJECT_DESIGN.md](./PROJECT_DESIGN.md) for the full multi-phase analysis and design document.

## Key Features (Planned)

- **Transcript Search** — Search by metadata and regex text patterns
- **Prompt Management** — Create, edit, and version LLM prompts for information extraction
- **Prompt Execution** — Run prompts against transcripts with configurable LLM models
- **Custom Fields** — Define custom output fields for prompt results
- **Evaluation Pipeline** — Maintain evaluation sets and measure prompt+model accuracy
- **Dashboard** — Visualize results and evaluation metrics

## Tech Stack (Planned)

| Layer | Technology |
|-------|-----------|
| Backend | Python + FastAPI |
| Frontend | React + TypeScript |
| Database | PostgreSQL (JSONB) |
| LLM | LiteLLM (multi-provider) |
| Task Queue | Celery + Redis |
| Deployment | Docker Compose |