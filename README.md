# Deploying an LLM App: From Console Script to Containerized API on AWS

> Built on the [Hyperskill starter project](https://github.com/hyperskill-content/AI-Engineer-Deploy) as part of the **AI Engineer Intensive Bootcamp (2026)**.
> Part 3 of the series, following [LLM-evals](https://github.com/DMLearn/LLM-evals) and [llm-evals-2-preparing-for-prod](https://github.com/DMLearn/llm-evals-2-preparing-for-prod). The task structure was provided; the API, infrastructure migration, container build and deployment are my implementation.

The production-hardened smartphone chatbot from part 2 (RAG, guardrails, Redis memory, prompt management) turned into a **stateless REST service**, moved to **managed cloud infrastructure**, packaged as a **hardened multi-stage Docker image** and **deployed to AWS EC2**.

---

## What I implemented

- **REST API with FastAPI** – `POST /ask` (user input, user ID, session ID → response) and `GET /health`. The interactive console loop was refactored into a request/response flow with session state kept in Redis, so the service itself stays stateless.
- **Migration to managed services** – local Docker dependencies replaced by cloud services: Qdrant Cloud (vector store), managed Redis (chat history), Langfuse Cloud (tracing and prompt management). All connections configured via environment variables.
- **Multi-stage Docker build**
  - builder stage compiles dependencies into a virtual environment, runtime stage contains only what is needed to run
  - runs as **non-root user**
  - built-in **health check** against `/health`
  - served by **Gunicorn with Uvicorn workers** (2 workers, preloaded app)
  - secrets kept out of the image and the repo via `.dockerignore` / `.gitignore`
- **Deployment to AWS EC2** – container running on an EC2 instance, reachable via HTTP.
- **Pinned dependencies** – exact versions in `requirements.txt` for reproducible builds.
- **API test scenarios** – [`tests.http`](tests.http) covers valid requests, inputs that must be blocked by the guardrails (returns, order tracking, off-topic) and a multi-turn conversation that checks session memory.

## Architecture

```
Client ──HTTP──► AWS EC2
                  └─ Docker container (non-root, health check)
                       └─ Gunicorn + Uvicorn workers
                            └─ FastAPI  POST /ask · GET /health
                                 │
                                 ├─ NeMo Guardrails (input rail)
                                 ├─ LangChain chains + tool calling
                                 │
                                 ├──► Qdrant Cloud       (product vector search)
                                 ├──► Managed Redis      (session history, TTL)
                                 ├──► Langfuse Cloud     (tracing, prompt versions)
                                 └──► OpenAI API         (LLM + embeddings)
```

## Tech stack

Python · FastAPI · Gunicorn/Uvicorn · Docker (multi-stage) · AWS EC2 · LangChain · NeMo Guardrails · Qdrant Cloud · Redis · Langfuse Cloud · OpenAI API

## Run it

```bash
# Environment (.env)
OPENAI_MODEL=...            OPENAI_BASE_URL=...      OPENAI_API_KEY=...
LANGFUSE_PUBLIC_KEY=...     LANGFUSE_SECRET_KEY=...  LANGFUSE_HOST=...
QDRANT_URL=...              QDRANT_API_KEY=...
REDIS_CONN_STRING=...

# Local
pip install -r requirements.txt
uvicorn main:app --port 8000

# Docker
docker build -t smartphone-assistant .
docker run --env-file .env -p 8000:8000 smartphone-assistant
```

Then run the scenarios in `tests.http` (JetBrains / VS Code HTTP client) or:

```bash
curl -X POST http://localhost:8000/ask -H "Content-Type: application/json" \
  -d '{"user_input": "Compare the Samsung Galaxy S24 with the Google Pixel 8.", "user_id": "u1", "session_id": "s1"}'
```

## Known limitations & next steps

This is a learning deployment, not a hardened production service. Before real users, I would add:

- **Authentication and rate limiting** on `/ask` – currently any client can trigger paid LLM calls
- **HTTPS** via a reverse proxy or load balancer
- **CI/CD** – build, test and push the image automatically (e.g. GitHub Actions → Amazon ECR)
- **Automated tests** – turn the `tests.http` scenarios into a pytest suite that runs in the pipeline
- **Build chains once at startup** instead of on every request

---

<details>
<summary>Original task description (Hyperskill)</summary>

The project is structured into four tasks, see the [tasks](./tasks) folder:

1. **Create an API endpoint** for the LLM application
2. **Migrate to managed services** (vector store, Redis, Langfuse)
3. **Containerize** the application with Docker
4. **Deploy** to AWS EC2

</details>
