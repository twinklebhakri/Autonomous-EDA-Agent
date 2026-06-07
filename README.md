# Autonomous EDA Agent

An agentic exploratory-data-analysis system that takes any CSV and autonomously profiles it, visualizes key findings, and cleans the data — then returns a findings report. The agent decides *what* to investigate and *what* to clean on its own; it is not a fixed pipeline.

Built with LangChain (tool-calling agent), GPT-4o, and FastAPI, and containerized with Docker.

## What it does

Given an arbitrary dataset, the agent runs an autonomous loop:

1. **Profiles** the dataset — infers each column's role (numeric, categorical, datetime, id-like, constant) and flags issues (high nulls, likely-miscoded categoricals, etc.).
2. **Investigates** — drills into problem columns, checks duplicates, and tests relationships between the target and its strongest predictors.
3. **Visualizes** — chooses and saves charts (histograms, bar charts, box plots, scatter plots, correlation heatmaps) that support its findings.
4. **Cleans** — handles missing values through guard-railed tools that validate every operation and log every change for auditability.
5. **Reports** — writes a findings-first summary citing concrete numbers and referencing the charts it produced.

Different datasets produce different investigations and different cleaning decisions, because the model chooses each step based on what it observes.

## Architecture

The system is built on a simple, reusable agent pattern:

- **Read tools** — `profile_dataset`, `inspect_column`, `analyze_relationship`, `check_duplicates`, `plot`. These observe and report; they never mutate data.
- **Write tool** — `handle_missing` mutates a *working copy* of the data, never the original. It enforces four guardrails before acting (column exists, operation is type-valid, there is work to do, strategy is on the allowed menu) and logs every transformation.
- **Agent loop** — a LangChain `create_agent` tool-calling loop. A system prompt defines the process; the model selects each tool call.
- **Isolation** — the agent is a class (`EDAAgent`) bound to a single dataset instance. No global state, so multiple datasets can be analyzed concurrently (important for the API).
- **Service** — a FastAPI app exposes a `POST /analyze` endpoint that accepts a CSV upload and returns the summary, cleaning log, and post-cleaning shape as JSON.

## Design notes

A few deliberate engineering choices:

- **Facts in code, judgment in the LLM.** All statistics and chart rendering are deterministic Python. The model only decides *what* to compute and *what* to clean — it never generates the analysis math or plotting code itself. This keeps output reliable and the agent debuggable.
- **Guarded write-tools.** Reading data is safe; mutating it is not. The cleaning tool refuses invalid operations and returns an explanatory message the agent can act on, rather than crashing — the "refuse and recover" pattern needed for any agent that touches real data.
- **Audit trail.** Every cleaning action is logged with the specific value used (e.g. "Filled 177 nulls in 'age' with median 28.0"), so results are verifiable rather than opaque.

## Running it

### Locally (FastAPI)

```bash
pip install -r requirements.txt
export OPENAI_API_KEY="your-key"      # Windows: set OPENAI_API_KEY=your-key
uvicorn app:app --reload
```

Then open `http://127.0.0.1:8000/docs`, upload a CSV to `/analyze`, and (optionally) name the target column.

### With Docker

```bash
docker build -t eda-agent .
docker run -p 8000:8000 -e OPENAI_API_KEY="your-key" eda-agent
```

## How it was built

I built this agent twice on purpose. First as a raw loop with the OpenAI SDK — manually serializing state into a prompt, parsing the model's chosen action, executing it, and feeding the result back. Then I rebuilt it in LangChain. Doing the raw version first meant I understood exactly what the framework abstracts away (the decide–parse–execute loop) and what it adds (tool binding, schema generation from type hints), rather than treating `create_agent` as a black box.

## Stack

LangChain · GPT-4o · FastAPI · Docker · pandas · matplotlib / seaborn

## Possible extensions

- Add a `compare_before_after` tool so the agent verifies its own cleaning.
- Broaden cleaning beyond missing-value handling (outlier treatment, type casting, deduplication).
- Cloud deployment (e.g. Render / Railway) for a live endpoint.# Autonomous-EDA-Agent
