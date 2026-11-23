# AGENTS.md

## Cursor Cloud specific instructions

Odysseus is a single-process FastAPI "self-hosted AI workspace" (Python 3.11+).
The entry point is `app.py`; routes live in `routes/`, business logic in `src/`,
core infra (auth, db, middleware) in `core/`, and the front-end is static files
in `static/`. Standard setup/run/test commands are documented in `README.md`
(Option 2: Manual install) and `CONTRIBUTING.md` — prefer those as the source of
truth. Notes below are only the non-obvious, Cloud-specific bits.

### Environment / running
- A Python venv is set up at `./venv` (the update script keeps its deps fresh).
  Run everything through it, e.g. `./venv/bin/python -m uvicorn app:app --host 0.0.0.0 --port 7000`.
- First-time only (already done; `data/` persists in the VM snapshot):
  `./venv/bin/python setup.py` creates `data/`, initializes the SQLite db, and
  creates an `admin` user. It prints a generated password unless
  `ODYSSEUS_ADMIN_PASSWORD` is set. This repo was bootstrapped with
  `admin` / `OdysseusDev123!`. `setup.py` is idempotent (skips existing).
- The dev server has no auto-reload by default; pass `--reload` to uvicorn if you
  want hot reload while editing.

### Auth / hitting the app
- Web UI: `http://localhost:7000`. Login API: `POST /api/auth/login` with JSON
  `{"username","password"}` — it sets an `odysseus_session` cookie. `GET /`
  returns 302 to the login page when unauthenticated.

### Expected, non-fatal startup behavior
- The bundled services (ChromaDB, SearXNG, ntfy) only run via `docker compose`
  and are NOT required for manual dev. Without ChromaDB you will see
  `MemoryVectorStore DEGRADED` / `Could not connect to a Chroma server` and a
  `ToolIndex init failed (will retry)` warning — this is expected; the app falls
  back to local FastEmbed embeddings + keyword retrieval.
- On first startup FastEmbed downloads a ~50MB ONNX embedding model from
  HuggingFace (needs network); it is cached under `~/.cache/fastembed` afterward.
- Chat / Agent / Deep Research features need an external LLM endpoint
  (Ollama / vLLM / OpenAI, configured in Settings) and are unavailable by
  default. For LLM-free verification, use the Documents, Notes, or Tasks
  features (e.g. create a document via Library → Documents).

### Tests / checks
- `./venv/bin/python -m pytest` — note 2 pre-existing failures in
  `tests/test_review_regressions.py` (`test_default_chat_*`). They are caused by
  an out-of-date import stub in that test that omits `Session` from its fake
  `core.database` module; the real `core.database` exports `Session` and the app
  imports fine. These failures are unrelated to environment setup.
