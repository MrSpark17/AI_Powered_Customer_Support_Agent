# AI-Powered Customer Support Agent

[![Python 3.12+](https://img.shields.io/badge/Python-3.12%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/API-FastAPI-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![Streamlit](https://img.shields.io/badge/UI-Streamlit-FF4B4B?logo=streamlit&logoColor=white)](https://streamlit.io/)

An AI copilot that helps support agents turn customer tickets into grounded,
personalized response drafts. It combines a knowledge base, customer-resolution
memory, controlled support tools, and human approval in one workflow.

**Live dashboard:** [http://3.106.56.13:8501](http://3.106.56.13:8501)

> The live deployment is a demonstration environment. Do not enter real customer
> personal data or production credentials.

## What it does

- Creates and manages customer-support tickets.
- Ingests Markdown knowledge-base files and retrieves relevant policy content.
- Searches prior accepted resolutions for both the individual customer and their
  company.
- Uses an LLM agent to draft concise, customer-ready support emails.
- Lets the agent call controlled support tools for plan/SLA and open-ticket data.
- Shows the evidence, knowledge sources, memory hits, and tool calls behind each
  draft.
- Requires a human to accept, edit, or discard every draft before it is treated
  as an accepted resolution.
- Optionally sends privacy-redacted traces to LangSmith and uses OpenAI
  moderation guardrails before and after generation.

## How it works

```text
Streamlit dashboard
        |
        v
FastAPI ticket API  --->  SQLite tickets, customers, drafts
        |
        v
Support Copilot
  |        |        |
  |        |        +--> controlled support tools
  |        +-----------> Mem0 customer/company memory
  +--------------------> Chroma knowledge-base retrieval
        |
        v
OpenAI / LangChain agent
        |
        v
Guardrail check --> human review --> accepted resolution is saved as memory
```

### Draft-generation flow

1. A support agent creates a ticket in the dashboard.
2. The API stores the customer and ticket in SQLite.
3. The copilot checks the ticket with optional safety guardrails.
4. It retrieves relevant knowledge-base chunks from Chroma and prior support
   resolutions from Mem0.
5. The LangChain agent can call allowlisted tools for account-plan and ticket-load
   information.
6. The model creates a customer-facing email draft with the retrieved context.
7. Output guardrails check the draft. Flagged or suspicious content is withheld
   for human review.
8. The support agent accepts, edits, or discards the draft. Accepted responses
   are added to memory for future personalization.

## Technology stack

| Area | Technologies |
| --- | --- |
| Frontend | Streamlit |
| Backend API | FastAPI, Uvicorn, Pydantic Settings |
| Agent orchestration | LangChain `create_agent`, LangGraph checkpointing |
| LLM and embeddings | OpenAI via `langchain-openai` |
| Knowledge retrieval | ChromaDB, OpenAI embeddings, LangChain text splitters |
| Long-term memory | Mem0, scoped by customer and company |
| Data store | SQLite |
| Safety | OpenAI Moderation API, prompt-injection detection, human review |
| Observability | LangSmith with email/phone/card redaction |
| Packaging and deployment | Docker, Docker Compose, GitHub Actions, AWS EC2 |
| Testing and dependency management | Pytest, uv |

## Project structure

```text
.
├── app.py                         # Streamlit dashboard
├── main.py                        # FastAPI server entry point
├── customer_support_agent/
│   ├── api/                       # FastAPI routes and dependencies
│   ├── core/                      # Configuration and startup helpers
│   ├── integrations/
│   │   ├── memory/                # Mem0 customer memory
│   │   ├── observability/         # LangSmith tracing and PII redaction
│   │   ├── rag/                   # Chroma knowledge base
│   │   ├── safety/                # Moderation and injection guardrails
│   │   └── tools/                 # Allowlisted support tools
│   ├── repositories/sqlite/       # Ticket, customer, and draft persistence
│   ├── services/                  # Copilot and draft workflows
│   └── tests/                     # API, safety, and observability tests
├── knowledge_base/                # Markdown support policies
├── docker-compose.yml             # API + dashboard deployment
└── .github/workflows/deploy-ec2.yml
```

## Run locally

### 1. Configure environment variables

Copy `.env.example` to `.env` and add your keys. Never commit `.env`.

```env
OPENAI_API_KEY=your_openai_key

# Optional observability
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=your_langsmith_key
LANGSMITH_PROJECT=support-copilot

# Optional safety controls
GUARDRAILS_ENABLED=true
GUARDRAILS_FAIL_OPEN=false
```

### 2. Start with Docker Compose

```bash
docker compose up --build
```

Open [http://localhost:8501](http://localhost:8501). The API health endpoint is
available at [http://localhost:8000/health](http://localhost:8000/health).

### 3. Ingest the knowledge base

From the dashboard sidebar, select **Ingest Knowledge Base**. The application
indexes the Markdown files in `knowledge_base/` into ChromaDB.

## Development

```bash
# Install dependencies
uv sync --dev

# Run the API
uv run python main.py

# In another terminal, run the dashboard
uv run streamlit run app.py

# Run tests
uv run pytest -q
```

## LangSmith and guardrails

LangSmith tracing is opt-in. When enabled with `LANGSMITH_TRACING=true` and a
`LANGSMITH_API_KEY`, the application creates traces for draft generation. Common
email addresses, phone numbers, and payment-card numbers are redacted before
trace input/output is sent.

Guardrails are also opt-in. When `GUARDRAILS_ENABLED=true`, the app checks ticket
input and generated text with OpenAI's moderation model. It also detects common
prompt-injection patterns. A flagged check, suspicious prompt, or failed safety
request is held for human review by default (`GUARDRAILS_FAIL_OPEN=false`).

## Deployment

The repository includes a GitHub Actions workflow that tests the project and
deploys the `master` branch to AWS EC2 using Docker Compose.

The deployment runs two containers:

- `api` on port `8000`
- `dashboard` on port `8501`

The dashboard calls the API over Docker's internal service network, and waits for
the API health check before starting.

For production, keep all credentials in GitHub Actions secrets (the project uses
an `EC2_ENV_FILE` secret), restrict EC2 security-group access, rotate keys, and
use a managed secret store rather than committing credentials.

## Interview highlights

This project demonstrates more than prompt-based chat:

- Grounded RAG answers with an inspectable evidence trail.
- Personalized support using customer and company-level memory.
- Tool-using agent behavior with bounded recursion and controlled tool access.
- Human-in-the-loop decision making for AI-generated drafts.
- Practical safety and observability controls for a customer-data workflow.
- Containerized deployment with automated CI/CD on AWS.

## License

Add a license before using this project outside personal or educational work.
