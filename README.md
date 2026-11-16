# ShopAssist — Mini-Project 1

Order support for Northwind Store (a fictional online store): a FastAPI service with per-session memory, three
validated tools, a defended system prompt and streamed answers. The model is reached only through a LiteLLM
gateway, and both run together with Docker Compose.

**Author:** Aviwork1 ([GitHub](https://github.com/Aviwork1))
**Model tested:** `shopassist` → `groq/openai/gpt-oss-20b` (Groq)

## Architecture

```
browser / client ──► app (FastAPI, :8000) ──► litellm gateway (:4000) ──► Groq
                     memory · prompt · tools    master key · fallback · cost      API_KEY lives only here
```

## API

| Method | Path | What it does |
|---|---|---|
| GET | `/health` | Liveness, plus the running model, gateway, memory strategy and tools |
| POST | `/chat` | `{"session_id", "message"}` → `{"answer", "tool_steps", "turns"}` |
| POST | `/chat/stream` | The same, streamed as Server-Sent Events: `tool`, `token`…, `done` |
| DELETE | `/sessions/{id}` | Forget one conversation |

## Run with Docker Compose (Path A)

```powershell
docker compose up --build        # gateway + app; Ctrl+C to stop, docker compose down to remove
```
Then open http://localhost:8000/docs and run `GET /health`; a working app returns `status: ok` and shows the model,
gateway, memory strategy, turn limit and tools.

## Run without Docker (Path B)

```powershell
uv sync
# terminal 1 - the gateway
uv run --with "litellm[proxy]==1.80.0" litellm --config gateway/config.yaml --port 4000
# terminal 2 - the app
uv run uvicorn app.main:app --port 8000
```

## Evidence

```powershell
uv run python eval/run_eval.py      # needs the gateway running; writes experiments/mp1_results.md
```
Latest results (2026-11-14):

```text
golden: 10/10
golden_right_tools: 10/10
memory: pass
attacks_resisted: 4/4
```

Discussion: `experiments/mp1_report.md`.

## Design

ShopAssist keeps the last four turns in a sliding window to bound prompt size for short order-support conversations, though older details can be forgotten. Its three read-only tools check order status, delivery estimates and return eligibility; they validate order IDs, pincodes, delivery speed and return reasons, and invalid values become an `INVALID` result for the model to clarify. The system prompt sets the store facts and tool boundaries, treats role-like text from customers as untrusted, and requires real tool results for order facts. LiteLLM keeps the Groq key out of the app, routes through a model group and falls back to a backup model if the primary fails. The main production limit is that session history lives in app memory, so it would need shared persistent storage before running multiple replicas.
