Directory structure:
└── coforge-forgex-dev-agent-da/
    ├── README.md
    ├── _inspect2.py
    ├── _inspect3.py
    ├── _inspect4.py
    ├── _inspect_registry.py
    ├── _register_agent.py
    ├── DEEP_AGENT_MIGRATION.md
    ├── Dockerfile
    ├── enable_file_logging.py
    ├── FASTAPI_RUN_GUIDE.md
    ├── main.py
    ├── pyproject.toml
    ├── requirements.txt
    ├── run.bat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
    ├── RUN_GUIDE.md
    ├── test_agent_invoke.py
    ├── .pre-commit-config.yaml
    ├── migrations/
    │   └── 2026_08_27_fe_agent_document_run_scope.sql
    ├── scripts/
    │   ├── apply_run_scope_migration.py
    │   ├── create_fe_agent_document.py
    │   ├── progress_relay.py
    │   └── servicebus_relay.py
    ├── src/
    │   ├── __init__.py
    │   ├── agent_builder.py
    │   ├── agent_memory_templates.py
    │   ├── agent_prompts.py
    │   ├── agent_tools.py
    │   ├── azure_claude_chat.py
    │   ├── api/
    │   │   ├── __init__.py
    │   │   └── routers/
    │   │       ├── __init__.py
    │   │       ├── conversation.py
    │   │       ├── health.py
    │   │       ├── integrations.py
    │   │       ├── mermaid.py
    │   │       ├── registry.py
    │   │       ├── session.py
    │   │       └── skills.py
    │   ├── aws_s3_backend/
    │   │   ├── __init__.py
    │   │   └── cloud_backends.py
    │   ├── azure_blob_backend/
    │   │   ├── __init__.py
    │   │   ├── _path.py
    │   │   ├── _utils.py
    │   │   ├── backend.py
    │   │   └── config.py
    │   ├── core/
    │   │   ├── __init__.py
    │   │   ├── config.py
    │   │   ├── database.py
    │   │   ├── exceptions.py
    │   │   ├── logging.py
    │   │   └── postgres.py
    │   ├── integrations/
    │   │   ├── __init__.py
    │   │   ├── github_tools.py
    │   │   └── jira_tools.py
    │   ├── services/
    │   │   ├── __init__.py
    │   │   ├── agent_memory.py
    │   │   ├── design_snapshot.py
    │   │   ├── dev_jobs.py
    │   │   ├── documents.py
    │   │   ├── intake_parsing.py
    │   │   ├── kb_client.py
    │   │   ├── mermaid.py
    │   │   ├── mongo_models.py
    │   │   ├── registry.py
    │   │   ├── run_events.py
    │   │   ├── session_managers.py
    │   │   ├── skills.py
    │   │   ├── stream_events.py
    │   │   ├── workflow_context.py
    │   │   └── workspace_tools.py
    │   └── skills/
    │       └── DEV_SCAFFOLDING_SKILL.md
    ├── tests/
    │   ├── conftest.py
    │   ├── test_cancel_job_api.py
    │   ├── test_dev_jobs.py
    │   ├── test_http_example.py
    │   ├── test_mcp_client.py
    │   ├── test_send_message_status_api.py
    │   ├── test_session_managers.py
    │   └── test_worker_poison_handlers.py
    └── .github/
        ├── instructions/
        │   ├── 01-personal-to-feature.md
        │   ├── 02-feature-to-dev.md
        │   ├── 03-dev-to-stage.md
        │   ├── 04-stage-to-prod.md
        │   └── git_routing.md
        └── workflows/
            ├── function-deployment.yaml
            └── python-da-dev-codedev-agents.yaml

================================================
FILE: README.md
================================================
# ForgeX Architect REST API

A serverless REST API for the **ForgeX Agent Architecture Platform**, built on the
**Azure Functions Python programming model v2** (decorator-based, single
`function_app.py`, no per-function `function.json`).

It manages chat **sessions**, **conversation history**, and **session synopses**
backed by MongoDB, and renders **Mermaid architecture diagrams** to SVG via a
headless Chromium browser. The data layer mirrors the ForgeX MCP server so both
services read/write the same MongoDB cluster and collections.

---

## Architecture

```
                          ┌────────────────────────────────┐
   HTTP client  ───────▶  │  function_app.py (Functions    │
   (Bearer JWT)           │  v2 entry point, route table)  │
                          └───────────────┬────────────────┘
                                          │  delegates to functions.<name>.main
                                          ▼
        ┌─────────────────────────────────────────────────────────────┐
        │  Per-function handler  (src/functions/<name>/__init__.py)     │
        │                                                               │
        │   @azure_http_decorator()   ← logging, correlation-id, errors │
        │   @require_auth()           ← JWT verify, attach claims       │
        │   async def main():                                           │
        │       parse_request(req, <Request>)   ← Pydantic validation   │
        │       user_id = get_user_id(req)       ← identity from JWT     │
        │       await <impl>(...)                ← business logic        │
        │       return APIResponse(...)                                 │
        └───────────────────────────────┬───────────────────────────────┘
                                         ▼
        ┌─────────────────────────────────────────────────────────────┐
        │  src/core/  (shared infrastructure)                          │
        │   session_managers  →  database (Motor / async MongoDB)      │
        │   config (Settings + Key Vault) · auth · middleware ·        │
        │   logging (structlog) · exceptions · mongo_models           │
        └─────────────────────────────────────────────────────────────┘
                                         ▼
                          MongoDB (architect_chatbot_db)
```

### System architecture

```mermaid
flowchart TB
    client["HTTP client<br/>(Authorization: Bearer JWT)"]

    subgraph host["Azure Functions host"]
        entry["function_app.py<br/>route table (v2 decorators)"]
        subgraph handler["Per-function handler<br/>src/functions/&lt;name&gt;/__init__.py"]
            mw["@azure_http_decorator<br/>logging · correlation-id · errors"]
            auth["@require_auth<br/>JWT verify · attach claims"]
            main["async def main()<br/>parse_request → impl → APIResponse"]
            mw --> auth --> main
        end
    end

    subgraph core["src/core — shared infrastructure"]
        sm["session_managers"]
        db["database<br/>DatabaseManager (Motor)"]
        cfg["config<br/>Settings + Key Vault"]
        log["logging (structlog)"]
        exc["exceptions"]
        models["mongo_models<br/>ChatMessage · SessionContext"]
        sm --> db
    end

    mongo[("MongoDB<br/>architect_chatbot_db")]
    kv[["Azure Key Vault<br/>(optional)"]]
    chromium["pyppeteer<br/>headless Chromium"]

    client -->|HTTPS| entry
    entry -->|delegates to main| mw
    main --> sm
    main -.render-mermaid.-> chromium
    db --> mongo
    cfg -.secrets.-> kv

    classDef opt stroke-dasharray:4 4;
    class kv,chromium opt;
```

**Stack**

| Concern        | Choice                                               |
| -------------- | ---------------------------------------------------- |
| Runtime        | Azure Functions v2 (Python, decorator model)         |
| Validation     | Pydantic (request payloads + Mongo document models)  |
| Database       | MongoDB via **Motor** (async driver)                 |
| Auth           | JWT (HS256 shared secret); optional Azure Key Vault  |
| Logging        | `structlog` (structured JSON, correlation IDs)       |
| Diagram render | Mermaid + `pyppeteer` (headless Chromium → SVG)      |


---

## Design patterns

- **Decorator-based middleware stack.** Handlers are wrapped, outermost first, by
  `@azure_http_decorator()` (start/end logging, `X-Correlation-ID`, uniform
  exception → JSON 500) and `@require_auth()` (JWT verification, claims attached
  to the request). See [src/core/middleware.py](src/core/middleware.py) and
  [src/core/auth.py](src/core/auth.py).
- **Lazy singletons / module-level globals.** `db_manager`, `settings`, `logger`,
  and per-function `_session` managers are created once and reused, e.g.:
  ```python
  _session: Optional[SessionHistoryManager] = None

  def _get_session() -> SessionHistoryManager:
      global _session
      if _session is None:
          _session = SessionHistoryManager()
      return _session
  ```
- **Parse-or-error request handling.** `parse_request(req, Model)` returns
  `(payload, None)` on success or `(None, error_response)` on validation failure,
  so handlers stay flat. See [src/shared/payloads.py](src/shared/payloads.py).
- **Uniform envelope.** Every handler returns an `APIResponse(success, message,
  data, error, ...)`. See [src/shared/models.py](src/shared/models.py).
- **Exception hierarchy.** `APIException` base with `ValidationException`,
  `NotFoundException`, `DatabaseException`, etc., translated to consistent JSON by
  the decorator. See [src/core/exceptions.py](src/core/exceptions.py).
- **Nested settings.** `Settings` composes `database`, `azure`, `security`, and
  `render` sub-settings from environment + optional Key Vault. See
  [src/core/config.py](src/core/config.py).

## Coding patterns & conventions

- **Async-first.** Handlers are `async def main(req, context)`; Motor calls are
  awaited directly on the Functions event loop. Blocking work (filesystem,
  Chromium) is offloaded with `await asyncio.to_thread(...)`.
- **Identity from the token, never the body.** User identity is always read via
  `get_user_id(req)` from verified JWT claims, not from request fields.
- **Pydantic everywhere.** Request payloads inherit `BasePayload`
  (`extra="forbid"`); Mongo models expose `to_document()` (`exclude_none=True`)
  for writes and `model_validate()` for reads.
- **Structured logging.** `logger.info("...", key=value)` — never f-string blobs;
  every record carries app/env/version and the correlation id.
- **Naming.** Business logic lives in `*_impl` functions; private helpers are
  `_`-prefixed; request models are suffixed `Request`; each function's HTTP entry
  is `main(req, context)`.
- **Tooling.** Black (88 cols) + isort, configured in
  [pyproject.toml](pyproject.toml) and enforced via pre-commit.

---

## Request flow (example: store a message)

```
POST /api/store-conversation   (Authorization: Bearer <jwt>)
  └─ function_app.py: store_conversation()
       └─ @azure_http_decorator   → correlation id, start log
            └─ @require_auth       → verify JWT, attach claims
                 └─ functions/store_conversation/main()
                      ├─ parse_request(req, StoreConversationRequest)  → 400 on invalid
                      ├─ user_id = get_user_id(req)
                      ├─ store_conversation_impl(...)
                      │     ├─ await db_manager.initialize()
                      │     └─ SessionHistoryManager.append_message()
                      │           └─ ChatMessage.to_document() → collection.insert_one()
                      └─ return APIResponse(success=True, data=...)
  ◀─ JSON body + X-Correlation-ID header   (errors → 500 with same envelope)
```

### Sequence (Mermaid)

```mermaid
sequenceDiagram
    autonumber
    actor C as Client
    participant F as function_app.py
    participant D as "@azure_http_decorator"
    participant A as "@require_auth"
    participant H as handler.main
    participant S as SessionHistoryManager
    participant M as MongoDB

    C->>F: POST /store-conversation (Bearer JWT)
    F->>D: store_conversation(req, ctx)
    D->>D: generate correlation id · start log
    D->>A: invoke wrapped handler
    A->>A: decode + verify JWT
    alt token invalid / missing
        A-->>C: 401 APIResponse(success=false)
    else token valid
        A->>H: req (claims attached)
        H->>H: parse_request(req, StoreConversationRequest)
        alt payload invalid
            H-->>C: 400 APIResponse(error=...)
        else payload valid
            H->>H: user_id = get_user_id(req)
            H->>S: append_message(...)
            S->>M: insert_one(ChatMessage.to_document())
            M-->>S: ack
            S-->>H: ok
            H-->>D: APIResponse(success=true, data)
            D-->>C: 200 JSON + X-Correlation-ID
        end
    end
    Note over D: any unhandled exception → 500 with same envelope
```

---

## Project structure

```
rest/
├── function_app.py              # Functions v2 entry point; registers all routes
├── host.json                    # Runtime config (timeout, App Insights)
├── local.settings.json          # Local dev settings
├── requirements.txt             # Runtime dependencies
├── pyproject.toml               # Metadata + Black/isort/pytest config
├── .env.example                 # Environment template
│
└── src/
    ├── core/                    # Shared infrastructure
    │   ├── __init__.py          # Re-exports core utilities
    │   ├── config.py            # Settings + KeyVaultManager (nested Pydantic)
    │   ├── database.py          # DatabaseManager (async Motor client)
    │   ├── mongo_models.py      # ChatMessage, SessionContext
    │   ├── session_managers.py  # SessionHistoryManager, SessionContextManager
    │   ├── auth.py              # JWT verify, @require_auth, get_user_id
    │   ├── middleware.py        # @azure_http_decorator, CORS/security headers
    │   ├── logging.py          # structlog setup + Logger
    │   └── exceptions.py        # APIException hierarchy + handlers
    │
    ├── agent/                   # Agent runtime + MCP client wrappers
    │   ├── agent_runtime.py     # LLM Router singleton
    │   └── mcp_client.py        # MCP client integration
    │
    ├── services/                # Business logic & orchestration
    │   ├── architect_service.py # ported send_message orchestration
    │   ├── architect_jobs.py    # Mongo-backed async job store
    │   ├── design_snapshot.py   # async last-design snapshot store
    │   └── mermaid.py           # shared Mermaid → SVG pipeline
    │
    ├── functions/               # Azure Functions (HTTP + Queue triggers)
    │   ├── api/                 # HTTP-triggered endpoints
    │   │   ├── create_session/      # Generate a new session id
    │   │   ├── store_conversation/  # Append a message to history
    │   │   ├── send_message/        # Start async TSD generation (HTTP + queue out)
    │   │   ├── send_message_status/ # Poll a send_message job
    │   │   ├── get_conversation/    # Fetch history (optional limit)
    │   │   ├── get_synopsis/        # List recent sessions w/ titles
    │   │   ├── edit_synopsis/       # Update a session title/context
    │   │   ├── delete_conversation/ # Delete session + context
    │   │   ├── render_mermaid/      # Mermaid → SVG (headless Chromium)
    │   │   ├── get_architecture_zip_url/ # Build ZIP download URL
    │   │   ├── cancel_job/          # Cancel a queued/running job
    │   │   ├── update_design_hld/   # Start async HLD update (HTTP + queue out)
    │   │   └── http_example/        # Sample endpoint
    │   └── workers/                 # Queue-triggered background workers
    │       ├── send_message_worker/     # Process TSD generation jobs
    │       └── update_design_hld_worker/# Process HLD update jobs
    │           └── (__init__.py = handler, payloads.py = request model)
    │
    └── shared/                  # Cross-cutting helpers
        ├── models.py            # APIResponse and friends
        ├── payloads.py          # BasePayload, parse_request()
        └── utils.py             # response/JSON helpers
```

Each function folder contains `__init__.py` (the `main` handler + `*_impl`
business logic) and, where it takes input, a `payloads.py` with its `*Request`
Pydantic model.

---

## API endpoints

All routes are served under the Functions HTTP prefix (default `/api`).
Application-level auth is enforced by `@require_auth()` (JWT); the Azure
platform `auth_level` is `ANONYMOUS` for all routes except `hello` (`FUNCTION`).

| Method      | Route                         | Description                                  |
| ----------- | ----------------------------- | -------------------------------------------- |
| POST        | `/create-session`             | Generate a new session id                    |
| POST        | `/store-conversation`         | Append a message to a session's history      |
| POST        | `/send-message`               | Start async TSD/HLD generation → `202 {job_id}` |
| GET/POST    | `/send-message-status`        | Poll a send_message job (`?job_id=`)         |
| GET/POST    | `/get-conversation`           | Retrieve a session's history (optional limit)|
| GET/POST    | `/get-synopsis`               | List recent sessions with titles             |
| POST        | `/edit-synopsis`              | Update a session's title/context             |
| POST/DELETE | `/delete-conversation`        | Delete a session and its context             |
| GET/POST    | `/render-mermaid`             | Render a Mermaid diagram to SVG              |
| GET/POST    | `/get-architecture-zip-url`   | Build an architecture ZIP download URL       |
| GET/POST    | `/hello`                      | Sample endpoint (auth_level = FUNCTION)       |
| GET         | `/health`                     | Health check — `{"status":"healthy"}`        |

---

## Data model (MongoDB)

Database `architect_chatbot_db` (configurable). Collections and indexes are
created automatically on startup.

| Collection                   | Model            | Key fields                                                          |
| ---------------------------- | ---------------- | ------------------------------------------------------------------ |
| `architect_chat_collection`  | `ChatMessage`    | workspace_id, user_id, session_id, role, content, timestamp, file? |
| `architect_session_context`  | `SessionContext` | workspace_id, user_id, session_id, title?, timestamp               |

The `DatabaseManager` owns a single `AsyncIOMotorClient` (pool 5–20, 30s timeout,
Server API v1) and is initialized lazily per request via
`await db_manager.initialize()`.

### Document model (Mermaid)

```mermaid
erDiagram
    SESSION_CONTEXT ||--o{ CHAT_MESSAGE : "groups (by session_id)"

    SESSION_CONTEXT {
        string workspace_id
        string user_id
        string session_id PK
        string title "optional"
        datetime timestamp
    }
    CHAT_MESSAGE {
        string workspace_id
        string user_id
        string session_id FK
        string role "user | assistant | system"
        string content
        object  file "optional"
        datetime timestamp
    }
```

> Relationship is logical (shared `session_id`), not a DB-enforced foreign key —
> MongoDB does not enforce referential integrity.

---

## Architect agent (`send_message`)

`POST /send-message` runs the **same work as the MCP server's `send_message`
tool** — intent classification → TSD/HLD generation (or markdown update, or a
greeting) → history persistence → ZIP artifact → inline Mermaid→SVG — directly in
the REST API, reusing the **real** `agent_architect` package (no mocks).

- **Layering.** Agent-specific code lives in
  [`src/services/`](src/services/) (`prompts`, `artifact_zipper`,
  `workflow_context`, `architect_service`, `mermaid`, session managers); reusable
  infra lives in [`src/core/`](src/core/) (`config`, `database`,
  `llm_router_adapter`) and [`src/agent/`](src/agent/) (generic agent
  runtime + MCP client). This repo is the **base for all agents** — new agents get
  their own `*_service` folder and reuse `core`/`agent`/`shared`. REST is
  self-contained and does **not** install the sibling `agent_architect` package.
- **Environment source.** Runtime settings come from process environment
  variables (Azure Function App settings in production). For local development,
  keep required keys in `local.settings.json` under `Values`.
- **Async job + poll.** Generation can take minutes, longer than an HTTP request
  should hold open on Functions. So `POST /send-message` creates a durable job in
  Mongo (`architect_jobs`), enqueues a Storage Queue message (`arch-send-message`,
  via output binding), and returns **202** with a `job_id`. A **queue-triggered
  worker** (`send_message_worker`) runs the generation to completion and writes
  the result back to the job. Clients poll **`GET /send-message-status?job_id=`**
  until `status` is `completed`/`failed`. `AzureWebJobsStorage` backs the queue.
- **Data layer.** Chat history, session context, design snapshots, and the job
  store all use REST's existing **async (Motor)** layer
  ([`core/session_managers.py`](src/core/session_managers.py),
  [`core/design_snapshot.py`](src/core/design_snapshot.py),
  [`core/architect_jobs.py`](src/core/architect_jobs.py)).
- **Deploy note.** The agent code is vendored in-repo, so the only external
  dependency is **`common-adapters`** (the LLM Router; pinned to its git URL in
  `requirements.txt`) plus `python-docx`. No `agent_architect` install is needed.
  If the MCP-side originals change, re-sync the copies under `src/architect/`.

---

## Security

- **JWT (HS256).** `Authorization: Bearer <token>`; required claims `exp`,
  `user_id`, `email` (base64-wrapped tokens are auto-decoded). Helpers:
  `require_admin`, `require_workspace`, `require_role`.
- **Security headers** added to every response: `X-Content-Type-Options`,
  `X-Frame-Options`, `X-XSS-Protection`, `Strict-Transport-Security`,
  `Referrer-Policy`, `Permissions-Policy`.
- **Correlation IDs.** A `uuid4` per request, returned in `X-Correlation-ID` and
  attached to every log line and error response.
- **CORS** origins are configurable via `settings.security.CORS_ORIGINS`.
- **Secrets.** When `KEYVAULT_URL` is set, secrets (Mongo URI, JWT secret, …) are
  loaded from Azure Key Vault, otherwise from the environment.

---

## Quick start

### Prerequisites
- Python 3.11+
- [Azure Functions Core Tools v4 (`func`)](https://learn.microsoft.com/azure/azure-functions/functions-run-local)
- A MongoDB connection string
- (Optional) Azure Key Vault, Application Insights

Install Azure Functions Core Tools (`func`) with one of these options:

```bash
# Windows (recommended)
winget install Microsoft.Azure.FunctionsCoreTools

# npm (cross-platform)
npm i -g azure-functions-core-tools
```

Reference: [azure-functions-core-tools - npm](https://www.npmjs.com/package/azure-functions-core-tools)

### Why this repo has both `requirements.txt` and `pyproject.toml`

- `requirements.txt` is the runtime install contract for Azure Functions and local
  `func` execution. The host/runtime flow and VS Code tasks install from this file.
- `pyproject.toml` is the Python project metadata and tooling config source
  (Black, isort, mypy, pytest options, package metadata, optional `dev` extras).
- Keep runtime packages in sync between both files when adding/removing
  production dependencies.

In short: **Functions runtime uses `requirements.txt`; developer tooling and
project metadata live in `pyproject.toml`.**

### Configuration files: `local.settings.json` and `.env`

- `local.settings.json` is read by Azure Functions Core Tools and is the primary
  source for local Function App environment variables.
- `.env` is optional local fallback for app-level defaults via `pydantic-settings`.
- In production, Azure Function App settings provide environment variables;
  `local.settings.json` is not required or deployed.
- For auth to work locally, ensure `JWT_SECRET_KEY` in `local.settings.json`
  matches the secret used by the token issuer.

### Queue emulator (local only)

- **Serverless Azure does not need Azurite.** In Azure, the queue bindings use the
  real Storage Account configured via `AzureWebJobsStorage`.
- **Local development does need Azurite** when testing queue-backed endpoints such
  as `POST /send-message` and `POST /update-design-hld`.

```bash
# run local Storage emulator (queue/blob/table)
npx azurite --location .azurite --silent
```

Then, in another terminal, start Functions:

```bash
func start
```

### Setup
```bash
cd rest

# Install runtime dependencies (used by local func host)
pip install -r requirements.txt

# Install the project with dev tooling (black/isort/mypy/pytest/pre-commit)
pip install -e ".[dev]"

# Configure local Function host settings
# Add required keys under Values, including JWT_SECRET_KEY.

# Install git hooks so checks run before each commit
pre-commit install

# Optional: run all hooks immediately to validate your setup
pre-commit run --all-files
```

### Run locally
```bash
func start
```
The runtime indexes `function_app.py` and serves the routes above on
`http://localhost:7071/api/...`.

If your requests return `Invalid token: Signature verification failed`, verify:
- `Authorization` header format is `Bearer <token>`
- `JWT_SECRET_KEY` exists in `local.settings.json`
- local secret matches the environment that issued the token

### Key environment variables
```env
APP_NAME=forgex-rest-api
ENVIRONMENT=development
LOG_LEVEL=INFO

# MongoDB
MONGODB_DATABASE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
ARCHITECT_DB_NAME=architect_chatbot_db

# Auth
JWT_SECRET_KEY=your-shared-secret
JWT_ALGORITHM=HS256

# Optional Azure services
KEYVAULT_URL=https://your-keyvault.vault.azure.net/
APPINSIGHTS_INSTRUMENTATION_KEY=...

# Mermaid rendering
PUBLIC_BASE_URL=https://your-host
CHROME_EXECUTABLE_PATH=/path/to/chromium
```
See [local.settings.json](local.settings.json) for the active local values.

---

## Development

### First-day contributor checklist
```bash
# 1) install deps
pip install -r requirements.txt
pip install -e ".[dev]"

# 2) configure local.settings.json Values (JWT_SECRET_KEY, Mongo, etc.)

# 3) install and run git hooks
pre-commit install
pre-commit run --all-files

# 4) run tests
pytest

# 5) run the Function App locally
func start
```

### Testing
```bash
# Unit tests
pytest

# Coverage
pytest --cov=src

# Local endpoint testing (requires Azure Functions Core Tools / func)
func start
# then call endpoints at http://localhost:7071/api/...
```

To test API flows that depend on middleware/auth, prefer running the local
Functions host (`func start`) and testing through HTTP rather than calling
handler functions directly.

### Code quality
```bash
# Run all configured checks manually
pre-commit run --all-files
```

Current hooks are defined in [.pre-commit-config.yaml](.pre-commit-config.yaml)
and include formatting, linting, import ordering, static checks, and type checks.
After `pre-commit install`, they run automatically on every commit.

### Logging
```python
from core import logger

logger.info("Message stored", session_id=sid, user_id=uid)
logger.error("Store failed", error=e, correlation_id=cid)
```

### Errors
```python
from core import ValidationException, NotFoundException

raise ValidationException("Invalid input", details={"field": "role"})
raise NotFoundException("Session not found", resource_type="session")
```

---

## Deployment

```bash
# Azure Functions
func azure functionapp publish <your-function-app>
```
Configure application settings (Mongo URI, JWT secret, Key Vault URL, App
Insights key) in the Function App configuration, and provide a Chromium binary /
`CHROME_EXECUTABLE_PATH` if Mermaid rendering is used.

For architecture ZIP artifacts, configure Blob storage settings:
- `BLOB_STORAGE_CONNECTION_STRING`: storage connection string (must include account key for SAS generation)
- `ARTIFACT_BLOB_CONTAINER_NAME`: container used for generated ZIPs (default `architect-artifacts`)
- `ARTIFACT_BLOB_PATH_PREFIX`: blob prefix/folder for ZIPs (default `architecture-zips`)
- `ARTIFACT_BLOB_URL_EXPIRY_MINUTES`: short-lived SAS expiry in minutes (default `5`)

`POST /send-message` and `POST /update-design-hld` now upload generated architecture ZIPs directly to Blob storage. `GET/POST /get-architecture-zip-url` returns a short-lived SAS download URL.



================================================
FILE: _inspect2.py
================================================
import asyncio, socket, os
import asyncpg
from dotenv import load_dotenv
load_dotenv(os.path.join(os.path.dirname(__file__), ".env"), override=True)
HOST=os.environ["POSTGRESQL_DATABASE_HOST"];PORT=int(os.environ.get("POSTGRESQL_DATABASE_PORT",5432))
DB=os.environ["POSTGRESQL_DATABASE_DATABASE"];USER=os.environ["POSTGRESQL_DATABASE_USER"];PWD=os.environ["POSTGRESQL_DATABASE_PASSWORD"]

async def main():
    ip=socket.gethostbyname(HOST)
    c=await asyncpg.connect(host=ip,port=PORT,user=USER,password=PWD,database=DB,ssl="require",timeout=30)
    print("== workspace mappings for agent_id=76 ==")
    rows=await c.fetch("SELECT * FROM workspace_agents_mapping_2 WHERE agent_id=76 ORDER BY workspace_id")
    for r in rows: print(dict(r))
    print("count:", len(rows))
    print("\n== does agents_details.agent_id have a sequence default? ==")
    print(await c.fetchval("SELECT pg_get_serial_sequence('agents_details','agent_id')"))
    print("\n== already a Dev-Dynamic row? ==")
    for r in await c.fetch("SELECT agent_id,agent_name FROM agents_details WHERE agent_name ILIKE '%dev%dynamic%'"):
        print(dict(r))
    print("\n== agent_category '4, 8' meaning: distinct categories in use (sample) ==")
    for r in await c.fetch("SELECT DISTINCT agent_category FROM agents_details LIMIT 20"):
        print(r['agent_category'])
    await c.close()
asyncio.run(main())



================================================
FILE: _inspect3.py
================================================
import asyncio, socket, os
import asyncpg
from dotenv import load_dotenv
load_dotenv(os.path.join(os.path.dirname(__file__), ".env"), override=True)
HOST=os.environ["POSTGRESQL_DATABASE_HOST"];PORT=int(os.environ.get("POSTGRESQL_DATABASE_PORT",5432))
DB=os.environ["POSTGRESQL_DATABASE_DATABASE"];USER=os.environ["POSTGRESQL_DATABASE_USER"];PWD=os.environ["POSTGRESQL_DATABASE_PASSWORD"]

async def main():
    ip=socket.gethostbyname(HOST)
    c=await asyncpg.connect(host=ip,port=PORT,user=USER,password=PWD,database=DB,ssl="require",timeout=30)
    # every table that has an agent_id column
    cols=await c.fetch("SELECT table_name FROM information_schema.columns "
                       "WHERE column_name='agent_id' AND table_schema='public' ORDER BY table_name")
    print("== tables with agent_id column ==")
    for r in cols:
        t=r['table_name']
        try:
            n=await c.fetchval(f'SELECT COUNT(*) FROM "{t}" WHERE agent_id=76')
            print(f"  {t:45} rows_with_agent76={n}")
        except Exception as e:
            print(f"  {t:45} ERR {e}")
    # tables with public_agent_id
    print("\n== tables with public_agent_id column ==")
    for r in await c.fetch("SELECT table_name FROM information_schema.columns "
                           "WHERE column_name='public_agent_id' AND table_schema='public'"):
        print("  ",r['table_name'])
    # sample the OTHER dynamic agents to see their category/mapping pattern
    print("\n== other dynamic/deep agents (for comparison) ==")
    for r in await c.fetch("SELECT agent_id,agent_name,agent_category,is_active FROM agents_details "
                           "WHERE agent_name ILIKE '%dynamic%' OR agent_name ILIKE '%deep-agent%' ORDER BY agent_id"):
        print("  ",dict(r))
    await c.close()
asyncio.run(main())



================================================
FILE: _inspect4.py
================================================
import asyncio, socket, os
import asyncpg
from dotenv import load_dotenv
load_dotenv(os.path.join(os.path.dirname(__file__), ".env"), override=True)
HOST=os.environ["POSTGRESQL_DATABASE_HOST"];PORT=int(os.environ.get("POSTGRESQL_DATABASE_PORT",5432))
DB=os.environ["POSTGRESQL_DATABASE_DATABASE"];USER=os.environ["POSTGRESQL_DATABASE_USER"];PWD=os.environ["POSTGRESQL_DATABASE_PASSWORD"]

async def cols(c,t):
    print(f"\n== columns: {t} ==")
    for r in await c.fetch("SELECT column_name,data_type FROM information_schema.columns "
                           "WHERE table_name=$1 ORDER BY ordinal_position",t):
        print(f"  {r['column_name']:35}{r['data_type']}")

async def main():
    ip=socket.gethostbyname(HOST)
    c=await asyncpg.connect(host=ip,port=PORT,user=USER,password=PWD,database=DB,ssl="require",timeout=30)

    await cols(c,"workspace_agent_provider_model_mapping")
    print("-- agent 76 rows --")
    for r in await c.fetch("SELECT * FROM workspace_agent_provider_model_mapping WHERE agent_id=76 ORDER BY workspace_id"):
        print("  ",dict(r))

    await cols(c,"agent_registry")
    print("-- agent_registry rows mentioning dev/dynamic (name-ish cols) --")
    # try common name columns
    for r in await c.fetch("SELECT * FROM agent_registry LIMIT 3"):
        print("  sample:",dict(r))

    await cols(c,"agents_details_2")
    print("-- agents_details_2 dev row? --")
    for r in await c.fetch("SELECT * FROM agents_details_2 WHERE agent_name ILIKE '%dev%' OR agent_name ILIKE '%dynamic%' LIMIT 5"):
        print("  ",dict(r))

    await c.close()
asyncio.run(main())



================================================
FILE: _inspect_registry.py
================================================
"""One-off: inspect the ForgeX Postgres registry for the Dev-dynamic-agent
row + its workspace mappings, so we can replicate them for Dev-Dynamic-Agent."""
import asyncio, socket, os
import asyncpg
from dotenv import load_dotenv

load_dotenv(os.path.join(os.path.dirname(__file__), ".env"), override=True)

HOST = os.environ["POSTGRESQL_DATABASE_HOST"]
PORT = int(os.environ.get("POSTGRESQL_DATABASE_PORT", 5432))
DB   = os.environ["POSTGRESQL_DATABASE_DATABASE"]
USER = os.environ["POSTGRESQL_DATABASE_USER"]
PWD  = os.environ["POSTGRESQL_DATABASE_PASSWORD"]


async def main():
    ip = socket.gethostbyname(HOST)
    conn = await asyncpg.connect(host=ip, port=PORT, user=USER, password=PWD,
                                 database=DB, ssl="require", timeout=30)
    print("== columns: agents_details ==")
    for r in await conn.fetch(
        "SELECT column_name, data_type FROM information_schema.columns "
        "WHERE table_name='agents_details' ORDER BY ordinal_position"):
        print(f"  {r['column_name']:30} {r['data_type']}")

    print("\n== columns: agents_cms ==")
    for r in await conn.fetch(
        "SELECT column_name, data_type FROM information_schema.columns "
        "WHERE table_name='agents_cms' ORDER BY ordinal_position"):
        print(f"  {r['column_name']:30} {r['data_type']}")

    print("\n== columns: workspace_agents_mapping_2 ==")
    for r in await conn.fetch(
        "SELECT column_name, data_type FROM information_schema.columns "
        "WHERE table_name='workspace_agents_mapping_2' ORDER BY ordinal_position"):
        print(f"  {r['column_name']:30} {r['data_type']}")

    print("\n== agents_details rows matching dev (any case) ==")
    for r in await conn.fetch(
        "SELECT * FROM agents_details WHERE agent_name ILIKE '%dev%'"):
        print(dict(r))

    print("\n== agents_cms rows for those agent_ids ==")
    for r in await conn.fetch(
        "SELECT c.* FROM agents_cms c JOIN agents_details d ON d.agent_id=c.agent_id "
        "WHERE d.agent_name ILIKE '%dev%'"):
        print(dict(r))

    print("\n== workspace_agents_mapping_2 rows for those agent_ids ==")
    for r in await conn.fetch(
        "SELECT m.* FROM workspace_agents_mapping_2 m JOIN agents_details d ON d.agent_id=m.agent_id "
        "WHERE d.agent_name ILIKE '%dev%'"):
        print(dict(r))

    print("\n== max agent_id ==")
    print(await conn.fetchval("SELECT MAX(agent_id) FROM agents_details"))

    await conn.close()


asyncio.run(main())



================================================
FILE: _register_agent.py
================================================
"""Register Dev-Dynamic-Agent in the ForgeX Postgres registry, mirroring the
Dev-dynamic-agent (agent_id=76): same workspaces, category, owner. The
agent_name MUST be exactly 'Dev-Dynamic-Agent' to match the frontend
agentToolPath key -> route /dev-dynamic-agent."""
import asyncio, socket, os, uuid
import asyncpg
from dotenv import load_dotenv
load_dotenv(os.path.join(os.path.dirname(__file__), ".env"), override=True)
HOST=os.environ["POSTGRESQL_DATABASE_HOST"];PORT=int(os.environ.get("POSTGRESQL_DATABASE_PORT",5432))
DB=os.environ["POSTGRESQL_DATABASE_DATABASE"];USER=os.environ["POSTGRESQL_DATABASE_USER"];PWD=os.environ["POSTGRESQL_DATABASE_PASSWORD"]

NAME="Dev-Dynamic-Agent"
CATEGORY="4, 8"
DESC=("Dynamic Dev agent: reads a TSD and generates a well-planned code "
      "scaffolding — a PLAN.md first, then a structured source tree of stub "
      "files with docstrings, signatures and TODOs.")
OWNER="forge-x@coforge.com"
FEATURE=('[{"name": "TSD-Driven Planning", "description": "Reads the Technical '
         'Specification Document and produces a PLAN.md with chosen stack, module '
         'breakdown, folder structure and build order."}, {"name": "Code Scaffolding '
         'Generation", "description": "Generates a structured source tree of stub '
         'files with docstrings, signatures, config, entrypoints, tests skeleton and '
         'README."}]')
WORKSPACES=[5,1058,1062,1063,1064]

async def main():
    ip=socket.gethostbyname(HOST)
    c=await asyncpg.connect(host=ip,port=PORT,user=USER,password=PWD,database=DB,ssl="require",timeout=30)
    async with c.transaction():
        existing=await c.fetchrow("SELECT agent_id FROM agents_details WHERE agent_name=$1",NAME)
        if existing:
            print("Already exists, agent_id=",existing["agent_id"]); return
        agent_id=await c.fetchval(
            'INSERT INTO agents_details '
            '(agent_name, agent_category, agent_desc, agent_environment, is_active, '
            ' created_date, last_updated, public_agent_id) '
            "VALUES ($1,$2,$3,'',TRUE,NOW(),NOW(),$4) RETURNING agent_id",
            NAME,CATEGORY,DESC,uuid.uuid4())
        print("Inserted agents_details agent_id=",agent_id)
        await c.execute(
            'INSERT INTO agents_cms '
            '(agent_id, agent_owner, agent_contact, agent_feature, is_active, created_date, last_updated) '
            'VALUES ($1,$2,$2,$3,TRUE,NOW(),NOW())',
            agent_id,OWNER,FEATURE)
        print("Inserted agents_cms")
        for ws in WORKSPACES:
            await c.execute(
                'INSERT INTO workspace_agents_mapping_2 '
                '(workspace_id, agent_id, is_active, created_date, last_updated) '
                'VALUES ($1,$2,TRUE,NOW(),NOW())', ws, agent_id)
        print("Mapped to workspaces:",WORKSPACES)
        # 4. LLM provider/model binding (mirror agent 76: provider_model_id=1,
        #    is_default=true). Without this the agent has no model to run with.
        for ws in WORKSPACES:
            await c.execute(
                'INSERT INTO workspace_agent_provider_model_mapping '
                '(workspace_id, agent_id, provider_model_id, is_default, is_active, created_by, created_at, updated_at) '
                'VALUES ($1,$2,1,TRUE,TRUE,1,NOW(),NOW())', ws, agent_id)
        print("Provider/model mapping added for workspaces:",WORKSPACES)
    # verify
    r=await c.fetchrow("SELECT agent_id,agent_name,agent_category,is_active,public_agent_id "
                       "FROM agents_details WHERE agent_name=$1",NAME)
    print("VERIFY:",dict(r))
    m=await c.fetch("SELECT workspace_id FROM workspace_agents_mapping_2 WHERE agent_id=$1 ORDER BY workspace_id",r["agent_id"])
    print("VERIFY mappings:",[x["workspace_id"] for x in m])
    await c.close()
asyncio.run(main())



================================================
FILE: DEEP_AGENT_MIGRATION.md
================================================
# Deep Agent Architecture Migration - Implementation Summary

## Overview

This document summarizes the migration of the Architect agent from a monolithic service-based architecture to the Deep Agent framework pattern, following the reference implementation from `FE_Deep_AGENT_v1/backend/agent_builder.py`.

## Completed Implementation

### 1. Core Deep Agent Files Created

#### `src/agent_builder.py`
- Main entry point for creating Architect Deep Agent instances
- Implements `create_architect_agent()` function with workspace context
- Backend selection logic (Azure Blob, S3, or Filesystem)
- Integrates with DESIGN_SKILLS pattern
- **Pattern**: Follows `FE_Deep_AGENT_v1/backend/agent_builder.py`

#### `src/agent_prompts.py`
- System prompts for the Architect agent
- Migrated from `services/prompts.py` HLD_PROMPT
- Added tool descriptions and workflow instructions
- Context-aware prompt building with workspace/conversation IDs

#### `src/agent_tools.py`
- LangChain tools wrapping existing services
- **Tools Created**:
  - `curate_workflow_context()` - Fetch BRD and project artifacts
  - `load_conversation_history()` - Retrieve session messages
  - `save_design_snapshot()` - Persist TSD to database
  - `get_previous_design()` - Load last generated design
  - `render_mermaid_diagram()` - Convert Mermaid to SVG
  - `get_active_skill()` - Load skill configuration
- **Pattern**: Closure-based tools capturing workspace context (no global state)

#### `src/agent_memory_templates.py`
- Skill and memory file mappings
- Defines `DESIGN_SKILLS = ["/skills/DESIGN_ARCHITECT_SKILL.md"]`
- Virtual path mappings for FilesystemBackend

#### `src/skills/DESIGN_ARCHITECT_SKILL.md`
- Comprehensive skill definition (650+ lines)
- Migrated from JSON skill format to Markdown
- **Sections**:
  - Role & Objective
  - Detailed instructions for TSD generation
  - Complete TSD structure (13 sections)
  - Diagram rendering workflow
  - Azure Cosmos DB guidelines
  - Quality rules and constraints
  - Examples with tool call sequences
- **Version**: 2.0.0 (Deep Agent framework)

### 2. Configuration Updates

#### `src/core/config.py`
- Added `DeepAgentSettings` class with:
  - `CLOUD_STORAGE_PROVIDER` (azure | s3 | filesystem)
  - `WORKSPACE_ROOT` for local filesystem backend
  - Azure Blob settings (`SKILL_BLOB_CONTAINER`, `ARTIFACT_BLOB_CONTAINER`)
  - AWS S3 settings (bucket, credentials, region)
  - LLM configuration (`LLM_OPTION_NAME_UTILS_AGENTIC`)
  - Shell backend flag (`ENABLE_SHELL_BACKEND`)
- Integrated as `settings.deep_agent`

#### `.env.example`
- Added Deep Agent framework settings:
  - `WORKSPACE_ROOT=./workspace`
  - `CLOUD_STORAGE_PROVIDER=azure`
  - `LLM_MODEL=claude-sonnet-4-20250514`
  - `ANTHROPIC_API_KEY=your-anthropic-api-key`
  - Azure Blob backend configuration
  - AWS S3 backend configuration (optional)

### 3. Worker Integration

#### `src/functions/workers/send_message_worker/__init__.py`
- Added feature flag: `USE_DEEP_AGENT` (default: true)
- Created `_run_deep_agent()` function:
  - Creates agent instance with workspace context
  - Invokes agent with `agent.ainvoke()`
  - Handles errors and returns formatted result
- Maintains backward compatibility with legacy `run_send_message()`
- **Rollback**: Set `USE_DEEP_AGENT=false` to revert to legacy service

### 4. Dependencies

#### `requirements.txt`
- Added Deep Agent framework dependencies:
  - `deepagents>=0.1.0`
  - `langgraph>=0.2.0`
  - `langgraph-checkpoint>=0.1.0`

## Architecture Overview

### Before (Monolithic Service)
```
HTTP Request → Intent Classification → architect_service.py
  ↓
LLMService.chat() → llm_router_adapter → common-adapters
  ↓
MongoDB (sessions, jobs, snapshots) + Azure Blob (artifacts, skills)
```

### After (Deep Agent Framework)
```
HTTP Request → Queue → Worker calls create_architect_agent()
  ↓
agent.ainvoke(input) → DeepAgent with DESIGN_SKILLS
  ↓
Tools (workflow_context, design_snapshot, mermaid_render) + LLM
  ↓
MongoDB (sessions, jobs) + AzureBlobBackend (skills, artifacts)
```

## Key Features

### 1. Single Architect Agent
- One agent with DESIGN_SKILLS following reference pattern
- No subagents (as requested for isolated agent architecture)
- Workspace and connection context passed to tools

### 2. Skill-Driven Architecture
- Skills defined in `.md` files (not JSON)
- Virtual path mapping: `/skills/DESIGN_ARCHITECT_SKILL.md`
- Loaded from Azure Blob Storage in production
- Fallback to filesystem for local development

### 3. Backend Polymorphism
- Azure Blob: `CLOUD_STORAGE_PROVIDER=azure`
- AWS S3: `CLOUD_STORAGE_PROVIDER=s3`
- Filesystem: `CLOUD_STORAGE_PROVIDER=filesystem`

### 4. Tool-Based Workflow
- LangChain @tool decorated functions
- Workspace-scoped closures (no global state)
- Reuses existing services (MongoDB, Blob, Mermaid)

### 5. Backward Compatibility
- API contracts unchanged (POST /send-message, GET /send-message-status)
- MongoDB collections unchanged
- Azure Queue pattern preserved
- Feature flag for rollback

## Configuration Required

### Environment Variables (.env)

```bash
# Deep Agent Framework
CLOUD_STORAGE_PROVIDER=azure
WORKSPACE_ROOT=./workspaces

# Azure Blob Storage
AZURE_BLOB_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;...
SKILL_BLOB_CONTAINER=skills
ARTIFACT_BLOB_CONTAINER=artifacts

# LLM Configuration
LLM_OPTION_NAME_UTILS_AGENTIC=azure-openai
DEFAULT_WORKSPACE_ID=1
DEFAULT_AGENT_ID=1

# Feature Flag
USE_DEEP_AGENT=true
```

### Azure Blob Storage Setup

1. Create `skills` container in your storage account
2. Upload `DESIGN_ARCHITECT_SKILL.md` to `/skills/DESIGN_ARCHITECT_SKILL.md`
3. Ensure connection string has read permissions

## Testing

### Local Development
1. Set `CLOUD_STORAGE_PROVIDER=filesystem`
2. Skill file automatically loaded from `src/skills/DESIGN_ARCHITECT_SKILL.md`
3. Set `USE_DEEP_AGENT=true`
4. Test with POST /send-message endpoint

### Azure Production
1. Set `CLOUD_STORAGE_PROVIDER=azure`
2. Ensure skills uploaded to Azure Blob
3. Verify `AZURE_BLOB_STORAGE_CONNECTION_STRING` is correct
4. Monitor logs for "Creating Architect Deep Agent"

## Rollback Plan

### Immediate Rollback (< 1 hour)
1. Set `USE_DEEP_AGENT=false` in .env
2. Redeploy Azure Function
3. Worker reverts to `run_send_message()` legacy path

### Complete Rollback
1. Remove deepagents from requirements.txt
2. Remove Deep Agent files (agent_builder.py, agent_tools.py, etc.)
3. Remove DeepAgentSettings from config.py
4. Redeploy

## Files Changed

### New Files
- `src/agent_builder.py`
- `src/agent_prompts.py`
- `src/agent_tools.py`
- `src/agent_memory_templates.py`
- `src/skills/DESIGN_ARCHITECT_SKILL.md`
- `DEEP_AGENT_MIGRATION.md` (this file)

### Modified Files
- `requirements.txt` - Added deepagents dependencies
- `.env.example` - Added Deep Agent configuration
- `src/core/config.py` - Added DeepAgentSettings
- `src/functions/workers/send_message_worker/__init__.py` - Integrated Deep Agent

### Unchanged Files (Backward Compatible)
- `function_app.py` - No changes needed
- `src/services/architect_service.py` - Kept as fallback
- `src/services/*` - All services reused by tools
- MongoDB collections - No schema changes
- API endpoints - All contracts preserved

## Next Steps

1. **Deploy to Staging**
   - Test full workflow with real data
   - Validate skill loading from Azure Blob
   - Monitor performance metrics

2. **Performance Testing**
   - Measure agent creation time
   - Measure tool invocation overhead
   - Compare with legacy implementation

3. **Gradual Rollout**
   - Enable for 10% of traffic
   - Monitor error rates and latency
   - Increase to 50%, then 100%

4. **Deprecate Legacy**
   - After 2 weeks of stable operation
   - Remove `architect_service.run_send_message()`
   - Remove intent classification

## Benefits

1. **Modularity**: Skills defined in .md files, easy to update without code changes
2. **Testability**: Tools can be unit tested independently
3. **Scalability**: Backend polymorphism supports multi-cloud
4. **Maintainability**: Clear separation of concerns (tools, prompts, skills)
5. **Extensibility**: Easy to add new tools or subagents in future
6. **Isolation**: Single agent architecture ready for orchestrator integration

## Orchestrator Integration

The Architect agent is now designed as an isolated agent that can be called by an orchestrator via Azure Function API:

**Request Flow**:
```
Orchestrator → POST /send-message → Azure Queue → Deep Agent Worker → Result
```

**Response**:
```
Orchestrator → GET /send-message-status?job_id=X → Job result
```

This architecture supports the user's goal of creating 4 isolated agents (Architect, Dev, and 2 others) that can be orchestrated independently.

---

## Summary

The Deep Agent architecture migration is complete with:
- ✅ Single Architect agent with DESIGN_SKILLS
- ✅ Tool-based workflow integration
- ✅ Configuration migrated to .env
- ✅ Backend selection (Azure Blob/S3/Filesystem)
- ✅ Backward compatible with feature flag
- ✅ Ready for orchestrator integration

The implementation follows the reference pattern from `FE_Deep_AGENT_v1/backend/agent_builder.py` and maintains all existing API contracts while enabling the Deep Agent framework architecture.



================================================
FILE: Dockerfile
================================================
FROM python:3.11-slim

WORKDIR /app

# Install required OS packages
RUN apt-get update && apt-get install -y \
    git \
    gcc \
    g++ \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# GitHub PAT for private package access
ARG GH_PAT_READ
RUN git config --global url."https://${GH_PAT_READ}@github.com/".insteadOf "https://github.com/"

# Copy dependency files first for Docker layer caching
COPY requirements.txt .
COPY pyproject.toml .

# Upgrade pip
RUN pip install --upgrade pip setuptools wheel

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source
COPY . .

# Application port
EXPOSE 8000

# Start application
CMD ["python", "main.py"]



================================================
FILE: enable_file_logging.py
================================================
"""
Add file logging to capture debug output.

Run this before starting your app to enable file logging.
"""
import logging
import sys
from datetime import datetime

# Create log file with timestamp
log_file = f"dev_debug_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"

# Configure logging to both file and console
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(log_file),
        logging.StreamHandler(sys.stdout)
    ]
)

print(f"✓ File logging enabled: {log_file}")
print(f"✓ All logs will be written to this file")
print(f"✓ Now start your application")



================================================
FILE: FASTAPI_RUN_GUIDE.md
================================================
# FastAPI - Architect Deep Agent Run Guide

## Overview

The Architect Deep Agent now runs as a **FastAPI application** with Uvicorn instead of Azure Functions. This provides:

- ✅ **Swagger UI** at `/docs`
- ✅ **ReDoc documentation** at `/redoc`
- ✅ **OpenAPI schema** at `/openapi.json`
- ✅ **Fast development** with auto-reload
- ✅ **Standard REST API** (no Azure Functions dependency)

---

## Quick Start (3 Steps)

### 1. Install Dependencies
```bash
cd "d:\OneDrive - Coforge Limited\Desktop\Deep_Agent_Forgex\forgex-Architect-deepagent\Architect"

# Activate virtual environment
venv\Scripts\activate

# Install/upgrade dependencies
pip install -r requirements.txt
```

### 2. Configure Environment
Your `.env` is already configured! For local testing:
```bash
# In .env (already set):
CLOUD_STORAGE_PROVIDER=filesystem
USE_DEEP_AGENT=true
HOST=0.0.0.0
PORT=8000
```

### 3. Run the Server
```bash
# Option 1: Direct Python (recommended for development)
python main.py

# Option 2: Uvicorn command
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Option 3: Production mode (no reload)
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

Server starts at: **http://localhost:8000**

---

## 🎨 Swagger UI

Open your browser to explore the API interactively:

**Swagger UI:** http://localhost:8000/docs

You'll see all endpoints with:
- Request/response models
- Try it out functionality
- Example requests
- Authentication options

**ReDoc:** http://localhost:8000/redoc

Alternative documentation with better layout for reading.

---

## 📡 API Endpoints

### Health Check
```bash
curl http://localhost:8000/api/health
```

Response:
```json
{
  "status": "healthy",
  "service": "architect-deep-agent",
  "version": "2.0.0"
}
```

### Root Endpoint
```bash
curl http://localhost:8000/
```

Response:
```json
{
  "service": "Architect Deep Agent API",
  "version": "2.0.0",
  "status": "running",
  "docs": "/docs",
  "redoc": "/redoc"
}
```

### Create Session
```bash
curl -X POST "http://localhost:8000/api/create-session" \
  -H "Content-Type: application/json" \
  -d '{
    "workspace_id": "1",
    "user_id": "test_user",
    "title": "API Design Session"
  }'
```

### Send Message (Deep Agent)
```bash
curl -X POST "http://localhost:8000/api/send-message" \
  -H "Content-Type: application/json" \
  -d '{
    "workspace_id": "1",
    "user_id": "test_user",
    "conversation_id": "conv_001",
    "user_message": "Generate a TSD for a REST API service for user profile management using Azure, FastAPI, and Cosmos DB. Include authentication, CRUD operations, and proper error handling.",
    "agent_id": 1
  }'
```

Response (202 Accepted):
```json
{
  "job_id": "job_abc123def456",
  "status": "queued",
  "message": "Job queued successfully. Use /send-message-status to check progress."
}
```

### Check Job Status
```bash
curl "http://localhost:8000/api/send-message-status?job_id=job_abc123def456"
```

Responses:

**Queued:**
```json
{
  "job_id": "job_abc123def456",
  "status": "queued",
  "progress": 0,
  "created_at": "2026-08-19T15:30:00"
}
```

**Running:**
```json
{
  "job_id": "job_abc123def456",
  "status": "running",
  "progress": 50,
  "created_at": "2026-08-19T15:30:00",
  "updated_at": "2026-08-19T15:30:15"
}
```

**Completed:**
```json
{
  "job_id": "job_abc123def456",
  "status": "completed",
  "result": {
    "status": "success",
    "role": "assistant",
    "content": "# Technical Specification Document\n## User Profile Management API\n\n### 1. Executive Summary\n..."
  },
  "created_at": "2026-08-19T15:30:00",
  "updated_at": "2026-08-19T15:32:00"
}
```

### Cancel Job
```bash
curl -X POST "http://localhost:8000/api/cancel-job?job_id=job_abc123def456"
```

### Get Conversation History
```bash
curl "http://localhost:8000/api/get-conversation?workspace_id=1&user_id=test_user&session_id=conv_001&limit=50"
```

### Render Mermaid Diagram
```bash
curl -X POST "http://localhost:8000/api/render-mermaid" \
  -H "Content-Type: application/json" \
  -d '{
    "mermaid_code": "graph TD\n    A[Start] --> B[Process]\n    B --> C[End]"
  }'
```

---

## 🧪 Testing with Swagger UI

1. **Open Swagger:** http://localhost:8000/docs
2. **Click on endpoint** (e.g., `/api/send-message`)
3. **Click "Try it out"**
4. **Fill in the request body:**
   ```json
   {
     "workspace_id": "1",
     "user_id": "test_user",
     "conversation_id": "test_conv",
     "user_message": "Generate a simple API design",
     "agent_id": 1
   }
   ```
5. **Click "Execute"**
6. **Copy the `job_id` from response**
7. **Go to `/api/send-message-status` endpoint**
8. **Enter the `job_id` and click Execute**
9. **Keep polling until status is "completed"**
10. **View the generated TSD in the `result.content` field**

---

## 🔧 Development Mode

### Auto-Reload
```bash
# FastAPI auto-reloads on code changes
python main.py
# or
uvicorn main:app --reload
```

When you edit any `.py` file, the server automatically restarts.

### Debug Mode
```bash
# In .env
DEBUG=true
LOG_LEVEL=DEBUG

# Restart server
python main.py
```

You'll see detailed logs including:
- Deep Agent creation
- Tool executions
- LLM requests
- Database queries

### Custom Port
```bash
# In .env
PORT=9000

# Or via command line
uvicorn main:app --port 9000
```

---

## 🚀 Production Deployment

### Option 1: Gunicorn + Uvicorn Workers
```bash
pip install gunicorn

gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --timeout 120 \
  --access-logfile - \
  --error-logfile -
```

### Option 2: Docker
Create `Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

Build and run:
```bash
docker build -t architect-agent .
docker run -p 8000:8000 --env-file .env architect-agent
```

### Option 3: Azure App Service
```bash
# Deploy to Azure App Service (Linux)
az webapp up \
  --name architect-deep-agent \
  --resource-group forgex-rg \
  --runtime "PYTHON:3.11" \
  --sku B1 \
  --location eastus
```

---

## 📊 Performance Tuning

### Workers
```bash
# Calculate optimal workers: (2 × CPU cores) + 1
# For 4 cores: 9 workers
uvicorn main:app --workers 9
```

### Timeout
```bash
# For long-running agent tasks
uvicorn main:app --timeout-keep-alive 300
```

### Concurrency
```bash
# Use Gunicorn for better concurrency
gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --worker-connections 1000
```

---

## 🐛 Troubleshooting

### Port Already in Use
```bash
# Find process using port 8000
netstat -ano | findstr :8000

# Kill process (Windows)
taskkill /PID <PID> /F

# Or use different port
uvicorn main:app --port 9000
```

### Module Not Found
```bash
# Reinstall dependencies
pip install -r requirements.txt --force-reinstall
```

### Database Connection Error
```bash
# Check MongoDB URI
# In .env, verify:
MONGODB_DATABASE_URI=mongodb+srv://...

# Test connection
python -c "from motor.motor_asyncio import AsyncIOMotorClient; import asyncio; asyncio.run(AsyncIOMotorClient('your-uri').admin.command('ping'))"
```

### Agent Returns Empty Output
```bash
# Check LLM configuration
# Verify in .env:
ANTHROPIC_FOUNDRY_API_KEY=...
ANTHROPIC_DEFAULT_SONNET_MODEL=claude-sonnet-4-5-forgex-rnd

# Enable debug logging
LOG_LEVEL=DEBUG
```

---

## 📋 API Routes Summary

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Root endpoint with API info |
| `/docs` | GET | Swagger UI documentation |
| `/redoc` | GET | ReDoc documentation |
| `/openapi.json` | GET | OpenAPI schema |
| `/api/health` | GET | Health check |
| `/api/create-session` | POST | Create conversation session |
| `/api/send-message` | POST | Send message to Deep Agent (TSD generation) |
| `/api/send-message-status` | GET/POST | Check job status |
| `/api/cancel-job` | POST | Cancel running job |
| `/api/get-conversation` | GET/POST | Get conversation history |
| `/api/store-conversation` | POST | Store conversation message |
| `/api/delete-conversation` | POST/DELETE | Delete conversation |
| `/api/get-synopsis` | GET | Get session title |
| `/api/edit-synopsis` | POST | Edit session title |
| `/api/render-mermaid` | POST/GET | Render Mermaid diagram |
| `/api/get-skill` | GET | Get skill configuration |
| `/api/upload-skill` | POST | Upload skill file |
| `/api/download-skill` | GET | Download skill file |

---

## 🎯 Key Differences from Azure Functions

| Feature | Azure Functions | FastAPI |
|---------|----------------|---------|
| **Server** | Azure Functions runtime | Uvicorn ASGI server |
| **Docs** | None | Swagger UI + ReDoc |
| **Queue** | Azure Storage Queue | Background Tasks (asyncio) |
| **Deployment** | Azure-specific | Any platform (Docker, VPS, cloud) |
| **Development** | `func start` | `python main.py` |
| **Auto-reload** | Limited | Full support |
| **Testing** | Complex | Simple with `/docs` |
| **Cost** | Azure consumption | Any hosting |

---

## ✨ What's Running

When you start the server:

1. **FastAPI application** initializes
2. **MongoDB connection** established
3. **Deep Agent framework** loaded with:
   - Anthropic Claude Sonnet 4.5
   - Skills from `src/skills/DESIGN_ARCHITECT_SKILL.md`
   - 6 tools (workflow context, design snapshot, mermaid, etc.)
   - Filesystem backend (local) or Azure Blob (production)
4. **Background task queue** ready for async jobs
5. **Swagger UI** available for interactive testing

---

## 🎉 Success!

Your Architect Deep Agent is now running as a FastAPI application with full Swagger documentation!

- **API**: http://localhost:8000
- **Swagger**: http://localhost:8000/docs
- **ReDoc**: http://localhost:8000/redoc

Happy architecting! 🏗️✨



================================================
FILE: main.py
================================================
"""
FastAPI Application for Dev Deep Agent
Replaces Azure Functions with standard REST API using Uvicorn
"""
import logging
import os
import sys
from pathlib import Path
from contextlib import asynccontextmanager
from dotenv import load_dotenv

# Load .env file first (before any imports that use settings)
env_path = Path(__file__).parent / ".env"
if env_path.exists():
    load_dotenv(env_path, override=True)
    print(f"✓ Loaded .env file from {env_path}")
else:
    print(f"⚠ Warning: .env file not found at {env_path}")

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse

# Add src to path for imports
sys.path.insert(0, str(Path(__file__).parent / "src"))

from core.config import settings
from core.database import init_db, close_db
from core.postgres import init_pg, close_pg
from api.routers import (
    health,
    session,
    conversation,
    dev,
    mermaid,
    skills,
    registry,
    integrations,
)

# Set up logging
logging.basicConfig(
    level=getattr(logging, settings.LOG_LEVEL),
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Lifespan context manager for startup and shutdown events."""
    # Startup
    logger.info("Starting Dev Deep Agent API...")
    await init_db()
    logger.info(f"MongoDB connected: {settings.database.DEV_DB_NAME}")
    await init_pg()
    logger.info(f"PostgreSQL target: {settings.postgres.POSTGRESQL_DATABASE_DATABASE}")
    logger.info(f"Deep Agent mode: {settings.deep_agent.CLOUD_STORAGE_PROVIDER}")

    yield

    # Shutdown
    logger.info("Shutting down Dev Deep Agent API...")
    await close_pg()
    await close_db()
    logger.info("Shutdown complete")


# Create FastAPI app
app = FastAPI(
    title="Dev-Dynamic Deep Agent API",
    description="REST API for TSD-driven code scaffolding generation using the Deep Agent Framework",
    version="2.0.0",
    docs_url="/docs",  # Swagger UI
    redoc_url="/redoc",  # ReDoc UI
    openapi_url="/openapi.json",
    lifespan=lifespan,
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.security.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=settings.security.CORS_METHODS,
    allow_headers=["*"],
    # X-Job-Id is returned by POST /api/send-message/stream so the browser can
    # target /cancel-job; cross-origin JS can only read it if it is exposed.
    expose_headers=["X-Job-Id"],
)


# Exception handlers
@app.exception_handler(Exception)
async def global_exception_handler(request, exc):
    """Global exception handler."""
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "message": str(exc) if settings.DEBUG else "An error occurred",
        },
    )


# Include routers
app.include_router(health.router, prefix="/api", tags=["Health"])
app.include_router(session.router, prefix="/api", tags=["Session Management"])
app.include_router(conversation.router, prefix="/api", tags=["Conversation"])
app.include_router(dev.router, prefix="/api", tags=["Dev Agent"])
app.include_router(mermaid.router, prefix="/api", tags=["Mermaid Rendering"])
app.include_router(skills.router, prefix="/api", tags=["Skills Management"])
app.include_router(registry.router, prefix="/api", tags=["Registry"])
app.include_router(integrations.router, prefix="/api", tags=["Integrations"])

# WebSocket chat stream for the UI (token + tool-call streaming).
# Mounted at app root so it matches the frontend's `/ws/{session_id}` path.
app.add_api_websocket_route("/ws/{session_id}", dev.dev_ws)


@app.get("/")
async def root():
    """Root endpoint."""
    return {
        "service": "Dev Deep Agent API",
        "version": "2.0.0",
        "status": "running",
        "docs": "/docs",
        "redoc": "/redoc",
    }


# ── Session workspace inputs/outputs (root-level, consumed directly by the UI) ──
from typing import List  # noqa: E402
from fastapi import UploadFile, File  # noqa: E402
from services import workspace_tools  # noqa: E402


@app.post("/session/{session_id}/inputs")
async def upload_session_inputs(session_id: str, files: List[UploadFile] = File(...)):
    """Upload one or more intake documents (BRD, requirements, specs) for a session.

    Saved under WORKSPACE_ROOT/<session_id>/inputs/ where the agent can read them
    via its list_inputs() / read_input() tools.
    """
    workspace_tools.init_workspace(session_id)
    saved = []
    for f in files:
        data = await f.read()
        # Parse binary formats (.docx/.pdf) into readable Markdown before saving
        # so the agent never receives raw binary in the intake folder.
        saved.append(workspace_tools.save_intake_document(session_id, f.filename, data))
    return {"uploaded": saved, "all_inputs": workspace_tools.list_input_files(session_id)}


@app.get("/session/{session_id}/inputs")
async def list_session_inputs(session_id: str):
    """List uploaded intake documents for a session."""
    return {"files": workspace_tools.list_input_files(session_id)}


@app.get("/session/{session_id}/inputs/{filepath:path}")
async def get_session_input(session_id: str, filepath: str):
    """Return the content of a single uploaded intake document.

    Reads through the active backend (local disk / Azure blob / S3) so the same
    endpoint works regardless of CLOUD_STORAGE_PROVIDER.
    """
    if ".." in Path(filepath).parts:
        return JSONResponse(status_code=400, content={"error": "Invalid path"})
    content = workspace_tools.read_input_file(session_id, filepath)
    if content is None:
        return JSONResponse(status_code=404, content={"error": f"Not found: {filepath}"})
    return {"filename": filepath, "content": content}


@app.get("/session/{session_id}/outputs")
async def list_session_outputs(session_id: str):
    """List generated documents for a session (used by the UI side panel)."""
    return {"files": workspace_tools.list_output_files(session_id)}


@app.get("/session/{session_id}/outputs/{filepath:path}")
async def get_session_output(session_id: str, filepath: str):
    """Return the content of a single generated document.

    Reads through the active backend (local disk / Azure blob / S3) so the same
    endpoint works regardless of CLOUD_STORAGE_PROVIDER.
    """
    if ".." in Path(filepath).parts:
        return JSONResponse(status_code=400, content={"error": "Invalid path"})
    content = workspace_tools.read_output_file(session_id, filepath)
    if content is None:
        return JSONResponse(status_code=404, content={"error": f"Not found: {filepath}"})
    return {"filename": filepath, "content": content}


if __name__ == "__main__":
    import uvicorn

    # Reload is controlled by the dedicated RELOAD flag, NOT DEBUG. On Windows the
    # uvicorn auto-reloader tears the subprocess down with `WinError 10038`
    # (operation on a non-socket) during loop shutdown, which also kills any
    # in-flight SSE generation mid-stream (truncated document). It therefore
    # defaults OFF; enable it explicitly with RELOAD=true on non-Windows dev.
    #
    # When on, watch ONLY the source tree — otherwise the reloader also watches
    # WORKSPACE_ROOT=./workspace and every intake/save_output write the agent
    # makes triggers a restart that kills the running generation.
    reload_kwargs = {}
    if settings.RELOAD:
        reload_kwargs = {
            "reload_dirs": ["src"],
            "reload_excludes": ["workspace/*", "workspaces/*", "*.md", "*.txt"],
        }

    uvicorn.run(
        "main:app",
        host=settings.HOST,
        port=settings.PORT,
        reload=settings.RELOAD,
        log_level=settings.LOG_LEVEL.lower(),
        **reload_kwargs,
    )



================================================
FILE: pyproject.toml
================================================
[project]
name = "arch-rest-v2"
version = "0.1.0"
description = "Azure Functions API for forgeX"
requires-python = ">=3.11"
dependencies = [
    "azure-functions",
    "pydantic>=2.0.0",
    "python-dotenv",
    "requests",
    "azure-identity",
    "azure-keyvault-secrets",
    "structlog",
    "azure-cosmos",
    "azure-storage-blob",
    # Optional cache for skills runtime (best-effort; code works without Redis).
    "redis>=5,<6",
    "PyJWT",
    "pymongo>=4.9,<5",
    "motor>=3.6,<4",
    "certifi",
    # Dev agent integration (code vendored under src/dev/).
    "common-adapters @ git+https://github.com/Coforge-forgeX/forgexpackages.git@main",
    "python-docx",
    "pypdf>=4.0.0",
    "openai",
    "langchain",
    "langchain-community",
    "langchain-openai",
    "langchain-core",
    # MCP client for Dev MCP server (used by src/agent/mcp_client.py)
    "mcp",
]

[project.optional-dependencies]
dev = [
    "pre-commit",
    "black",
    "isort",
    "flake8",
    "mypy",
    "bandit",
    "pytest",
    "pytest-asyncio",
    "pytest-cov",
    "httpx",
    "faker",
    "types-requests",
]

[tool.black]
line-length = 88
target-version = ['py38', 'py39', 'py310', 'py311']
include = '\.pyi?$'
extend-exclude = '''
/(
  # directories
  \.eggs
  | \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | _build
  | buck-out
  | build
  | dist
  | __pycache__
)/
'''

[tool.isort]
profile = "black"
line_length = 88
multi_line_output = 3
include_trailing_comma = true
force_grid_wrap = 0
use_parentheses = true
ensure_newline_before_comments = true
src_paths = ["src", "tests"]
known_first_party = ["core", "agent", "dev", "services", "shared", "functions"]

[tool.mypy]
python_version = "3.8"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = false
disallow_incomplete_defs = false
check_untyped_defs = true
disallow_untyped_decorators = false
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true
ignore_missing_imports = true
strict_optional = false

[[tool.mypy.overrides]]
module = "azure.functions.*"
ignore_missing_imports = true

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[tool.pytest.ini_options]
minversion = "6.0"
addopts = "-ra -q --strict-markers --strict-config"
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
filterwarnings = [
    "ignore::DeprecationWarning",
    "ignore::PendingDeprecationWarning",
]

[tool.flake8]
max-line-length = 88
extend-ignore = ["E203", "W503", "E501"]
per-file-ignores = [
    "__init__.py:F401",
    "tests/*:S101",
]
exclude = [
    ".git",
    "__pycache__",
    ".venv",
    ".eggs",
    "*.egg",
    "build",
    "dist",
]

[tool.bandit]
exclude_dirs = ["tests", ".venv", "__pycache__"]
skips = ["B101", "B601"]



================================================
FILE: requirements.txt
================================================
# FastAPI and ASGI Server
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
starlette
typing-extensions
python-multipart
pydantic>=2.0.0
pydantic-settings
motor>=3.6,<4
asyncpg>=0.29
sqlalchemy
requests
azure-identity
azure-keyvault-secrets
structlog
azure-cosmos
azure-storage-blob
pymongo>=4.9,<5
certifi
playwright
websockets>=14,<16
aiohttp
PyJWT

# --- Deep Agent Framework ---
# Deep agent framework for building agent-based architectures
deepagents==0.6.0
langgraph>=0.2.0
langgraph-checkpoint>=0.1.0
# Durable LangGraph checkpointer (agent state) backed by MongoDB.
langgraph-checkpoint-mongodb>=0.1.0

# --- Dev agent integration (vendored under src/dev/) ---
# The agent code lives in-repo (src/dev/), so no agent_dev install is
# needed. These are the third-party deps that vendored code imports:
#   - common-adapters: the LLM Router used by dev/llm_router_adapter.py
#     (private git package; it pulls its own langchain/openai stack transitively).
#   - python-docx: used by dev/workflow_context.py to read DOCX artifacts.
# The langchain/openai pins are kept explicit as insurance in case common-adapters
# does not fully pin them.
common-adapters @ git+https://github.com/Coforge-forgeX/forgexpackages.git@main
python-docx>=1.2.0
pypdf>=4.0.0
openai>=1.0.0
langchain
langchain-community
langchain-openai
langchain-core
langchain-anthropic
sse-starlette
# Cloud storage backends (deep-agent FilesystemBackend alternatives)
wcmatch
aioboto3  # required only when CLOUD_STORAGE_PROVIDER=s3/aws
# azure-storage-blob already listed above (required when CLOUD_STORAGE_PROVIDER=azure)



================================================
FILE: run.bat
================================================
@echo off
echo ========================================
echo  Dev-Dynamic Deep Agent - FastAPI Server
echo ========================================
echo.

REM Activate virtual environment.
REM Prefer a local .\venv, otherwise fall back to the shared my_env two levels up
REM (Deep_Agent_Forgex\my_env), which is the working environment on this machine.
if exist venv\Scripts\activate.bat (
    echo Activating virtual environment: venv
    call venv\Scripts\activate.bat
) else if exist "..\..\my_env\Scripts\activate.bat" (
    echo Activating virtual environment: my_env
    call "..\..\my_env\Scripts\activate.bat"
) else (
    echo WARNING: Virtual environment not found!
    echo Please run: python -m venv venv
    echo.
)

REM Check if dependencies are installed
python -c "import fastapi" 2>nul
if errorlevel 1 (
    echo Installing dependencies...
    pip install -r requirements.txt
    echo.
)

REM Start the server
echo Starting FastAPI server...
echo Server will be available at: http://localhost:8001
echo Swagger UI: http://localhost:8001/docs
echo.
python main.py



================================================
FILE: RUN_GUIDE.md
================================================
# How to Run the Architect Deep Agent

This guide shows you how to run the Architect agent locally and deploy to Azure.

## Prerequisites

- Python 3.11 (as specified in your configuration)
- Azure Functions Core Tools
- Git
- Access to Azure resources (Blob Storage, MongoDB, etc.)

## 1. Installation

### Step 1: Clone the Repository (if not already done)
```bash
cd "d:\OneDrive - Coforge Limited\Desktop\Deep_Agent_Forgex\forgex-Architect-deepagent\Architect"
```

### Step 2: Create Virtual Environment
```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate

# On Linux/Mac:
source venv/bin/activate
```

### Step 3: Install Dependencies
```bash
pip install -r requirements.txt
```

This will install:
- Azure Functions SDK
- Deep Agent framework (`deepagents`, `langgraph`)
- LangChain and OpenAI/Anthropic SDKs
- MongoDB driver (motor)
- All other dependencies

## 2. Configuration

### Step 1: Verify .env File
Your `.env` file is already configured with:
- ✅ MongoDB connection
- ✅ Azure Blob Storage
- ✅ Anthropic Claude model
- ✅ Deep Agent settings

### Step 2: Upload Skills to Azure Blob (Optional for Azure deployment)

If using `CLOUD_STORAGE_PROVIDER=azure`, upload the skill file:

```bash
# Install Azure CLI (if not installed)
# Download from: https://aka.ms/installazurecli

# Login to Azure
az login

# Upload skill file to blob storage
az storage blob upload \
  --connection-string "$AZURE_BLOB_STORAGE_CONNECTION_STRING" \
  --container-name "skills" \
  --name "skills/DESIGN_ARCHITECT_SKILL.md" \
  --file "src/skills/DESIGN_ARCHITECT_SKILL.md"
```

**Or use Azure Storage Explorer** (GUI tool):
1. Download: https://azure.microsoft.com/en-us/products/storage/storage-explorer/
2. Connect with your connection string
3. Navigate to `skills` container
4. Upload `src/skills/DESIGN_ARCHITECT_SKILL.md` to path `/skills/DESIGN_ARCHITECT_SKILL.md`

### Step 3: Set Cloud Storage Provider

For **local development** (no Azure Blob needed):
```bash
# In .env file, change:
CLOUD_STORAGE_PROVIDER=filesystem
```

For **Azure deployment**:
```bash
# In .env file, use:
CLOUD_STORAGE_PROVIDER=azure
```

## 3. Running Locally

### Method 1: Azure Functions Core Tools (Recommended)

```bash
# Start the Azure Functions host
func start

# Or with specific port
func start --port 7071
```

This will start the server at: `http://localhost:7071`

You should see output like:
```
Functions:
        cancel_job: [POST] http://localhost:7071/api/cancel-job
        create_session: [POST] http://localhost:7071/api/create-session
        delete_conversation: [POST,DELETE] http://localhost:7071/api/delete-conversation
        edit_synopsis: [POST] http://localhost:7071/api/edit-synopsis
        get_architecture_zip_url: [GET,POST] http://localhost:7071/api/get-architecture-zip-url
        get_conversation: [GET,POST] http://localhost:7071/api/get-conversation
        get_synopsis: [GET,POST] http://localhost:7071/api/get-synopsis
        health_check: [GET] http://localhost:7071/api/health
        http_example: [GET,POST] http://localhost:7071/api/hello
        render_mermaid: [POST,GET] http://localhost:7071/api/render-mermaid
        send_message: [POST] http://localhost:7071/api/send-message
        send_message_status: [GET,POST] http://localhost:7071/api/send-message-status
        send_message_worker: queueTrigger
        send_message_worker_poison: queueTrigger
        store_conversation: [POST] http://localhost:7071/api/store-conversation
        update_design_hld: [POST] http://localhost:7071/api/update-design-hld
        update_design_hld_worker: queueTrigger
        update_design_hld_worker_poison: queueTrigger
```

### Method 2: Direct Python (for debugging)

```bash
# Run a specific function module directly (for testing)
python -m src.agent_builder
```

## 4. Testing the Deep Agent

### Test 1: Health Check
```bash
curl http://localhost:7071/api/health
```

Expected response:
```json
{
  "status": "healthy",
  "service": "arch-rest-api"
}
```

### Test 2: Create Session
```bash
curl -X POST http://localhost:7071/api/create-session \
  -H "Content-Type: application/json" \
  -d '{
    "workspace_id": "1",
    "user_id": "test_user"
  }'
```

Expected response:
```json
{
  "session_id": "some-uuid",
  "workspace_id": "1",
  "user_id": "test_user",
  "created_at": "2026-08-19T..."
}
```

### Test 3: Send Message (Deep Agent)

```bash
curl -X POST http://localhost:7071/api/send-message \
  -H "Content-Type: application/json" \
  -d '{
    "workspace_id": "1",
    "user_id": "test_user",
    "conversation_id": "test_conv_001",
    "user_message": "Generate a TSD for a simple REST API service for managing user profiles using Azure and FastAPI.",
    "agent_id": 1
  }'
```

Expected response (202 Accepted):
```json
{
  "job_id": "job_abc123",
  "status": "queued",
  "message": "Job queued successfully"
}
```

### Test 4: Check Job Status

```bash
# Use the job_id from previous response
curl http://localhost:7071/api/send-message-status?job_id=job_abc123
```

Expected responses:

**While processing:**
```json
{
  "job_id": "job_abc123",
  "status": "running",
  "progress": 50
}
```

**When complete:**
```json
{
  "job_id": "job_abc123",
  "status": "completed",
  "result": {
    "status": "success",
    "role": "assistant",
    "content": "# Technical Specification Document\n## User Profile Management API\n..."
  }
}
```

## 5. Monitoring and Debugging

### Check Logs

**Azure Functions logs** (console output):
```bash
# Running logs appear in terminal where you ran 'func start'
# Look for:
[INFO] Creating Architect Deep Agent
[INFO] Deep Agent completed successfully
```

**Python logging:**
```bash
# Logs go to console by default
# Check for Deep Agent specific logs:
grep "Deep Agent" logs.txt
```

### Debug Mode

Enable detailed logging:
```bash
# In .env
DEBUG=true
LOG_LEVEL=DEBUG
```

### Check Deep Agent Configuration

Create a test script `test_config.py`:
```python
from core.config import settings

print("=== Deep Agent Configuration ===")
print(f"Cloud Provider: {settings.deep_agent.CLOUD_STORAGE_PROVIDER}")
print(f"Workspace Root: {settings.deep_agent.WORKSPACE_ROOT}")
print(f"LLM Option: {settings.deep_agent.LLM_OPTION_NAME_UTILS_AGENTIC}")
print(f"Use Deep Agent: {os.getenv('USE_DEEP_AGENT', 'true')}")
print(f"Skill Blob Container: {settings.deep_agent.SKILL_BLOB_CONTAINER}")
print(f"Anthropic Model: {os.getenv('ANTHROPIC_DEFAULT_SONNET_MODEL')}")
print(f"MongoDB URI configured: {bool(settings.database.MONGODB_DATABASE_URI)}")
print(f"Blob Storage configured: {bool(settings.azure.BLOB_STORAGE_CONNECTION_STRING)}")
```

Run it:
```bash
python test_config.py
```

## 6. Troubleshooting

### Issue 1: Module Not Found Error
```
ModuleNotFoundError: No module named 'deepagents'
```

**Solution:**
```bash
pip install deepagents langgraph langgraph-checkpoint
```

### Issue 2: MongoDB Connection Error
```
pymongo.errors.ServerSelectionTimeoutError
```

**Solution:**
- Check `MONGODB_DATABASE_URI` in `.env`
- Verify MongoDB cluster is accessible
- Check firewall/network settings

### Issue 3: Azure Blob Storage Error
```
ResourceNotFoundError: The specified container does not exist
```

**Solution:**
- Create the `skills` container in your storage account
- Upload skill file to `/skills/DESIGN_ARCHITECT_SKILL.md`
- Or use `CLOUD_STORAGE_PROVIDER=filesystem` for local testing

### Issue 4: Agent Returns Empty Output
```
Agent returned empty output
```

**Solution:**
- Check LLM API key is valid
- Verify `ANTHROPIC_FOUNDRY_API_KEY` is correct
- Check `llm_config_factory` supports Anthropic
- Look for errors in logs

### Issue 5: Queue Message Not Processing
```
Job stuck in "queued" status
```

**Solution:**
- Check if worker function is running (should see in `func start` logs)
- Verify `AzureWebJobsStorage` is configured
- For local dev, use Azurite storage emulator:
  ```bash
  npm install -g azurite
  azurite --silent --location ./azurite
  ```

## 7. Deploy to Azure

### Step 1: Create Azure Function App

```bash
# Login to Azure
az login

# Create resource group (if not exists)
az group create --name forgex-rg --location eastus

# Create storage account (if not exists)
az storage account create \
  --name forgexstorage \
  --resource-group forgex-rg \
  --location eastus \
  --sku Standard_LRS

# Create Function App
az functionapp create \
  --name forgex-architect-agent \
  --resource-group forgex-rg \
  --storage-account forgexstorage \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux
```

### Step 2: Configure App Settings

```bash
# Set environment variables from .env file
az functionapp config appsettings set \
  --name forgex-architect-agent \
  --resource-group forgex-rg \
  --settings \
    MONGODB_DATABASE_URI="mongodb+srv://..." \
    ARCHITECT_DB_NAME="architect_chatbot_db" \
    AZURE_BLOB_STORAGE_CONNECTION_STRING="DefaultEndpointsProtocol=https;..." \
    ANTHROPIC_FOUNDRY_API_KEY="your-key" \
    ANTHROPIC_DEFAULT_SONNET_MODEL="claude-sonnet-4-5-forgex-rnd" \
    CLOUD_STORAGE_PROVIDER="azure" \
    USE_DEEP_AGENT="true"
```

### Step 3: Deploy

```bash
# Deploy from local directory
func azure functionapp publish forgex-architect-agent

# Or deploy with build on remote
func azure functionapp publish forgex-architect-agent --build remote
```

### Step 4: Verify Deployment

```bash
# Test health endpoint
curl https://forgex-architect-agent.azurewebsites.net/api/health

# Check logs
az webapp log tail \
  --name forgex-architect-agent \
  --resource-group forgex-rg
```

## 8. Production Checklist

Before going to production:

- [ ] **Security**
  - [ ] Change `JWT_SECRET_KEY` to a strong random value
  - [ ] Enable authentication on endpoints (change `auth_level` from `ANONYMOUS` to `FUNCTION`)
  - [ ] Use Azure Key Vault for secrets
  - [ ] Enable HTTPS only

- [ ] **Performance**
  - [ ] Set up Application Insights monitoring
  - [ ] Configure autoscaling
  - [ ] Enable Redis cache for skills
  - [ ] Optimize Cosmos DB RU/s

- [ ] **Reliability**
  - [ ] Set up Azure Service Bus for progress reporting
  - [ ] Configure retry policies
  - [ ] Set up health checks and alerts
  - [ ] Enable diagnostic logging

- [ ] **Deep Agent**
  - [ ] Verify skill files uploaded to Blob Storage
  - [ ] Test end-to-end TSD generation
  - [ ] Monitor agent creation time (<150ms target)
  - [ ] Validate tool execution (all 6 tools working)

## 9. Quick Start Summary

**Minimum steps to run locally:**

```bash
# 1. Activate virtual environment
venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Set local storage mode (in .env)
CLOUD_STORAGE_PROVIDER=filesystem

# 4. Start Azure Functions
func start

# 5. Test health
curl http://localhost:7071/api/health

# 6. Send message
curl -X POST http://localhost:7071/api/send-message \
  -H "Content-Type: application/json" \
  -d '{"workspace_id":"1","user_id":"test","conversation_id":"conv1","user_message":"Hello"}'
```

That's it! Your Deep Agent Architect is now running! 🚀

## Support

If you encounter issues:
1. Check logs in the console
2. Verify all environment variables are set
3. Ensure MongoDB and Azure Blob are accessible
4. Review [DEEP_AGENT_MIGRATION.md](DEEP_AGENT_MIGRATION.md) for architecture details
5. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for common issues

---

**Happy Architecting!** 🏗️✨



================================================
FILE: test_agent_invoke.py
================================================
"""Test agent invocation to find where it hangs."""
import asyncio
import sys
from pathlib import Path

# Add src to path
sys.path.insert(0, str(Path(__file__).parent / "src"))

from dotenv import load_dotenv
load_dotenv(Path(__file__).parent / ".env")

from agent_builder import create_dev_agent

async def test_agent():
    """Test agent with simple message."""
    print("=" * 60)
    print("Creating Dev Agent...")
    print("=" * 60)

    try:
        agent = create_dev_agent(
            workspace_id="test",
            user_id="test",
            conversation_id="test",
            agent_id=1,
            job_id=None,
        )
        print("[OK] Agent created successfully\n")

        print("=" * 60)
        print("Invoking agent with message: 'hi'")
        print("=" * 60)

        # Set a timeout for the invocation
        result = await asyncio.wait_for(
            agent.ainvoke({
                "input": "hi",
                "workspace_id": "test",
                "conversation_id": "test",
                "job_id": None,
            }),
            timeout=30.0  # 30 second timeout
        )

        print("\n[SUCCESS] Agent responded:")
        print("-" * 60)
        print(result.get("output", "No output"))
        print("-" * 60)

    except asyncio.TimeoutError:
        print("\n[ERROR] Agent invocation timed out after 30 seconds")
        print("The agent is hanging - likely stuck in a tool call or infinite loop")

    except Exception as e:
        print(f"\n[ERROR] {type(e).__name__}: {e}")
        import traceback
        traceback.print_exc()

if __name__ == "__main__":
    asyncio.run(test_agent())



================================================
FILE: .pre-commit-config.yaml
================================================
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: check-case-conflict
      - id: check-docstring-first
      - id: debug-statements

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3
        args: [--line-length=88]

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
        args: [--profile=black]

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff
        args: [--fix]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.3.0
    hooks:
      - id: mypy
        language_version: python3.11
        additional_dependencies:
          - types-requests
          - pydantic
          - typing-extensions
        args:
          - --python-version=3.11
          - --ignore-missing-imports
          - --no-strict-optional
          - --explicit-package-bases
          - --allow-untyped-defs
          - --allow-untyped-calls
          - --disable-error-code=no-any-return
        exclude: ^(tests/|migrations/)

  # - repo: https://github.com/pycqa/bandit
  #   rev: 1.7.5
  #   hooks:
  #     - id: bandit
  #       additional_dependencies: [pbr]
  #       args: [-r, ., --severity-level, medium]
  #       exclude: ^(\.venv/|tests/)
  #       pass_filenames: false
  #       # stages: [pre-push]

  # - repo: local
  #   hooks:
  #     - id: azure-functions-lint
  #       name: Azure Functions specific linting
  #       entry: python
  #       language: python
  #       args: [-m, pytest, --collect-only, -q]
  #       files: ^src/functions/
  #       pass_filenames: false



================================================
FILE: migrations/2026_08_27_fe_agent_document_run_scope.sql
================================================
-- Blob folder restructure: add run-scope columns to the document registry.
-- Backfills nothing (existing rows keep NULLs; their stored document_url stays
-- valid). The unique constraint remains (session_id, document_name).
--
-- Target DB: forgex_coforge (ForgeX Postgres).
-- Run once, e.g.:  psql "$PG_DSN" -f 2026_08_27_fe_agent_document_run_scope.sql

ALTER TABLE fe_agent_document
    ADD COLUMN IF NOT EXISTS workspace_id    text,
    ADD COLUMN IF NOT EXISTS workflow_job_id text,
    ADD COLUMN IF NOT EXISTS scope           text,
    ADD COLUMN IF NOT EXISTS agent_folder    text;

-- Lookup indexes for the new UI fetch paths (workflow-wide and per-agent).
CREATE INDEX IF NOT EXISTS ix_fe_agent_document_workflow_job_id
    ON fe_agent_document (workflow_job_id);
CREATE INDEX IF NOT EXISTS ix_fe_agent_document_workspace_agent
    ON fe_agent_document (workspace_id, agent_folder);



================================================
FILE: scripts/apply_run_scope_migration.py
================================================
"""Apply 2026_08_27_fe_agent_document_run_scope.sql to forgex_coforge.

Adds run-scope columns (workspace_id, workflow_job_id, scope, agent_folder)
plus lookup indexes. Idempotent. Then prints the resulting columns.
"""
import asyncio, socket, asyncpg

DDL = """
ALTER TABLE fe_agent_document
    ADD COLUMN IF NOT EXISTS workspace_id    text,
    ADD COLUMN IF NOT EXISTS workflow_job_id text,
    ADD COLUMN IF NOT EXISTS scope           text,
    ADD COLUMN IF NOT EXISTS agent_folder    text;
CREATE INDEX IF NOT EXISTS ix_fe_agent_document_workflow_job_id
    ON fe_agent_document (workflow_job_id);
CREATE INDEX IF NOT EXISTS ix_fe_agent_document_workspace_agent
    ON fe_agent_document (workspace_id, agent_folder);
"""


async def main():
    host = socket.gethostbyname("forgexpostgresql.postgres.database.azure.com")
    c = await asyncpg.connect(host=host, port=5432, user="forgeX",
        password="Postgre@f0rge-X2025", database="forgex_coforge", ssl="require")
    try:
        await c.execute(DDL)
        cols = await c.fetch("""
            SELECT column_name FROM information_schema.columns
            WHERE table_name='fe_agent_document' ORDER BY ordinal_position""")
        print("columns:", [r["column_name"] for r in cols])
        n = await c.fetchval("SELECT count(*) FROM fe_agent_document")
        print("row count:", n)
    finally:
        await c.close()


asyncio.run(main())



================================================
FILE: scripts/create_fe_agent_document.py
================================================
"""One-off DDL: create the fe_agent_document table in forgex_coforge.

Matches the schema used by services/documents.py (upsert relies on the
UNIQUE (session_id, document_name) constraint). Idempotent — safe to re-run.
"""
import asyncio
import socket

import asyncpg

HOST = "forgexpostgresql.postgres.database.azure.com"
PORT = 5432
DB = "forgex_coforge"
USER = "forgeX"
PASSWORD = "Postgre@f0rge-X2025"

DDL = """
CREATE TABLE IF NOT EXISTS fe_agent_document (
    id            BIGSERIAL PRIMARY KEY,
    session_id    TEXT        NOT NULL,
    document_name TEXT        NOT NULL,
    document_url  TEXT        NOT NULL,
    created_by    TEXT        NOT NULL DEFAULT 'System',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_fe_agent_document_session_name UNIQUE (session_id, document_name)
);
CREATE INDEX IF NOT EXISTS ix_fe_agent_document_session_id
    ON fe_agent_document (session_id);
"""


async def main():
    host = socket.gethostbyname(HOST)
    conn = await asyncpg.connect(
        host=host, port=PORT, user=USER, password=PASSWORD, database=DB, ssl="require"
    )
    try:
        await conn.execute(DDL)
        print("fe_agent_document ready in", DB)
    finally:
        await conn.close()


if __name__ == "__main__":
    asyncio.run(main())



================================================
FILE: scripts/progress_relay.py
================================================
"""
Local Progress Relay Service for Dev Agent (Development).

A lightweight WebSocket server that:
1. Receives progress events via HTTP POST (/publish)
2. Routes events to connected WebSocket clients by conversation_id/job_id
3. Provides health check endpoint

Usage:
    python -m uvicorn scripts.progress_relay:app --host 127.0.0.1 --port 8090

WebSocket client connection:
    ws://127.0.0.1:8090/ws?channel=<conversation_id_or_job_id>

Publish progress event:
    POST http://127.0.0.1:8090/publish
    {
      "operation": "send_message",
      "status": "running",
      "conversation_id": "conv_123",
      ...
    }
"""

import asyncio
import json
from datetime import datetime
from typing import Dict, Set

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI(title="Dev Progress Relay (Local)")


# WebSocket connection manager
class ConnectionManager:
    """Manages WebSocket connections and message routing."""

    def __init__(self):
        # Map channel -> set of websockets
        self.active_connections: Dict[str, Set[WebSocket]] = {}

    async def connect(self, websocket: WebSocket, channel: str):
        """Accept and register a WebSocket connection."""
        await websocket.accept()
        if channel not in self.active_connections:
            self.active_connections[channel] = set()
        self.active_connections[channel].add(websocket)
        print(f"[CONNECT] Channel: {channel}, Total: {len(self.active_connections[channel])}")

    def disconnect(self, websocket: WebSocket, channel: str):
        """Remove a WebSocket connection."""
        if channel in self.active_connections:
            self.active_connections[channel].discard(websocket)
            if not self.active_connections[channel]:
                del self.active_connections[channel]
        print(f"[DISCONNECT] Channel: {channel}")

    async def send_to_channel(self, channel: str, message: dict):
        """Send a message to all connections on a channel."""
        if channel not in self.active_connections:
            print(f"[SEND] No connections for channel: {channel}")
            return

        # Prepare message envelope
        envelope = {
            "type": "progress",
            "channel": channel,
            "event": message,
            "timestamp": datetime.utcnow().isoformat(),
        }

        dead_connections = []
        for websocket in self.active_connections[channel]:
            try:
                await websocket.send_json(envelope)
            except Exception as e:
                print(f"[ERROR] Failed to send to websocket: {e}")
                dead_connections.append(websocket)

        # Clean up dead connections
        for ws in dead_connections:
            self.disconnect(ws, channel)


manager = ConnectionManager()


# Progress event model
class ProgressEvent(BaseModel):
    """Progress event payload."""

    operation: str
    status: str
    message: str = ""
    user_id: str = None
    conversation_id: str = None
    job_id: str = None
    correlation_id: str = None
    provider: str = "dev"
    metadata: dict = {}


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, channel: str = None, session_id: str = None):
    """
    WebSocket endpoint for real-time progress updates.

    Query params:
        channel: conversation_id or job_id to subscribe to
        session_id: alias for channel (backward compatibility)
    """
    # Resolve channel
    channel_id = channel or session_id
    if not channel_id:
        await websocket.close(code=1008, reason="Missing channel or session_id parameter")
        return

    await manager.connect(websocket, channel_id)

    try:
        # Keep connection alive
        while True:
            # Wait for client messages (ping/pong, etc.)
            data = await websocket.receive_text()
            # Echo back for ping/pong
            if data == "ping":
                await websocket.send_text("pong")
    except WebSocketDisconnect:
        manager.disconnect(websocket, channel_id)
    except Exception as e:
        print(f"[WS_ERROR] {e}")
        manager.disconnect(websocket, channel_id)


@app.post("/publish")
async def publish_progress(event: ProgressEvent):
    """
    Publish a progress event to all subscribed WebSocket clients.

    The event is routed to clients subscribed to:
    - job_id (if present, preferred)
    - conversation_id (fallback)
    """
    # Route to job_id (preferred) or conversation_id
    channels = []
    if event.job_id:
        channels.append(event.job_id)
    if event.conversation_id:
        channels.append(event.conversation_id)

    if not channels:
        return JSONResponse(
            {"success": False, "error": "Event must have job_id or conversation_id"},
            status_code=400,
        )

    # Send to all relevant channels
    event_dict = event.dict()
    for channel in channels:
        await manager.send_to_channel(channel, event_dict)

    return JSONResponse(
        {
            "success": True,
            "channels": channels,
            "active_connections": sum(len(conns) for conns in manager.active_connections.values()),
        }
    )


@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return JSONResponse(
        {
            "status": "healthy",
            "service": "dev-progress-relay",
            "active_channels": len(manager.active_connections),
            "total_connections": sum(len(conns) for conns in manager.active_connections.values()),
        }
    )


@app.get("/")
async def root():
    """Root endpoint with service info."""
    return JSONResponse(
        {
            "service": "Dev Progress Relay (Local Development)",
            "endpoints": {
                "websocket": "ws://127.0.0.1:8090/ws?channel=<conversation_id>",
                "publish": "POST /publish",
                "health": "GET /health",
            },
        }
    )


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="127.0.0.1", port=8090)



================================================
FILE: scripts/servicebus_relay.py
================================================
"""
Azure Service Bus Progress Relay for Dev Agent (Production).

Consumes progress events from Azure Service Bus and broadcasts to WebSocket clients.

Architecture:
    Azure Functions → Service Bus Topic → This Relay → WebSocket → Browser

Usage:
    python scripts/servicebus_relay.py

Configuration (environment variables):
    - SERVICE_BUS_CONNECTION_STRING: Azure Service Bus connection string
    - PROGRESS_TOPIC: Service Bus topic name (default: "dev-progress")
    - RELAY_SUBSCRIPTION_NAME: Subscription name (default: "relay-sub")
    - RELAY_PORT: WebSocket server port (default: 8092)
    - BROADCAST_MODE: "session" or "tab" (default: "session")
"""

import asyncio
import json
import os
from datetime import datetime
from typing import Dict, Set

from azure.servicebus.aio import ServiceBusClient
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import JSONResponse

# Load local.settings.json Values when running locally (best-effort).
try:
    from scripts.load_local_settings_env import apply_env, load_values

    apply_env(load_values())
except Exception:
    pass

# Configuration
# Accept either SERVICE_BUS_CONNECTION_STRING (preferred) or the common
# Azure Functions-style prefix used elsewhere in this repo.
SERVICE_BUS_CONN_STR = os.getenv("SERVICE_BUS_CONNECTION_STRING") or os.getenv(
    "SERVICEBUS_CONNECTION_STRING"
)
PROGRESS_TOPIC = os.getenv("PROGRESS_TOPIC", "dev-progress")
SUBSCRIPTION_NAME = os.getenv("RELAY_SUBSCRIPTION_NAME", "relay-sub")
RELAY_PORT = int(os.getenv("RELAY_PORT", "8092"))
BROADCAST_MODE = os.getenv("BROADCAST_MODE", "session")

if not SERVICE_BUS_CONN_STR:
    print("[ERROR] SERVICE_BUS_CONNECTION_STRING environment variable is required")
    exit(1)

app = FastAPI(title="Dev Progress Relay (Service Bus)")


# Connection manager
class ConnectionManager:
    """Manages WebSocket connections and routes messages by channel."""

    def __init__(self):
        self.connections: Dict[str, Set[WebSocket]] = {}
        self.tab_connections: Dict[str, Dict[str, WebSocket]] = {}  # session -> tab_id -> ws

    async def connect(self, websocket: WebSocket, session_id: str, tab_id: str = None):
        """Connect a WebSocket client."""
        await websocket.accept()

        if BROADCAST_MODE == "tab" and tab_id:
            if session_id not in self.tab_connections:
                self.tab_connections[session_id] = {}
            self.tab_connections[session_id][tab_id] = websocket
            print(f"[CONNECT] Session: {session_id}, Tab: {tab_id}")
        else:
            if session_id not in self.connections:
                self.connections[session_id] = set()
            self.connections[session_id].add(websocket)
            print(f"[CONNECT] Session: {session_id}, Total: {len(self.connections[session_id])}")

    def disconnect(self, websocket: WebSocket, session_id: str, tab_id: str = None):
        """Disconnect a WebSocket client."""
        if BROADCAST_MODE == "tab" and tab_id:
            if session_id in self.tab_connections:
                self.tab_connections[session_id].pop(tab_id, None)
                if not self.tab_connections[session_id]:
                    del self.tab_connections[session_id]
        else:
            if session_id in self.connections:
                self.connections[session_id].discard(websocket)
                if not self.connections[session_id]:
                    del self.connections[session_id]
        print(f"[DISCONNECT] Session: {session_id}")

    async def send_to_session(self, session_id: str, message: dict):
        """Send message to all connections for a session."""
        envelope = {
            "type": "progress",
            "channel": session_id,
            "event": message,
            "timestamp": datetime.utcnow().isoformat(),
        }

        # Session mode: broadcast to all connections
        if session_id in self.connections:
            dead = []
            for ws in self.connections[session_id]:
                try:
                    await ws.send_json(envelope)
                except Exception as e:
                    print(f"[SEND_ERROR] {e}")
                    dead.append(ws)
            for ws in dead:
                self.disconnect(ws, session_id)

        # Tab mode: send to specific tabs
        if session_id in self.tab_connections:
            dead_tabs = []
            for tab_id, ws in self.tab_connections[session_id].items():
                try:
                    await ws.send_json(envelope)
                except Exception as e:
                    print(f"[SEND_ERROR] Tab {tab_id}: {e}")
                    dead_tabs.append(tab_id)
            for tab_id in dead_tabs:
                self.disconnect(None, session_id, tab_id)


manager = ConnectionManager()


@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    channel: str = None,
    session_id: str = None,
    tab_id: str = None,
):
    """
    WebSocket endpoint.

    Query params:
        channel or session_id: conversation_id or job_id
        tab_id: optional tab identifier
    """
    session = channel or session_id
    if not session:
        await websocket.close(code=1008, reason="Missing channel/session_id")
        return

    await manager.connect(websocket, session, tab_id)

    try:
        while True:
            data = await websocket.receive_text()
            if data == "ping":
                await websocket.send_text("pong")
    except WebSocketDisconnect:
        manager.disconnect(websocket, session, tab_id)
    except Exception as e:
        print(f"[WS_ERROR] {e}")
        manager.disconnect(websocket, session, tab_id)


@app.get("/health")
async def health():
    """Health check."""
    return JSONResponse(
        {
            "status": "healthy",
            "service": "dev-servicebus-relay",
            "active_sessions": len(manager.connections) + len(manager.tab_connections),
            "total_connections": sum(len(conns) for conns in manager.connections.values())
            + sum(len(tabs) for tabs in manager.tab_connections.values()),
        }
    )


async def process_service_bus_messages():
    """
    Background task: consume messages from Service Bus and broadcast to WebSocket clients.
    """
    print(f"[SERVICE_BUS] Connecting to topic: {PROGRESS_TOPIC}, subscription: {SUBSCRIPTION_NAME}")

    async with ServiceBusClient.from_connection_string(
        SERVICE_BUS_CONN_STR
    ) as client:
        receiver = client.get_subscription_receiver(
            topic_name=PROGRESS_TOPIC,
            subscription_name=SUBSCRIPTION_NAME,
            max_wait_time=5,
        )

        async with receiver:
            print(f"[SERVICE_BUS] Listening for messages...")
            while True:
                try:
                    messages = await receiver.receive_messages(max_message_count=10, max_wait_time=5)
                    for msg in messages:
                        try:
                            # Parse message body
                            body = json.loads(str(msg))
                            print(f"[RECEIVED] {body.get('operation')} - {body.get('status')}")

                            # Route to appropriate channel
                            job_id = body.get("job_id")
                            conversation_id = body.get("conversation_id")

                            if job_id:
                                await manager.send_to_session(job_id, body)
                            if conversation_id:
                                await manager.send_to_session(conversation_id, body)

                            # Complete the message
                            await receiver.complete_message(msg)

                        except Exception as e:
                            print(f"[PROCESS_ERROR] {e}")
                            await receiver.abandon_message(msg)

                except Exception as e:
                    print(f"[RECEIVER_ERROR] {e}")
                    await asyncio.sleep(5)


@app.on_event("startup")
async def startup_event():
    """Start Service Bus consumer on app startup."""
    asyncio.create_task(process_service_bus_messages())


if __name__ == "__main__":
    import uvicorn

    print(f"[RELAY] Starting on port {RELAY_PORT}...")
    print(f"[CONFIG] Topic: {PROGRESS_TOPIC}, Subscription: {SUBSCRIPTION_NAME}")
    print(f"[CONFIG] Broadcast mode: {BROADCAST_MODE}")

    uvicorn.run(app, host="0.0.0.0", port=RELAY_PORT)



================================================
FILE: src/__init__.py
================================================
[Empty file]


================================================
FILE: src/agent_builder.py
================================================
"""
Deep Agent builder for Dev service.
Follows pattern from FE_Deep_AGENT_v1/backend/agent_builder.py
"""

import logging
import os
from pathlib import Path
from typing import Optional

from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langchain_anthropic import ChatAnthropic
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver

from azure_claude_chat import AzureClaudeChat
from core.database import get_db_manager

# Durable LangGraph checkpointer (graph state per thread_id) backed by MongoDB.
# Optional dependency: if langgraph-checkpoint-mongodb isn't installed we fall
# back to the in-process MemorySaver so the agent still runs.
#
# The async-only ``AsyncMongoDBSaver`` (langgraph.checkpoint.mongodb.aio) was
# deprecated and removed. The standard ``MongoDBSaver`` now supports both sync
# and async environments natively (it wraps pymongo, exposing ``aget_tuple`` etc.
# by running the sync driver in a thread), so we use it for both paths.
try:
    from langgraph.checkpoint.mongodb import MongoDBSaver
except Exception as _ckpt_exc:  # noqa: BLE001
    MongoDBSaver = None
    _checkpoint_import_error = _ckpt_exc

# Azure and S3 backends are vendored under src/ (ported from the ForwardEngineering
# reference) since the installed deepagents build ships filesystem only. Imported
# conditionally so a missing optional SDK (azure-storage-blob / aioboto3) degrades
# gracefully to the filesystem backend instead of crashing at startup.
try:
    from azure_blob_backend import AzureBlobBackend, AzureBlobConfig
except Exception as _azure_exc:  # noqa: BLE001
    AzureBlobBackend = None
    AzureBlobConfig = None
    log_import_azure_error = _azure_exc

try:
    from aws_s3_backend import S3Backend, S3Config
except Exception as _s3_exc:  # noqa: BLE001
    S3Backend = None
    S3Config = None
    log_import_s3_error = _s3_exc

from agent_memory_templates import DESIGN_SKILLS, MEMORY_FILES
from agent_prompts import build_dev_prompt
from agent_tools import create_dev_tools
from core.config import settings
from integrations.github_tools import create_github_tools
from integrations.jira_tools import create_jira_tools

log = logging.getLogger("dev.agent_builder")


def _get_or_create_llm(runtime_config: Optional[dict] = None):
    """Build the LLM instance for the request's provider/model selection.

    Selects the provider from ``runtime_config["llm"]`` (set via the
    Configuration panel's provider/model switch, see
    api/routers/integrations.py) when present, else the static ``MODEL_USED``
    default, and returns a streaming LangChain chat model. Native token
    streaming is required for ``agent.astream(...)`` SSE.
    """
    runtime_config = runtime_config or {}
    selection = runtime_config.get("llm") or {}
    provider = (selection.get("provider") or settings.deep_agent.MODEL_USED or "").strip().lower()

    # Output token budget. 8192 truncates full HLD+LLD+TSD documents; even
    # 16000 truncates a complete 13-section TSD mid-document. Claude Sonnet
    # 4.5 supports up to 64000 output tokens natively, so default to 32000 —
    # enough for a full TSD in one turn without truncation.
    try:
        max_tokens = int(os.getenv("DEV_MAX_TOKENS", "32000"))
    except ValueError:
        max_tokens = 32000

    if provider == "openai":
        api_key = selection.get("api_key") or settings.deep_agent.AZURE_OPENAI_API_KEY
        base_url = settings.deep_agent.AZURE_OPENAI_BASE_URL
        model = selection.get("model") or settings.deep_agent.AZURE_OPENAI_MODEL
        if not api_key:
            raise ValueError("AZURE_OPENAI_API_KEY not configured")
        if not base_url:
            raise ValueError("AZURE_OPENAI_BASE_URL not configured")
        if not model:
            raise ValueError("AZURE_OPENAI_MODEL not configured")
        base_url = base_url.rstrip("/")
        if base_url.endswith("/openai"):
            base_url = f"{base_url}/v1"

        log.info(
            "Creating streaming Azure OpenAI ChatOpenAI instance: %s",
            model,
        )
        return ChatOpenAI(
            model=model,
            base_url=base_url,
            api_key=api_key,
            temperature=0.0,
            max_tokens=max_tokens,
            max_retries=8,
            timeout=300,
            streaming=True,
        )
    elif provider == "claude":
        api_key = selection.get("api_key") or settings.deep_agent.ANTHROPIC_FOUNDRY_API_KEY
        model = selection.get("model") or settings.deep_agent.ANTHROPIC_DEFAULT_SONNET_MODEL
        base_url = settings.deep_agent.ANTHROPIC_FOUNDRY_BASE_URL
        if not api_key:
            raise ValueError("ANTHROPIC_FOUNDRY_API_KEY not configured")

        use_legacy = os.getenv("DEV_USE_LEGACY_LLM", "").lower() in (
            "1",
            "true",
            "yes",
        )

        if use_legacy:
            log.info(
                "Creating legacy AzureClaudeChat instance (no streaming): %s",
                model,
            )
            return AzureClaudeChat(
                api_key=api_key,
                base_url=base_url,
                model=model,
                temperature=0.0,
                max_tokens=max_tokens,
            )
        log.info(
            "Creating streaming ChatAnthropic instance: %s", model
        )
        return ChatAnthropic(
            model=model,
            base_url=base_url,
            api_key=api_key,
            default_headers={"api-key": api_key},
            temperature=0.0,
            max_tokens=max_tokens,
            max_retries=8,
            timeout=300,
            streaming=True,
        )
    else:
        raise ValueError("MODEL_USED must be either 'openai' or 'claude'")


def _initialize_llm(temperature: float = 0.0, runtime_config: Optional[dict] = None):
    """Get the LLM instance for this request."""
    return _get_or_create_llm(runtime_config)


def create_dev_agent(
    workspace_id: str,
    user_id: str,
    conversation_id: str,
    agent_id: int = 1,
    job_id: Optional[str] = None,
    extra_tools: Optional[list] = None,
    workspace_dir: Optional[str] = None,
    auth_token: Optional[str] = None,
    agent_profile: Optional[dict] = None,
    run_location=None,
    runtime_config: Optional[dict] = None,
):
    """Create Dev Deep Agent with workspace context and DESIGN_SKILLS.

    Args:
        workspace_id: Workspace identifier for multi-tenancy
        user_id: User identifier
        conversation_id: Conversation/session identifier
        agent_id: Agent identifier for LLM routing (default: 1)
        job_id: Optional workflow job ID for context curation
        auth_token: Optional bearer token for KB search authentication
        agent_profile: Optional resolved agent record from the Postgres registry
            (agents_details + agents_cms). When present, its name/description are
            woven into the system prompt so the agent reflects its registry
            identity. Absent/None keeps the default behavior.

    Returns:
        Deep Agent instance configured for architecture design
    """

    log.info(
        "Creating Dev Deep Agent",
        extra={
            "workspace_id": workspace_id,
            "user_id": user_id,
            "conversation_id": conversation_id,
            "agent_id": agent_id,
            "job_id": job_id,
        },
    )

    # Long-term memory partition: shared across a dynamic workflow, or isolated
    # per agent for individual runs (see RunLocation / the blob restructure).
    memory_scope = None
    if run_location is not None:
        if (
            run_location.scope == "dynamic_workflow"
            and run_location.workflow_job_id
        ):
            memory_scope = f"workflow:{run_location.workflow_job_id}"
        else:
            memory_scope = f"agent:{run_location.agent_folder}"

    # Create workspace-scoped tools
    tools = create_dev_tools(
        workspace_id=workspace_id,
        user_id=user_id,
        conversation_id=conversation_id,
        job_id=job_id,
        auth_token=auth_token,
        memory_scope=memory_scope,
        run_location=run_location,
    )
    if workspace_dir:
        jira_config = (runtime_config or {}).get("jira") or {}
        github_config = (runtime_config or {}).get("github") or {}
        if jira_config.get("active"):
            tools = list(tools) + create_jira_tools(
                workspace_dir,
                run_location.agent_folder if run_location else "dev",
                config=jira_config,
                workspace_id=workspace_id,
                run_location=run_location,
            )
        if github_config.get("active"):
            tools = list(tools) + create_github_tools(
                workspace_dir,
                run_location.agent_folder if run_location else "dev",
                config=github_config,
                workspace_id=workspace_id,
                run_location=run_location,
            )

    # Append any per-session workspace tools (save_output / read_output / list_outputs)
    if extra_tools:
        tools = list(tools) + list(extra_tools)

    # Build system prompt with workspace context
    system_prompt = build_dev_prompt(
        workspace_id=workspace_id,
        conversation_id=conversation_id,
    )

    # Weave resolved registry identity into the prompt when available.
    if agent_profile:
        name = agent_profile.get("agent_name")
        desc = agent_profile.get("agent_desc")
        identity_lines = []
        if name:
            identity_lines.append(
                f"You are operating as the registered agent: {name}."
            )
        if desc:
            identity_lines.append(f"Agent description: {desc}")
        if identity_lines:
            system_prompt = system_prompt + "\n\n" + "\n".join(identity_lines)

    # Select backend based on configuration. When a per-session workspace dir is
    # provided, scope the filesystem backend to it so generated files are visible
    # under WORKSPACE_ROOT/<session_id>.
    if run_location is not None:
        backend = _build_backend(
            workspace_dir=workspace_dir,
            blob_prefix=run_location.blob_prefix,
            db_session_id=run_location.db_session_id,
            doc_meta={
                "workspace_id": run_location.workspace_id,
                "workflow_job_id": run_location.workflow_job_id,
                "scope": run_location.scope,
            },
        )
    else:
        backend = _build_backend(
            workspace_dir=workspace_dir, workspace_id=workspace_id
        )

    # Initialize LLM (direct Anthropic integration via Azure Foundry)
    llm = _initialize_llm(temperature=0.0, runtime_config=runtime_config)

    # Durable checkpointer so graph state survives restarts / spans instances.
    checkpointer = _build_checkpointer()

    # Create Deep Agent
    agent = create_deep_agent(
        model=llm,
        system_prompt=system_prompt,
        backend=backend,
        skills=DESIGN_SKILLS,  # ["/skills/DEV_SCAFFOLDING_SKILL.md"]
        memory=MEMORY_FILES,
        tools=tools,
        checkpointer=checkpointer,
    )

    log.info(
        "Dev Deep Agent created successfully",
        extra={
            "workspace_id": workspace_id,
            "skill_count": len(DESIGN_SKILLS),
            "tool_count": len(tools),
        },
    )

    return agent


# Cached pymongo client dedicated to the checkpointer. ``MongoDBSaver`` wraps the
# synchronous pymongo driver (running it in a thread for its async methods), so it
# needs a real ``pymongo.MongoClient`` rather than the Motor async client used
# elsewhere. Cached at module level so we open at most one extra connection pool.
_checkpoint_mongo_client = None


def _get_checkpoint_mongo_client():
    """Lazily create (and cache) the pymongo client for the checkpointer."""
    global _checkpoint_mongo_client
    if _checkpoint_mongo_client is not None:
        return _checkpoint_mongo_client

    uri = settings.database.MONGODB_DATABASE_URI
    if not uri:
        return None
    from pymongo import MongoClient

    _checkpoint_mongo_client = MongoClient(uri)
    return _checkpoint_mongo_client


def _build_checkpointer():
    """Return a durable MongoDB checkpointer, or MemorySaver as a fallback.

    Stores graph state in the configured ``dev_checkpoints`` /
    ``dev_checkpoint_writes`` collections via the standard
    ``MongoDBSaver`` (async-capable). Falls back to the in-process
    ``MemorySaver`` when the optional ``langgraph-checkpoint-mongodb`` dependency
    is missing or no Mongo URI is configured.
    """
    if MongoDBSaver is None:
        log.warning(
            "langgraph-checkpoint-mongodb unavailable (%s); using in-memory "
            "MemorySaver (checkpoints will not survive restart)",
            globals().get("_checkpoint_import_error", "import failed"),
        )
        return MemorySaver()

    client = _get_checkpoint_mongo_client()
    if client is None:
        log.warning(
            "MongoDB URI not configured; using in-memory MemorySaver for "
            "checkpoints this run"
        )
        return MemorySaver()

    try:
        return MongoDBSaver(
            client,
            db_name=settings.database.DEV_DB_NAME,
            checkpoint_collection_name=settings.database.DEV_CHECKPOINTS_COLLECTION,
            writes_collection_name=settings.database.DEV_CHECKPOINT_WRITES_COLLECTION,
        )
    except Exception as exc:  # noqa: BLE001
        log.warning(
            "Failed to build MongoDB checkpointer (%s); falling back to MemorySaver",
            exc,
        )
        return MemorySaver()


def _build_backend(
    workspace_dir: Optional[str] = None,
    workspace_id: Optional[str] = None,
    blob_prefix: Optional[str] = None,
    db_session_id: Optional[str] = None,
    doc_meta: Optional[dict] = None,
):
    """Create storage backend based on CLOUD_STORAGE_PROVIDER setting.

    If ``workspace_dir`` is given (per-run), the filesystem backend is rooted
    there so the agent's file writes land under that run's folder.

    ``blob_prefix`` explicitly sets the blob/object prefix (the run-scoped
    location — e.g. ``<workspace_id>/dynamic_workflow/<job>``). When omitted it
    degrades to the legacy ``<workspace_id>/<session_id>`` derived from the
    workspace dir basename. ``db_session_id`` is the key used for document
    registration (Azure config ``session_id``); it defaults to the basename.
    ``doc_meta`` carries extra columns (workspace_id/workflow_job_id/scope) for
    the ``fe_agent_document`` registry.
    """
    provider = settings.deep_agent.CLOUD_STORAGE_PROVIDER.lower()

    log.info(f"Initializing backend: {provider}")

    # The session id is the workspace dir basename (mirrors the ForwardEngineering
    # reference's module_id). The blob/object prefix groups it under the workspace
    # when a workspace_id is available; otherwise it degrades to the session id.
    session_id = Path(workspace_dir).name if workspace_dir else ""
    session_prefix = blob_prefix or (
        f"{workspace_id}/{session_id}"
        if (workspace_id and session_id)
        else session_id
    )
    doc_session_id = db_session_id or session_id
    doc_meta = doc_meta or {}

    if provider == "azure":
        if AzureBlobBackend is None or AzureBlobConfig is None:
            log.warning(
                "Azure backend unavailable (%s); falling back to filesystem",
                globals().get("log_import_azure_error", "import failed"),
            )
            provider = "filesystem"
        else:
            conn_str = settings.azure.BLOB_STORAGE_CONNECTION_STRING
            account_name = settings.azure.BLOB_ACCOUNT_NAME
            account_key = settings.azure.BLOB_ACCOUNT_KEY
            # Prefer AZURE_BLOB_STORAGE_CONTAINER_NAME (settings.azure.BLOB_CONTAINER_NAME);
            # WORKSPACE_BLOB_CONTAINER is only an explicit override when set.
            container = (
                settings.deep_agent.WORKSPACE_BLOB_CONTAINER
                or settings.azure.BLOB_CONTAINER_NAME
            )

            kwargs: dict = {
                "container_name": container,
                "prefix": session_prefix,
                "session_id": doc_session_id,
                "workspace_id": doc_meta.get("workspace_id"),
                "workflow_job_id": doc_meta.get("workflow_job_id"),
                "scope": doc_meta.get("scope"),
            }
            if conn_str:
                kwargs["connection_string"] = conn_str
            elif account_name and account_key:
                kwargs["account_url"] = (
                    f"https://{account_name}.blob.core.windows.net"
                )
                kwargs["account_key"] = account_key
            else:
                raise ValueError(
                    "azure provider requires either BLOB_STORAGE_CONNECTION_STRING "
                    "or BLOB_ACCOUNT_NAME + BLOB_ACCOUNT_KEY"
                )

            cfg = AzureBlobConfig(**kwargs)
            log.info(
                f"Using AzureBlobBackend: container={container} prefix={session_prefix}"
            )
            return AzureBlobBackend(cfg)

    if provider in ("s3", "aws"):
        if S3Backend is None or S3Config is None:
            log.warning(
                "S3 backend unavailable (%s); falling back to filesystem. "
                "Install 'aioboto3' to enable it.",
                globals().get("log_import_s3_error", "import failed"),
            )
            provider = "filesystem"
        else:
            bucket = settings.deep_agent.AWS_S3_BUCKET_NAME

            if not bucket:
                raise ValueError(
                    "AWS_S3_BUCKET_NAME is required for s3 provider"
                )

            cfg = S3Config(
                bucket=bucket,
                prefix=session_prefix,
                access_key_id=settings.deep_agent.AWS_ACCESS_KEY_ID,
                secret_access_key=settings.deep_agent.AWS_SECRET_ACCESS_KEY,
                region=settings.deep_agent.AWS_DEFAULT_REGION,
            )

            log.info(
                f"Using S3Backend: bucket={bucket} prefix={session_prefix}"
            )
            return S3Backend(cfg)

    # DISABLED: Dev artifacts must never fall back to local filesystem
    # persistence. Keep the previous implementation commented for reference.
    # if workspace_dir:
    #     workspace_root = Path(workspace_dir)
    # else:
    #     workspace_root = Path(settings.deep_agent.WORKSPACE_ROOT)
    # workspace_root.mkdir(parents=True, exist_ok=True)
    # log.info(f"Using FilesystemBackend: root={workspace_root}")
    # return FilesystemBackend(
    #     root_dir=str(workspace_root),
    #     virtual_mode=True,
    # )
    raise RuntimeError(
        f"Cloud storage backend '{settings.deep_agent.CLOUD_STORAGE_PROVIDER}' "
        "is unavailable; local filesystem persistence is disabled."
    )



================================================
FILE: src/agent_memory_templates.py
================================================
"""
Memory templates and skill file mappings for Deep Agent.
Defines which skill files and memory files to load.
"""

# Design skills - loaded at agent creation
# These are virtual paths mapped by the backend
DESIGN_SKILLS = [
    "/skills/DEV_SCAFFOLDING_SKILL.md",
]

# Memory files - optional conversation history
# Can be used to persist agent memory across sessions
MEMORY_FILES = [
    # Empty by design. Durable memory is provided at runtime instead of via
    # static files:
    #   - conversation transcript: load_conversation_history() tool
    #   - long-term notes: remember_fact() / recall_memory() tools backed by the
    #     dev_agent_memory collection (services/agent_memory.py)
    #   - graph state: MongoDB checkpointer (services build in agent_builder.py)
]

# Skill file content mapping (for FilesystemBackend)
# In production with Azure Blob, skills are loaded from blob storage
SKILL_FILE_MAPPINGS = {
    "/skills/DEV_SCAFFOLDING_SKILL.md": "src/skills/DEV_SCAFFOLDING_SKILL.md",
}



================================================
FILE: src/agent_prompts.py
================================================
"""
System prompts for Dev-Dynamic Deep Agent.
Cloned from the Dev agent; the role/workflow is code scaffolding from a TSD.
"""
from datetime import date


DEV_PROMPT = """You are a Senior Staff Software Engineer with 20+ years of experience building production systems, and an expert at turning a Technical Specification Document (TSD) into a clean, well-planned code scaffold.

ROLE & OBJECTIVE:
You READ a TSD (Technical Specification Document) and PRODUCE a well-structured **code
scaffolding** for the described system: first a build **PLAN**, then a real source tree of
folders and stub files (with docstrings, TODOs, signatures, config, entrypoints, a test
skeleton, and a README). You do NOT write full business logic — you scaffold the project so a
development team can start implementing immediately, with the structure, contracts, and build
order already decided.

All artifacts are in ENGLISH. Markdown docs are MARKDOWN; code stubs use the language/framework
implied by the TSD (default Python/FastAPI if the TSD does not specify). You apply strong
engineering judgment: sensible layering, clear module boundaries, dependency direction, and
naming conventions.

**CRITICAL**: Only scaffold when the user provides a TSD or requirements. For greetings ("hi",
"hello", "thanks"), respond briefly without generating anything.

WORKSPACE CONTEXT:
- Workspace ID: {workspace_id}
- Conversation ID: {conversation_id}
- Date: {current_date}

YOUR TOOLS:
You have access to the following tools for reading context and managing outputs:

1. curate_workflow_context(agent_feed=None) → Fetches upstream artifacts (BRD/TSD produced earlier)
2. load_conversation_history(limit) → Retrieves recent messages
3. save_output(filename, content) → Creates/overwrites a file with its FIRST chunk
4. append_output(filename, content) → Appends each subsequent chunk (build large files incrementally)
5. get_previous_design() → Retrieves the last generated scaffold/plan
6. render_mermaid_diagram(code) → Converts Mermaid to an SVG URL (for diagrams in PLAN.md)
7. get_active_skill() → Loads current skill configuration
8. remember_fact(key, content) → Persist a durable, long-term note that should survive across turns and future sessions (e.g. tech stack, target platforms, key constraints, naming conventions, deadlines, stakeholder decisions). Re-using a key overwrites the prior value.
9. recall_memory() → Load all durable long-term notes saved for this conversation.
10. kb_search(query, user_prompt) → Retrieve relevant knowledge-base context (LightRAG/KBCurator).

OPTIONAL INTEGRATION TOOLS (available only when enabled for this run):
- GitHub: `list_pushable_documents`, `push_document_to_github`,
  `pull_document_from_github`, `push_code_to_github`, `pull_code_from_github`,
  `create_github_branch`, `create_branch_and_push`, and `create_github_pr`.
- Jira: `test_jira_connection`, `get_jira_projects`, `get_jira_issues`,
  `create_jira_story`, `push_user_stories_to_jira`, and
  `pull_jira_stories_to_workspace`.
Use these only for explicit integration requests after the scaffold artifacts
are saved. They read/write the active run backend, not chat history.

INTAKE DOCUMENTS (workspace tools):
- list_inputs() → lists intake documents the user uploaded (the TSD, requirements, schemas)
- read_input(filename) → reads the FULL content of an uploaded intake document (read each ONCE)
ALWAYS call list_inputs() early; find and read_input(...) the TSD (e.g. "TSD.md") and any schema
or requirements files, and ground the entire scaffold in their contents. Reads may run in parallel;
read each file exactly once and keep it in working context.

WORKFLOW (execute in order):
1. Check if input is a greeting → respond with a brief acknowledgment, do NOT generate anything.
2. Call list_inputs() and curate_workflow_context() to locate the TSD and project context.
3. read_input(...) the TSD (and schemas/requirements). If no TSD is present, ask the user to
   provide one instead of guessing.
4. Call load_conversation_history() to understand any previous scaffold in this conversation.
   - Update/resume request → call list_outputs() first, inspect the existing scaffold,
     call get_previous_design() when needed, and continue only missing work. Never
     regenerate or overwrite completed files just because the prior run stopped.
5. **PLANNING (MANDATORY — do this BEFORE any code stub):**
   Produce **PLAN.md** capturing the engineering plan derived from the TSD:
   - Chosen stack & rationale (language, framework, datastore, messaging) — honor what the TSD
     specifies; only default when it is silent.
   - Module/service breakdown mapped back to TSD components.
   - The full target folder structure (as a tree) with a one-line purpose per folder.
   - Build order / milestones and key risks or open questions.
   - A Mermaid component or dependency diagram (render it via render_mermaid_diagram BEFORE saving
     the chunk that references it, so the saved markdown already includes the ![Diagram](url) link).
6. **SCAFFOLDING (MANDATORY — this is how work is delivered):**
   You MUST persist every file via save_output/append_output. Text in your chat reply is NOT saved
   and is NOT a downloadable artifact. Generate the scaffold as real files:
   - Folders are implied by file paths — save files at their intended relative paths
     (e.g. save_output("src/api/routes/users.py", <stub>)).
   - Each stub file contains: module docstring, imports, class/function signatures with type hints,
     concise `# TODO:` markers for the logic to be implemented, and any obvious wiring
     (router registration, DI, config access) — but NOT full business logic.
   - Include: entrypoint(s) (e.g. main.py / app factory), config module, data models / schemas,
     service and repository layers, API routes/controllers, a tests/ skeleton (one test stub per
     module) with a conftest, requirements/pyproject (or package.json / pom.xml as appropriate),
     Dockerfile, .env.example, and a top-level README.md with run instructions.
   - Save files in a FEW LARGE CHUNKS per file when a file is long: save_output(...) the first
     chunk, then ~2-5 append_output(...) calls; do NOT make one tiny call per function. Never issue
     parallel appends to the SAME file (they race and lose content); appends to one file must be
     sequential. Different files may be created independently.
   - Finish and save one file before moving to the next.

7. Return a SHORT summary (a few lines): the chosen stack, the top-level tree, and the count of
   files scaffolded — do NOT paste full file bodies again in the final message into the PLAN.md by creating a section at last.

DOCUMENT/CODE PERSISTENCE (workspace tools):
- save_output(filename, content) → creates/overwrites a file with its FIRST chunk (downloadable by the user)
- append_output(filename, content) → appends the next chunk to that file (sequential, never parallel)
- read_output(filename) → reads back a previously saved file
- list_outputs() → lists all saved files

Conversation history belongs only in `memory/chat_history.json`; never write it to
PLAN.md, README.md, source files, or any other generated artifact.

IMPORTANT: For greetings like "hi", "hello", "thanks" → DO NOT generate any files. Only scaffold
when the user provides a TSD / requirements or explicitly requests a scaffold.

ENGINEERING DEFAULTS (override when the TSD specifies otherwise):
- Language/Framework: Python 3.11 + FastAPI (async) for services; adapt to Java/Spring, Node/Express,
  or others when the TSD calls for them.
- Layout: clean layered architecture (api → service → repository → model), config isolated,
  dependency-injection friendly, 12-factor config via environment.
- Data: PostgreSQL / Cosmos DB / Redis per the TSD; define models and migration placeholders.
- Testing: pytest (or the ecosystem-standard runner) with a mirrored tests/ tree and fixtures.
- Packaging/Ops: Dockerfile, .env.example, requirements.txt/pyproject.toml (or equivalent), CI stub.
- Quality: consistent naming, small focused modules, explicit interfaces, no dead files.

STUB QUALITY RULES:
- Every stub is syntactically valid and importable (no half-written lines).
- Signatures include type hints and docstrings; bodies use `# TODO:` or `raise NotImplementedError`.
- No placeholder junk like "foo/bar" — name things from the TSD's real domain entities.
- The folder tree in PLAN.md MUST match the files you actually save.
- Mermaid diagrams in PLAN.md must be syntactically valid (graph TD, flowchart, classDiagram).

OUTPUT RULES:
- PLAN.md and README.md are Markdown; code files are valid source in the target language.
- Do NOT wrap entire files in code fences inside save_output — save the raw file content.
- Ground every decision in the TSD; include brief rationale for stack and structure choices in PLAN.md.
"""


def build_dev_prompt(
    workspace_id: str,
    conversation_id: str,
) -> str:
    """Build the Dev-Dynamic agent system prompt with context.

    Args:
        workspace_id: Workspace identifier
        conversation_id: Conversation identifier

    Returns:
        Formatted system prompt with context variables
    """
    current_date = date.today().isoformat()

    return DEV_PROMPT.format(
        workspace_id=workspace_id,
        conversation_id=conversation_id,
        current_date=current_date,
    )



================================================
FILE: src/agent_tools.py
================================================
"""
LangChain tools for Dev Deep Agent.
Wraps existing services (workflow_context, design_snapshot, mermaid, etc.)
"""
from typing import Optional, List
from langchain_core.tools import tool

from services.workflow_context import WorkflowContextManager
from services.session_managers import SessionHistoryManager
from services.design_snapshot import get_snapshot
from services.kb_client import lightrag_query
from services.agent_memory import remember as _remember_memory, recall as _recall_memory
from services.mermaid import render_mermaid_to_url
from services.skills import get_active_skill as _get_active_skill
from core.config import settings

# Design snapshots are intentionally not exposed by Dev-Dynamic; PLAN.md in
# the run workspace is the canonical planning artifact.


def create_dev_tools(
    workspace_id: str,
    user_id: str,
    conversation_id: str,
    job_id: Optional[str] = None,
    auth_token: Optional[str] = None,
    memory_scope: Optional[str] = None,
    run_location=None,
):
    """Create workspace-scoped tools for Dev agent.

    Tools are closures capturing workspace context to avoid global state.

    Args:
        workspace_id: Workspace identifier
        user_id: User identifier
        conversation_id: Conversation/session identifier
        job_id: Optional workflow job ID for context curation
        auth_token: Optional bearer token for KB search authentication
        memory_scope: Optional long-term-memory partition key. When set it
            replaces ``conversation_id`` for remember/recall so memory can be
            *shared* across a dynamic workflow (``workflow:<job>``) or isolated
            *per agent* (``agent:<folder>``). Defaults to ``conversation_id``.

    Returns:
        List of LangChain tools
    """
    memory_key = memory_scope or conversation_id

    # Initialize service managers (async-safe singletons)
    workflow_ctx = WorkflowContextManager()
    session_history = SessionHistoryManager()

    @tool
    async def curate_workflow_context(agent_feed: Optional[List[str]] = None) -> str:
        """Fetch project context (BRD, requirements) from workflow database.

        Args:
            agent_feed: List of artifact types to fetch (default: ["BRD"])

        Returns:
            Curated project context as formatted string
        """
        agent_feed = agent_feed or ["BRD"]
        context = await workflow_ctx.curate_context(
            workspace_id=workspace_id,
            job_id=job_id,
            agent_feed=agent_feed,
        )
        return context or "No project context available (standalone mode)"

    @tool
    async def load_conversation_history(limit: int = 10) -> str:
        """Load recent conversation history for this session.

        Args:
            limit: Maximum number of messages to retrieve

        Returns:
            Formatted conversation history
        """
        history = await session_history.load_history(
            workspace_id=workspace_id,
            user_id=user_id,
            session_id=conversation_id,
            limit=limit,
        )
        if not history:
            return "No previous conversation history"

        formatted = []
        for msg in history:
            role = msg.get("role", "unknown")
            content = msg.get("content", "")
            # Truncate long messages
            preview = content[:200] + "..." if len(content) > 200 else content
            formatted.append(f"[{role}]: {preview}")
        return "\n".join(formatted)

    @tool
    async def kb_search(query: str, user_prompt: str = "") -> str:
        """Search the knowledge base (LightRAG/KBCurator) for relevant context.

        Args:
            query: The retrieval question / requirement query.
            user_prompt: Optional extra instruction passed to the KB.

        Returns:
            Retrieved context, or a fallback message when nothing is found.
        """
        namespace = {"workspace_id": int(workspace_id), "user_id": user_id}
        # Construct Authorization header if auth_token is provided
        auth_header = f"Bearer {auth_token}" if auth_token else ""
        return await lightrag_query(
            prompt=query,
            namespace=namespace,
            user_prompt=user_prompt,
            auth_header=auth_header,
        )

    # @tool
    # async def save_design_snapshot(design_markdown: str = "") -> str:
    #     """Persist the generated TSD to the database (final step).

    #     You do NOT need to pass the whole document here — if you already wrote it
    #     to your output folder (save_output/append_output on TSD.md), call this with
    #     no argument and the already-saved TSD is persisted from that file. This
    #     avoids re-streaming the full document (which can be truncated).

    #     Args:
    #         design_markdown: Optional full TSD markdown. Leave empty to snapshot the
    #             TSD.md already saved in your output folder.

    #     Returns:
    #         Confirmation message
    #     """
    #     # Prefer the explicitly passed markdown; otherwise fall back to the TSD
    #     # already persisted in this agent's output folder, so the final snapshot
    #     # never depends on the model re-emitting the whole (truncation-prone) doc.
    #     if not (design_markdown or "").strip() and run_location is not None:
    #         try:
    #             from services import workspace_tools

    #             saved = workspace_tools.read_output_file(
    #                 conversation_id, "TSD.md", run_location=run_location
    #             )
    #             if saved and saved.strip():
    #                 design_markdown = saved
    #         except Exception as exc:  # noqa: BLE001 — best-effort fallback
    #             logger.warning(f"save_design_snapshot file fallback failed: {exc}")

    #     if not (design_markdown or "").strip():
    #         return (
    #             "No design content to snapshot. Save the TSD first with "
    #             "save_output('TSD.md', <first section>) and append_output('TSD.md', "
    #             "<next section>), then call save_design_snapshot() again."
    #         )

    #     design_struct = {
    #         "format": "markdown",
    #         "markdown": design_markdown,
    #     }

    #     await upsert_snapshot(
    #         workspace_id=workspace_id,
    #         user_id=user_id,
    #         conversation_id=conversation_id,
    #         last_hld_struct=design_struct,
    #         last_hld_plaintext=design_markdown,
    #     )

    #     return f"Design snapshot saved successfully ({len(design_markdown)} chars)"

    @tool
    async def get_previous_design() -> str:
        """Retrieve the last generated design for this conversation.

        Returns:
            Previous design markdown or empty string
        """
        snap = await get_snapshot(
            workspace_id=workspace_id,
            user_id=user_id,
            conversation_id=conversation_id,
        )
        if not snap:
            return "No previous design found"

        plaintext = snap.get("last_hld_plaintext", "")
        return plaintext or "Previous design unavailable"

    @tool
    async def render_mermaid_diagram(mermaid_code: str) -> str:
        """Convert Mermaid diagram code to publicly accessible SVG URL.

        Args:
            mermaid_code: Mermaid diagram code (graph TD, sequenceDiagram, etc.)

        Returns:
            Public URL to rendered SVG image or code block if rendering fails
        """
        public_base_url = settings.render.PUBLIC_BASE_URL
        if not public_base_url:
            return f"```mermaid\n{mermaid_code}\n```"  # Fallback to code block

        try:
            url = await render_mermaid_to_url(
                raw_code=mermaid_code,
                public_base_url=public_base_url,
                public_dir=settings.render.PUBLIC_ASSET_DIR,
                chrome_path=settings.render.CHROME_EXECUTABLE_PATH,
            )
            return f"![Architecture Diagram]({url})"
        except Exception as e:
            return f"Error rendering diagram: {e}\n```mermaid\n{mermaid_code}\n```"

    @tool
    async def remember_fact(key: str, content: str) -> str:
        """Persist a durable note to long-term memory for this conversation.

        Use for stable facts worth recalling in future turns/sessions (e.g. the
        chosen tech stack, naming conventions, key constraints). Re-using a
        ``key`` overwrites the prior value.

        Args:
            key: Short identifier for the memory (e.g. "tech_stack").
            content: The fact to remember.

        Returns:
            Confirmation message.
        """
        await _remember_memory(
            workspace_id=workspace_id,
            user_id=user_id,
            conversation_id=memory_key,
            key=key,
            content=content,
        )
        return f"Remembered '{key}'."

    @tool
    async def recall_memory() -> str:
        """Load durable long-term memory notes saved for this conversation.

        Returns:
            Formatted "key: content" lines, or a note if none exist.
        """
        entries = await _recall_memory(
            workspace_id=workspace_id,
            user_id=user_id,
            conversation_id=memory_key,
        )
        if not entries:
            return "No long-term memory stored yet"
        return "\n".join(f"{e.get('key')}: {e.get('content', '')}" for e in entries)

    @tool
    async def get_active_skill() -> str:
        """Load the current active skill configuration for this agent.

        Returns:
            Skill configuration as formatted string
        """
        try:
            skill_doc = await _get_active_skill(
                workspace_id=workspace_id,
                agent_id=1,  # Dev agent ID
                skill_id="dev_scaffolding_generation",
            )

            role = skill_doc.get("role", "")
            objective = skill_doc.get("objective", "")
            instructions = skill_doc.get("instructions", [])

            formatted = []
            if role:
                formatted.append(f"Role: {role}")
            if objective:
                formatted.append(f"Objective: {objective}")
            if instructions:
                formatted.append("Instructions:")
                formatted.extend([f"- {inst}" for inst in instructions[:10]])  # Limit to 10

            return "\n".join(formatted) if formatted else "No active skill found"
        except Exception as e:
            return f"Error loading skill: {e}"

    return [
        curate_workflow_context,
        load_conversation_history,
        kb_search,
        # save_design_snapshot,
        get_previous_design,
        remember_fact,
        recall_memory,
        render_mermaid_diagram,
        get_active_skill,
    ]



================================================
FILE: src/azure_claude_chat.py
================================================
"""
Complete LangChain-compatible chat model for Azure Anthropic endpoints.
Uses direct HTTP calls that match the working curl command exactly.
"""

import asyncio
import json
import time
from typing import Any, Dict, List, Optional, Sequence, Union

import httpx

# Granular timeout: allow a long read for slow agent LLM calls, but keep the
# connect timeout short so genuinely dead endpoints fail fast. `read=None`
# disables the read timeout entirely (bounded only by the model's own runtime).
REQUEST_TIMEOUT = httpx.Timeout(connect=15.0, read=600.0, write=60.0, pool=60.0)

# Retry transient read/connect timeouts a couple of times with backoff.
MAX_RETRIES = 3
RETRY_BACKOFF_SECONDS = 2.0
from langchain_core.language_models.chat_models import BaseChatModel
from langchain_core.messages import (
    AIMessage,
    BaseMessage,
    HumanMessage,
    SystemMessage,
    ToolMessage,
)
from langchain_core.outputs import ChatGeneration, ChatResult
from langchain_core.runnables import Runnable, RunnableConfig
from langchain_core.tools import BaseTool
from pydantic import Field


class AzureClaudeChat(BaseChatModel):
    """LangChain chat model for Azure-hosted Anthropic Claude.

    Uses direct HTTP requests that match the proven working curl command.
    """

    api_key: str
    base_url: str
    model: str = "claude-sonnet-4-5-forgex-rnd"
    # None = omit `temperature` from the API payload. Newer Claude models
    # (Opus 4.5+/5) reject the request with a 400 when temperature is sent, so it
    # is optional and only forwarded when explicitly set.
    temperature: Optional[float] = None
    max_tokens: int = 8192
    tools: Optional[List[Any]] = None

    class Config:
        arbitrary_types_allowed = True

    @property
    def _llm_type(self) -> str:
        return "azure-claude"

    def _get_endpoint_url(self) -> str:
        """Construct proper endpoint URL for Anthropic messages API."""
        url = self.base_url.rstrip("/")
        if not url.endswith("/v1/messages"):
            if url.endswith("/v1"):
                url = f"{url}/messages"
            else:
                url = f"{url}/v1/messages"
        return url

    def _convert_messages_to_anthropic(
        self, messages: List[BaseMessage]
    ) -> tuple[Optional[Union[str, List[Dict[str, Any]]]], List[Dict[str, Any]]]:
        """Convert LangChain messages to Anthropic format."""
        system_parts = []
        converted = []

        for msg in messages:
            if isinstance(msg, SystemMessage):
                if isinstance(msg.content, str) and msg.content:
                    system_parts.append(msg.content)
                elif isinstance(msg.content, list):
                    system_parts.extend(msg.content)
                continue

            if isinstance(msg, HumanMessage):
                content = msg.content if msg.content is not None else ""
                # Anthropic rejects empty user content; provide a placeholder.
                if isinstance(content, str) and content == "":
                    content = "(empty)"
                converted.append({"role": "user", "content": content})
                continue

            if isinstance(msg, AIMessage):
                content_blocks = []
                if isinstance(msg.content, str) and msg.content:
                    content_blocks.append({"type": "text", "text": msg.content})
                elif isinstance(msg.content, list):
                    content_blocks.extend(msg.content)

                tool_calls = getattr(msg, "tool_calls", None) or []
                for tc in tool_calls:
                    content_blocks.append({
                        "type": "tool_use",
                        "id": tc["id"],
                        "name": tc["name"],
                        "input": tc.get("args", {}),
                    })

                # Drop empty text blocks — Anthropic rejects text blocks
                # whose "text" is empty/whitespace-only.
                content_blocks = [
                    b for b in content_blocks
                    if not (
                        isinstance(b, dict)
                        and b.get("type") == "text"
                        and not (b.get("text") or "").strip()
                    )
                ]

                # An assistant turn with no content at all is invalid; skip it.
                if not content_blocks:
                    continue

                converted.append({"role": "assistant", "content": content_blocks})
                continue

            if isinstance(msg, ToolMessage):
                tool_result_block = {
                    "type": "tool_result",
                    "tool_use_id": msg.tool_call_id,
                    "content": msg.content if isinstance(msg.content, (str, list)) else str(msg.content),
                }
                if (
                    converted
                    and converted[-1]["role"] == "user"
                    and isinstance(converted[-1]["content"], list)
                    and converted[-1]["content"]
                    and isinstance(converted[-1]["content"][0], dict)
                    and converted[-1]["content"][0].get("type") == "tool_result"
                ):
                    converted[-1]["content"].append(tool_result_block)
                else:
                    converted.append({"role": "user", "content": [tool_result_block]})
                continue

        system_param = None
        if system_parts:
            if all(isinstance(s, str) for s in system_parts):
                system_param = "\n\n".join([s for s in system_parts if s])
            else:
                system_param = system_parts

        return system_param, converted

    def _convert_tools_to_anthropic(
        self, tools: Sequence
    ) -> List[Dict[str, Any]]:
        """Convert LangChain tools to Anthropic tool format."""
        anthropic_tools = []

        for tool in tools:
            if isinstance(tool, dict):
                name = tool.get("name")
                desc = tool.get("description", "") or ""
                schema = (
                    tool.get("input_schema")
                    or tool.get("parameters")
                    or {"type": "object", "properties": {}}
                )
                if isinstance(schema, dict) and "type" not in schema:
                    schema["type"] = "object"
                anthropic_tools.append({
                    "name": name,
                    "description": desc,
                    "input_schema": schema,
                })
            elif hasattr(tool, "name"):
                name = getattr(tool, "name")
                desc = getattr(tool, "description", "") or ""
                if hasattr(tool, "args_schema") and tool.args_schema:
                    if hasattr(tool.args_schema, "model_json_schema"):
                        schema = tool.args_schema.model_json_schema()
                    elif hasattr(tool.args_schema, "schema"):
                        schema = tool.args_schema.schema()
                    else:
                        schema = {"type": "object", "properties": {}}
                elif hasattr(tool, "args"):
                    schema = {"type": "object", "properties": getattr(tool, "args", {})}
                else:
                    schema = {"type": "object", "properties": {}}

                if isinstance(schema, dict) and "type" not in schema:
                    schema["type"] = "object"

                anthropic_tools.append({
                    "name": name,
                    "description": desc,
                    "input_schema": schema,
                })

        return anthropic_tools

    def _make_request(
        self,
        messages: List[BaseMessage],
        stop: Optional[List[str]] = None,
        **kwargs: Any,
    ) -> Dict[str, Any]:
        """Make HTTP request to Azure Anthropic endpoint."""
        # Convert messages
        system_msg, anthropic_messages = self._convert_messages_to_anthropic(
            messages
        )

        # Prepare payload - matching working curl command
        payload = {
            "model": self.model,
            "max_tokens": self.max_tokens,
            "messages": anthropic_messages,
        }
        if self.temperature is not None:
            payload["temperature"] = self.temperature

        if system_msg:
            payload["system"] = system_msg

        if stop:
            payload["stop_sequences"] = stop

        # Add tools if bound
        if self.tools:
            payload["tools"] = self._convert_tools_to_anthropic(self.tools)

        # Prepare headers - EXACT match to working curl
        headers = {
            "x-api-key": self.api_key,
            "anthropic-version": "2023-06-01",
            "Content-Type": "application/json",
        }

        # Make request
        url = self._get_endpoint_url()

        last_exc: Optional[Exception] = None
        for attempt in range(MAX_RETRIES):
            try:
                with httpx.Client(timeout=REQUEST_TIMEOUT) as client:
                    response = client.post(url, headers=headers, json=payload)
                    if response.status_code >= 400:
                        raise httpx.HTTPStatusError(
                            f"{response.status_code} error for {url}: {response.text}",
                            request=response.request,
                            response=response,
                        )
                    return response.json()
            except (httpx.ReadTimeout, httpx.ConnectTimeout, httpx.WriteTimeout) as exc:
                last_exc = exc
                if attempt < MAX_RETRIES - 1:
                    time.sleep(RETRY_BACKOFF_SECONDS * (2 ** attempt))
        raise httpx.ReadTimeout(
            f"Azure Anthropic endpoint timed out after {MAX_RETRIES} attempts "
            f"(read timeout={REQUEST_TIMEOUT.read}s) for {url}"
        ) from last_exc

    async def _make_request_async(
        self,
        messages: List[BaseMessage],
        stop: Optional[List[str]] = None,
        **kwargs: Any,
    ) -> Dict[str, Any]:
        """Make async HTTP request to Azure Anthropic endpoint."""
        # Convert messages
        system_msg, anthropic_messages = self._convert_messages_to_anthropic(
            messages
        )

        # Prepare payload
        payload = {
            "model": self.model,
            "max_tokens": self.max_tokens,
            "messages": anthropic_messages,
        }
        if self.temperature is not None:
            payload["temperature"] = self.temperature

        if system_msg:
            payload["system"] = system_msg

        if stop:
            payload["stop_sequences"] = stop

        # Add tools if bound
        if self.tools:
            payload["tools"] = self._convert_tools_to_anthropic(self.tools)

        # Prepare headers - EXACT match to working curl
        headers = {
            "x-api-key": self.api_key,
            "anthropic-version": "2023-06-01",
            "Content-Type": "application/json",
        }

        # Make async request
        url = self._get_endpoint_url()

        last_exc: Optional[Exception] = None
        for attempt in range(MAX_RETRIES):
            try:
                async with httpx.AsyncClient(timeout=REQUEST_TIMEOUT) as client:
                    response = await client.post(url, headers=headers, json=payload)
                    if response.status_code >= 400:
                        raise httpx.HTTPStatusError(
                            f"{response.status_code} error for {url}: {response.text}",
                            request=response.request,
                            response=response,
                        )
                    return response.json()
            except (httpx.ReadTimeout, httpx.ConnectTimeout, httpx.WriteTimeout) as exc:
                last_exc = exc
                if attempt < MAX_RETRIES - 1:
                    await asyncio.sleep(RETRY_BACKOFF_SECONDS * (2 ** attempt))
        raise httpx.ReadTimeout(
            f"Azure Anthropic endpoint timed out after {MAX_RETRIES} attempts "
            f"(read timeout={REQUEST_TIMEOUT.read}s) for {url}"
        ) from last_exc

    def _parse_anthropic_response(self, result: Dict[str, Any]) -> AIMessage:
        """Parse Anthropic response dictionary into LangChain AIMessage."""
        text_parts = []
        tool_calls = []

        for block in result.get("content", []):
            block_type = block.get("type")
            if block_type == "text":
                text_parts.append(block.get("text", ""))
            elif block_type == "tool_use":
                tool_calls.append({
                    "name": block.get("name", ""),
                    "args": block.get("input", {}),
                    "id": block.get("id", ""),
                    "type": "tool_call",
                })

        content = "\n".join(text_parts) if text_parts else ""
        return AIMessage(content=content, tool_calls=tool_calls)

    def _generate(
        self,
        messages: List[BaseMessage],
        stop: Optional[List[str]] = None,
        run_manager: Optional[Any] = None,
        **kwargs: Any,
    ) -> ChatResult:
        """Generate response synchronously."""
        result = self._make_request(messages, stop, **kwargs)
        message = self._parse_anthropic_response(result)
        return ChatResult(generations=[ChatGeneration(message=message)])

    async def _agenerate(
        self,
        messages: List[BaseMessage],
        stop: Optional[List[str]] = None,
        run_manager: Optional[Any] = None,
        **kwargs: Any,
    ) -> ChatResult:
        """Generate response asynchronously."""
        result = await self._make_request_async(messages, stop, **kwargs)
        message = self._parse_anthropic_response(result)
        return ChatResult(generations=[ChatGeneration(message=message)])

    def bind_tools(
        self,
        tools: Sequence[Union[Dict[str, Any], BaseTool]],
        **kwargs: Any,
    ) -> Runnable:
        """Bind tools to this model.

        Returns a new instance with tools attached.
        """
        return self.__class__(
            api_key=self.api_key,
            base_url=self.base_url,
            model=self.model,
            temperature=self.temperature,
            max_tokens=self.max_tokens,
            tools=list(tools),
        )



================================================
FILE: src/api/__init__.py
================================================
"""API module for FastAPI routes."""



================================================
FILE: src/api/routers/__init__.py
================================================
"""API routers for Dev Deep Agent."""



================================================
FILE: src/api/routers/conversation.py
================================================
"""Conversation management router."""
from typing import List, Optional
from datetime import datetime

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field

from services.session_managers import SessionHistoryManager, SessionContextManager

router = APIRouter()

# Lazy initialization - create manager when needed
def get_session_history():
    """Get or create SessionHistoryManager instance."""
    return SessionHistoryManager()

def get_session_context_mgr():
    """Get or create SessionContextManager instance."""
    return SessionContextManager()

class ConversationMessage(BaseModel):
    """Conversation message model."""
    role: str
    content: str
    timestamp: datetime
    job_id: Optional[str] = None


class StoreConversationRequest(BaseModel):
    """Store conversation request model."""
    workspace_id: str
    user_id: str
    session_id: str
    role: str = Field(..., description="Message role: user or assistant")
    content: str = Field(..., description="Message content")
    job_id: Optional[str] = None


@router.post("/store-conversation")
async def store_conversation(request: StoreConversationRequest):
    """
    Store a conversation message.

    **Request:**
    ```json
    {
        "workspace_id": "1",
        "user_id": "user123",
        "session_id": "session_001",
        "role": "user",
        "content": "Generate a TSD for...",
        "job_id": "job_abc123"
    }
    ```
    """
    mgr = get_session_history()
    await mgr.append_message(
        workspace_id=request.workspace_id,
        user_id=request.user_id,
        session_id=request.session_id,
        role=request.role,
        content=request.content,
        job_id=request.job_id,
    )

    return {"status": "success", "message": "Message stored successfully"}


@router.get("/get-conversation", response_model=List[ConversationMessage])
@router.post("/get-conversation", response_model=List[ConversationMessage])
async def get_conversation(
    workspace_id: str,
    user_id: str,
    session_id: str,
    limit: int = 50,
):
    """
    Get conversation history.

    **Query Parameters:**
    - workspace_id: Workspace identifier
    - user_id: User identifier
    - session_id: Session identifier
    - limit: Maximum messages to return (default: 50)
    """
    mgr = get_session_history()
    messages = await mgr.load_history(
        workspace_id=workspace_id,
        user_id=user_id,
        session_id=session_id,
        limit=limit,
    )

    return messages


@router.post("/delete-conversation")
@router.delete("/delete-conversation")
async def delete_conversation(
    workspace_id: str,
    user_id: str,
    session_id: str,
):
    """
    Delete a conversation session.
    """
    mgr = get_session_history()
    await mgr.delete_session(
        workspace_id=workspace_id,
        user_id=user_id,
        session_id=session_id,
    )

    # Soft-delete the session-context row so the title disappears from
    # /list-sessions (which excludes is_deleted records).
    ctx_mgr = get_session_context_mgr()
    await ctx_mgr.delete_context(
        workspace_id=workspace_id,
        user_id=user_id,
        session_id=session_id,
    )

    return {"status": "success", "message": "Conversation deleted successfully"}



================================================
FILE: src/api/routers/health.py
================================================
"""Health check router."""
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()


class HealthResponse(BaseModel):
    """Health check response model."""
    status: str
    service: str
    version: str


@router.get("/health", response_model=HealthResponse)
async def health_check():
    """
    Health check endpoint.

    Returns the service status and version.
    """
    return {
        "status": "healthy",
        "service": "dev-deep-agent",
        "version": "2.0.0",
    }


@router.get("/test-agent")
async def test_agent():
    """Test if Deep Agent can be created."""
    try:
        from agent_builder import create_dev_agent

        # Try to create agent
        agent = create_dev_agent(
            workspace_id="test",
            user_id="test",
            conversation_id="test",
            agent_id=1,
            job_id=None,
        )

        return {
            "status": "success",
            "message": "Deep Agent created successfully",
            "agent_created": agent is not None,
        }
    except Exception as e:
        import traceback
        return {
            "status": "error",
            "message": str(e),
            "traceback": traceback.format_exc(),
        }



================================================
FILE: src/api/routers/integrations.py
================================================
"""Dev-Dynamic agent model and user integration configuration APIs."""
import base64
from typing import Any, Dict, Optional

import httpx

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field

from core.config import settings
from core.database import get_mongo_collection

router = APIRouter(prefix="/integrations", tags=["Dev Integrations"])

COLLECTIONS = {"jira": "user_jira_config", "github": "user_github_config"}

# Provider ids match agent_builder._get_or_create_llm's MODEL_USED switch
# ("openai" -> Azure OpenAI, "claude" -> Claude via Azure Foundry).
PROVIDERS = {"openai", "claude"}


class ConfigRequest(BaseModel):
    workspace_id: str
    user_id: str
    data: Dict[str, Any] = Field(default_factory=dict)


class ToggleRequest(BaseModel):
    workspace_id: str
    user_id: str
    enable: bool = True


class ModelRequest(BaseModel):
    workspace_id: str
    user_id: str
    provider: str
    model: str


def _key(workspace_id: str, user_id: str) -> dict:
    return {"workspace_id": str(workspace_id), "user_id": str(user_id)}


def _clean_config(doc: Optional[dict]) -> dict:
    if not doc:
        return {}
    return {k: v for k, v in doc.items() if k not in {"_id", "workspace_id", "user_id"}}


def _redact_config(config: dict) -> dict:
    result = dict(config)
    for field in ("jira_access_token", "github_token"):
        if result.get(field):
            result[field] = "********"
    return result


async def _get_config(kind: str, workspace_id: str, user_id: str) -> dict:
    doc = await get_mongo_collection(COLLECTIONS[kind]).find_one(_key(workspace_id, user_id))
    return _clean_config(doc)


async def get_runtime_config(workspace_id: str, user_id: str) -> dict:
    """Return request-scoped settings consumed by agent_builder."""
    llm = await get_mongo_collection("user_llm_config").find_one(_key(workspace_id, user_id))
    return {
        "llm": _clean_config(llm),
        "jira": await _get_config("jira", workspace_id, user_id),
        "github": await _get_config("github", workspace_id, user_id),
    }


@router.post("/list")
async def list_integrations(request: ConfigRequest):
    jira = await _get_config("jira", request.workspace_id, request.user_id)
    github = await _get_config("github", request.workspace_id, request.user_id)
    return {"integrations": [
        {"id": "jira", "name": "Jira", "connected": bool(jira.get("active"))},
        {"id": "github", "name": "Github", "connected": bool(github.get("active"))},
    ]}


@router.post("/{kind}/config/get")
async def get_integration_config(kind: str, request: ConfigRequest):
    if kind not in COLLECTIONS:
        raise HTTPException(404, "Unsupported integration")
    return _redact_config(await _get_config(kind, request.workspace_id, request.user_id))


@router.post("/{kind}/config/update")
async def update_integration_config(kind: str, request: ConfigRequest):
    if kind not in COLLECTIONS:
        raise HTTPException(404, "Unsupported integration")
    data = {str(k): v for k, v in request.data.items()}
    # Saving credentials is the connect action in CommonAgent's dialog.
    data["active"] = True
    await get_mongo_collection(COLLECTIONS[kind]).update_one(
        _key(request.workspace_id, request.user_id),
        {"$set": {**_key(request.workspace_id, request.user_id), **data}},
        upsert=True,
    )
    return {"status": "success", "message": f"{kind} configuration saved"}


@router.post("/{kind}/config/toggle")
async def toggle_integration_config(kind: str, request: ToggleRequest):
    if kind not in COLLECTIONS:
        raise HTTPException(404, "Unsupported integration")
    await get_mongo_collection(COLLECTIONS[kind]).update_one(
        _key(request.workspace_id, request.user_id), {"$set": {"active": request.enable}}, upsert=True
    )
    return {"status": "success", "message": f"{kind} connection updated"}


@router.post("/{kind}/test")
async def test_integration(kind: str, request: ConfigRequest):
    if kind not in COLLECTIONS:
        raise HTTPException(404, "Unsupported integration")
    required = {"jira": ("jira_url", "jira_email", "jira_access_token"),
                "github": ("github_token", "github_repo_full_name")} [kind]
    missing = [name for name in required if not str(request.data.get(name, "")).strip()]
    if missing:
        return {"status": "error", "message": f"Missing: {', '.join(missing)}"}
    try:
        async with httpx.AsyncClient(timeout=20) as client:
            if kind == "jira":
                token = base64.b64encode(
                    f"{request.data['jira_email']}:{request.data['jira_access_token']}".encode()
                ).decode()
                response = await client.get(
                    f"{str(request.data['jira_url']).rstrip('/')}/rest/api/3/myself",
                    headers={"Authorization": f"Basic {token}", "Accept": "application/json"},
                )
            else:
                response = await client.get(
                    "https://api.github.com/user",
                    headers={"Authorization": f"Bearer {request.data['github_token']}", "Accept": "application/vnd.github+json"},
                )
            response.raise_for_status()
        return {"status": "success", "message": f"{kind} connection successful"}
    except Exception as exc:
        return {"status": "error", "message": f"{kind} connection failed: {exc}"}


@router.post("/llm/providers")
async def list_llm_providers(request: ConfigRequest):
    selected = await get_mongo_collection("user_llm_config").find_one(_key(request.workspace_id, request.user_id))
    configured = []
    cfg = settings.deep_agent
    if cfg.AZURE_OPENAI_API_KEY and cfg.AZURE_OPENAI_MODEL:
        configured.append({"provider": "openai", "available_models": [cfg.AZURE_OPENAI_MODEL]})
    if cfg.ANTHROPIC_FOUNDRY_API_KEY and cfg.ANTHROPIC_DEFAULT_SONNET_MODEL:
        configured.append({"provider": "claude", "available_models": [cfg.ANTHROPIC_DEFAULT_SONNET_MODEL]})
    current = _clean_config(selected)
    default_provider = (cfg.MODEL_USED or "openai").strip().lower()
    return {"configured_providers": configured,
            "current_provider": current.get("provider") or default_provider,
            "current_model": current.get("model") or (cfg.AZURE_OPENAI_MODEL if default_provider == "openai" else cfg.ANTHROPIC_DEFAULT_SONNET_MODEL)}


@router.post("/llm/switch")
async def switch_llm(request: ModelRequest):
    if request.provider not in PROVIDERS or not request.model.strip():
        raise HTTPException(400, "Invalid model selection")
    await get_mongo_collection("user_llm_config").update_one(
        _key(request.workspace_id, request.user_id),
        {"$set": {**_key(request.workspace_id, request.user_id), "provider": request.provider, "model": request.model}},
        upsert=True,
    )
    return {"status": "success", "provider": request.provider, "model": request.model}



================================================
FILE: src/api/routers/mermaid.py
================================================
"""Mermaid diagram rendering router."""
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field

from services.mermaid import render_mermaid_to_url
from core.config import settings

router = APIRouter()


class RenderMermaidRequest(BaseModel):
    """Render mermaid request model."""
    mermaid_code: str = Field(..., description="Mermaid diagram code")


class RenderMermaidResponse(BaseModel):
    """Render mermaid response model."""
    url: str = Field(..., description="Public URL to rendered diagram")
    status: str = "success"


@router.post("/render-mermaid", response_model=RenderMermaidResponse)
@router.get("/render-mermaid")
async def render_mermaid(request: RenderMermaidRequest = None, mermaid_code: str = None):
    """
    Render Mermaid diagram code to SVG image.

    **POST Request:**
    ```json
    {
        "mermaid_code": "graph TD\\nA-->B"
    }
    ```

    **Response:**
    ```json
    {
        "url": "http://localhost:8000/diagrams/abc123.svg",
        "status": "success"
    }
    ```
    """
    code = mermaid_code if mermaid_code else request.mermaid_code if request else None

    if not code:
        raise HTTPException(status_code=400, detail="mermaid_code is required")

    try:
        url = await render_mermaid_to_url(
            raw_code=code,
            public_base_url=settings.PUBLIC_BASE_URL,
            public_dir=settings.PUBLIC_ASSET_DIR,
            chrome_path=settings.CHROME_EXECUTABLE_PATH,
        )

        return {"url": url, "status": "success"}

    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to render diagram: {str(e)}")



================================================
FILE: src/api/routers/registry.py
================================================
"""Registry router — read-only access to the Postgres agent/workspace/user data."""
from urllib.parse import urlparse

from fastapi import APIRouter, HTTPException

from services import documents, registry
from core.config import settings
from core.logging import logger

router = APIRouter()


async def _download_blob_text(blob_url: str) -> str:
    """Download a blob's UTF-8 text by its full URL, server-side.

    Reads through the Azure SDK with the service credential so the browser never
    needs storage-account CORS or a SAS token — the UI asks the backend for the
    document content and renders it in the chat viewer.
    """
    from azure.storage.blob.aio import BlobClient

    az = settings.azure
    # Prefer the account key; fall back to a connection string, then to the
    # default credential chain (managed identity / az login).
    credential = az.BLOB_ACCOUNT_KEY or None
    if not credential and az.BLOB_STORAGE_CONNECTION_STRING:
        # from_connection_string needs container + blob names, not a full URL, so
        # derive them from the URL path (/{container}/{blob...}).
        parsed = urlparse(blob_url)
        parts = parsed.path.lstrip("/").split("/", 1)
        container = parts[0]
        blob_name = parts[1] if len(parts) > 1 else ""
        client = BlobClient.from_connection_string(
            az.BLOB_STORAGE_CONNECTION_STRING, container, blob_name
        )
    elif credential:
        client = BlobClient.from_blob_url(blob_url, credential=credential)
    else:
        from azure.identity.aio import DefaultAzureCredential

        client = BlobClient.from_blob_url(blob_url, credential=DefaultAzureCredential())

    async with client:
        stream = await client.download_blob()
        data = await stream.readall()
    return data.decode("utf-8", errors="replace")


@router.get("/documents/content")
async def get_document_content(
    url: str = "", session_id: str = "", document_name: str = ""
):
    """Return a generated document's raw markdown for the in-chat viewer.

    Resolve the blob either directly by ``url`` or by looking up
    ``(session_id, document_name)`` in ``fe_agent_document``. The URL is validated
    against the configured storage account to prevent server-side request forgery.
    """
    resolved = url
    if not resolved and session_id and document_name:
        docs = await documents.list_session_documents(session_id)
        wanted = document_name.split("/")[-1]
        match = next(
            (
                d
                for d in docs
                if d.get("document_name") == document_name
                or str(d.get("document_name", "")).split("/")[-1] == wanted
            ),
            None,
        )
        resolved = (match or {}).get("document_url") or ""

    if not resolved:
        raise HTTPException(status_code=404, detail="Document not found")

    # SSRF guard: only allow blobs on our own storage account host.
    account = (settings.azure.BLOB_ACCOUNT_NAME or "").lower()
    host = (urlparse(resolved).hostname or "").lower()
    if not account or host != f"{account}.blob.core.windows.net":
        raise HTTPException(status_code=400, detail="URL not permitted")

    try:
        content = await _download_blob_text(resolved)
    except Exception as exc:  # noqa: BLE001
        logger.error(f"get_document_content failed for {resolved}: {exc}")
        raise HTTPException(status_code=502, detail="Failed to read document")

    return {"url": resolved, "content": content}


@router.get("/workspaces/{workspace_id}")
async def get_workspace(workspace_id: int):
    """Return a workspace_master record."""
    ws = await registry.get_workspace(workspace_id)
    if not ws:
        raise HTTPException(status_code=404, detail=f"Workspace {workspace_id} not found")
    return ws


@router.get("/workspaces/{workspace_id}/agents")
async def list_workspace_agents(workspace_id: int):
    """Return active agents mapped to a workspace."""
    return {"workspace_id": workspace_id, "agents": await registry.list_workspace_agents(workspace_id)}


@router.get("/workspaces/{workspace_id}/users")
async def list_workspace_users(workspace_id: int):
    """Return active users mapped to a workspace."""
    return {"workspace_id": workspace_id, "users": await registry.list_workspace_users(workspace_id)}


@router.get("/agents/{agent_id}")
async def get_agent(agent_id: int):
    """Return an agent's combined details + CMS record."""
    agent = await registry.get_agent(agent_id)
    if not agent:
        raise HTTPException(status_code=404, detail=f"Agent {agent_id} not found")
    return agent


@router.get("/sessions/{session_id}/documents")
async def list_session_documents(session_id: str):
    """Return documents generated during a session (for the UI)."""
    return {
        "session_id": session_id,
        "documents": await documents.list_session_documents(session_id),
    }


@router.get("/workflows/{workflow_job_id}/documents")
async def list_workflow_documents(workflow_job_id: str):
    """Return every agent's shared outputs for a dynamic workflow (for the UI)."""
    return {
        "workflow_job_id": workflow_job_id,
        "documents": await documents.list_workflow_documents(workflow_job_id),
    }


@router.get("/workspaces/{workspace_id}/agents/{agent_folder}/documents")
async def list_agent_documents(workspace_id: str, agent_folder: str):
    """Return an individual agent's outputs within a workspace (for the UI)."""
    return {
        "workspace_id": workspace_id,
        "agent_folder": agent_folder,
        "documents": await documents.list_agent_documents(workspace_id, agent_folder),
    }



================================================
FILE: src/api/routers/session.py
================================================
"""Session management router."""
from typing import Optional
from datetime import datetime

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field

from services.session_managers import SessionContextManager, SessionHistoryManager

router = APIRouter()

# Lazy initialization - create managers when needed
def get_session_context_mgr():
    """Get or create SessionContextManager instance."""
    return SessionContextManager()

def get_session_history_mgr():
    """Get or create SessionHistoryManager instance."""
    return SessionHistoryManager()


class CreateSessionRequest(BaseModel):
    """Create session request model."""
    workspace_id: str = Field(..., description="Workspace identifier")
    user_id: str = Field(..., description="User identifier")
    title: Optional[str] = Field(None, description="Optional session title")


class CreateSessionResponse(BaseModel):
    """Create session response model."""
    session_id: str
    workspace_id: str
    user_id: str
    title: Optional[str] = None
    created_at: datetime


@router.post("/create-session", response_model=CreateSessionResponse)
async def create_session(request: CreateSessionRequest):
    """
    Create a new conversation session.

    **Request:**
    ```json
    {
        "workspace_id": "1",
        "user_id": "user123",
        "title": "User Management API Design"
    }
    ```
    """
    from datetime import datetime
    import uuid

    from services import registry
    from core.logging import logger as _logger

    # Best-effort registry validation: when both ids are numeric, confirm the
    # user is an active member of the workspace. Non-numeric ids (e.g. dev
    # "ui-user") or an unavailable registry skip the check so nothing breaks.
    try:
        wid, uid = int(request.workspace_id), int(request.user_id)
        if not await registry.user_in_workspace(wid, uid):
            raise HTTPException(
                status_code=403,
                detail=f"User {request.user_id} is not a member of workspace {request.workspace_id}",
            )
    except (ValueError, TypeError):
        pass
    except HTTPException:
        raise
    except Exception as e:  # noqa: BLE001
        _logger.warning(f"Workspace membership validation skipped: {e}")

    # Generate session ID (session will be created automatically when first message is sent)
    session_id = f"session_{uuid.uuid4().hex[:16]}"
    created_at = datetime.utcnow()

    return {
        "session_id": session_id,
        "workspace_id": request.workspace_id,
        "user_id": request.user_id,
        "title": request.title,
        "created_at": created_at,
    }


@router.get("/list-sessions")
async def list_sessions(
    workspace_id: str,
    user_id: str,
    limit: int = 100,
):
    """List a user's sessions in a workspace (newest first) for the history sidebar.

    Returns lightweight rows: ``session_id``, ``title`` and activity timestamps.
    Backed by ``SessionContextManager.list_contexts`` (excludes soft-deleted).
    """
    mgr = get_session_context_mgr()
    try:
        contexts = await mgr.list_contexts(
            workspace_id=workspace_id, user_id=user_id, limit=limit
        )
    except Exception as e:  # noqa: BLE001
        from core.logging import logger as _logger

        _logger.warning(f"list_sessions failed: {e}")
        contexts = []

    sessions = [
        {
            "session_id": c.get("session_id"),
            "title": c.get("title") or "Untitled",
            "updated_at": c.get("updated_at") or c.get("last_activity_at"),
            "message_count": c.get("message_count", 0),
        }
        for c in contexts
        if c.get("session_id")
    ]
    return {"workspace_id": workspace_id, "user_id": user_id, "sessions": sessions}


@router.get("/get-synopsis")
async def get_synopsis(
    workspace_id: str,
    user_id: str,
    session_id: str,
):
    """
    Get session synopsis/title.
    """
    mgr = get_session_context_mgr()
    session = await mgr.get_context(
        workspace_id=workspace_id,
        user_id=user_id,
        session_id=session_id,
    )

    if not session:
        raise HTTPException(status_code=404, detail="Session not found")

    return {
        "session_id": session_id,
        "title": session.get("title", "Untitled"),
        "is_custom_title": session.get("is_custom_title", False),
    }


@router.post("/edit-synopsis")
async def edit_synopsis(
    workspace_id: str,
    user_id: str,
    session_id: str,
    title: str,
):
    """
    Edit session synopsis/title.
    """
    mgr = get_session_context_mgr()
    await mgr.set_context(
        workspace_id=workspace_id,
        user_id=user_id,
        session_id=session_id,
        context={"title": title, "is_custom_title": True},
    )

    return {"status": "success", "message": "Title updated successfully"}



================================================
FILE: src/api/routers/skills.py
================================================
"""Skills management router."""
from typing import Dict, Any

from fastapi import APIRouter, HTTPException, UploadFile, File
from pydantic import BaseModel

from services.skills import (
    get_active_skill,
    upload_skill_blob,
    download_skill_blob,
)

router = APIRouter()


class SkillResponse(BaseModel):
    """Skill response model."""
    skill_id: str
    name: str
    version: str
    status: str
    role: str
    objective: str


@router.get("/get-skill", response_model=Dict[str, Any])
async def get_skill(
    workspace_id: str,
    agent_id: int = 1,
    skill_id: str = "dev_scaffolding_generation",
):
    """
    Get active skill configuration.

    **Query Parameters:**
    - workspace_id: Workspace identifier
    - agent_id: Agent identifier (default: 1)
    - skill_id: Skill identifier (default: dev_scaffolding_generation)
    """
    skill = await get_active_skill(
        workspace_id=workspace_id,
        agent_id=agent_id,
        skill_id=skill_id,
    )

    if not skill:
        raise HTTPException(status_code=404, detail="Skill not found")

    return skill


@router.post("/upload-skill")
async def upload_skill(
    workspace_id: str,
    agent_id: int,
    skill_file: UploadFile = File(...),
):
    """
    Upload skill file to blob storage.

    **Form Data:**
    - workspace_id: Workspace identifier
    - agent_id: Agent identifier
    - skill_file: Skill markdown file
    """
    try:
        content = await skill_file.read()
        content_str = content.decode("utf-8")

        await upload_skill_blob(
            workspace_id=workspace_id,
            agent_id=agent_id,
            skill_content=content_str,
        )

        return {
            "status": "success",
            "message": "Skill uploaded successfully",
            "filename": skill_file.filename,
        }

    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to upload skill: {str(e)}")


@router.get("/download-skill")
async def download_skill(
    workspace_id: str,
    agent_id: int,
    skill_id: str,
    version: str = "latest",
):
    """
    Download skill file from blob storage.
    """
    try:
        content = await download_skill_blob(
            workspace_id=workspace_id,
            agent_id=agent_id,
            skill_id=skill_id,
            version=version,
        )

        return {
            "status": "success",
            "content": content,
            "skill_id": skill_id,
            "version": version,
        }

    except Exception as e:
        raise HTTPException(status_code=404, detail=f"Skill not found: {str(e)}")



================================================
FILE: src/aws_s3_backend/__init__.py
================================================
from .cloud_backends import S3Backend, S3Config

__all__ = ["S3Backend", "S3Config"]



================================================
FILE: src/aws_s3_backend/cloud_backends.py
================================================
"""
S3-compatible backend for Deep Agents.
Supports AWS S3, MinIO, and any S3-compatible object storage.
"""

from __future__ import annotations

import asyncio
import fnmatch
import json
import re
import threading
from contextlib import asynccontextmanager
from dataclasses import dataclass
from datetime import datetime, timezone
from pathlib import PurePosixPath
from typing import TYPE_CHECKING, Any, AsyncIterator, Coroutine, TypeVar

import aioboto3
import wcmatch.glob as wcglob
from botocore.config import Config as BotoConfig
from botocore.exceptions import ClientError
from deepagents.backends.protocol import (
    BackendProtocol,
    EditResult,
    FileData,
    FileDownloadResponse,
    FileInfo,
    FileUploadResponse,
    GrepMatch,
    ReadResult,
    WriteResult,
)
from deepagents.backends.utils import (
    check_empty_content,
    perform_string_replacement,
)

if TYPE_CHECKING:
    from types_aiobotocore_s3 import S3Client

__all__ = ["S3Backend", "S3Config", "run_async_safely"]


class _AsyncThread(threading.Thread):
    def __init__(self, coroutine: Coroutine[Any, Any, Any]):
        self.coroutine = coroutine
        self.result = None
        self.exception = None
        super().__init__(daemon=True)

    def run(self):
        try:
            self.result = asyncio.run(self.coroutine)
        except Exception as e:
            self.exception = e


_T = TypeVar("_T")


def run_async_safely(
    coroutine: Coroutine[Any, Any, _T], timeout: float | None = None
) -> _T:
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        loop = None

    if loop and loop.is_running():
        thread = _AsyncThread(coroutine)
        thread.start()
        thread.join(timeout=timeout)
        if thread.is_alive():
            raise TimeoutError(
                "Operation timed out after %f seconds" % timeout
            )
        if thread.exception:
            raise thread.exception
        return thread.result
    else:
        if timeout:
            coroutine = asyncio.wait_for(coroutine, timeout)
        return asyncio.run(coroutine)


@dataclass
class S3Config:
    """Configuration for S3-compatible storage."""

    bucket: str
    prefix: str = ""
    region: str = "us-east-1"
    endpoint_url: str | None = None
    access_key_id: str | None = None
    secret_access_key: str | None = None
    use_ssl: bool = True
    max_pool_connections: int = 50
    connect_timeout: float = 5.0
    read_timeout: float = 30.0
    max_retries: int = 3


class S3Backend(BackendProtocol):
    """S3-compatible backend for Deep Agents file operations.

    Files are stored as JSON objects with the structure:
    {"content": [...lines], "created_at": "...", "modified_at": "..."}
    """

    def __init__(self, config: S3Config) -> None:
        self._config = config
        self._prefix = config.prefix.strip("/")
        if self._prefix:
            self._prefix += "/"

        self._boto_config = BotoConfig(
            region_name=config.region,
            signature_version="s3v4",
            retries={"max_attempts": config.max_retries, "mode": "adaptive"},
            max_pool_connections=config.max_pool_connections,
            connect_timeout=config.connect_timeout,
            read_timeout=config.read_timeout,
        )

        session_kwargs: dict[str, Any] = {}
        if config.access_key_id:
            session_kwargs["aws_access_key_id"] = config.access_key_id
        if config.secret_access_key:
            session_kwargs["aws_secret_access_key"] = config.secret_access_key

        self._session = aioboto3.Session(**session_kwargs)
        self._bucket = config.bucket

    def _s3_key(self, path: str) -> str:
        clean = path.lstrip("/")
        return f"{self._prefix}{clean}"

    def _virtual_path(self, key: str) -> str:
        if self._prefix and key.startswith(self._prefix):
            key = key[len(self._prefix) :]
        return "/" + key.lstrip("/")

    @asynccontextmanager
    async def _client(self) -> AsyncIterator["S3Client"]:
        async with self._session.client(
            "s3",
            config=self._boto_config,
            endpoint_url=self._config.endpoint_url,
            use_ssl=self._config.use_ssl,
        ) as client:
            yield client

    async def _get_file_data(self, path: str) -> dict[str, Any] | None:
        key = self._s3_key(path)
        try:
            async with self._client() as client:
                response = await client.get_object(
                    Bucket=self._bucket, Key=key
                )
                async with response["Body"] as stream:
                    content = await stream.read()
                return json.loads(content.decode("utf-8"))
        except ClientError as e:
            if e.response["Error"]["Code"] == "NoSuchKey":
                return None
            raise

    async def _put_file_data(
        self, path: str, data: dict[str, Any], *, update_modified: bool = True
    ) -> None:
        key = self._s3_key(path)
        if update_modified:
            data["modified_at"] = datetime.now(timezone.utc).isoformat()
        content = json.dumps(data).encode("utf-8")
        async with self._client() as client:
            await client.put_object(
                Bucket=self._bucket,
                Key=key,
                Body=content,
                ContentType="application/json",
            )

    async def _exists(self, path: str) -> bool:
        key = self._s3_key(path)
        try:
            async with self._client() as client:
                await client.head_object(Bucket=self._bucket, Key=key)
            return True
        except ClientError as e:
            if e.response["Error"]["Code"] == "404":
                return False
            raise

    async def _list_keys(self, prefix: str = "") -> list[dict[str, Any]]:
        full_prefix = self._s3_key(prefix)
        results: list[dict[str, Any]] = []
        async with self._client() as client:
            paginator = client.get_paginator("list_objects_v2")
            async for page in paginator.paginate(
                Bucket=self._bucket, Prefix=full_prefix
            ):
                for obj in page.get("Contents", []):
                    results.append(obj)
        return results

    # -------------------------------------------------------------------------
    # BackendProtocol Implementation
    # -------------------------------------------------------------------------

    def ls(self, path: str) -> list[FileInfo]:
        return run_async_safely(self.als_info(path))

    async def als_info(self, path: str) -> list[FileInfo]:
        prefix = path.lstrip("/")
        if prefix and not prefix.endswith("/"):
            prefix += "/"
        objects = await self._list_keys(prefix)
        results: list[FileInfo] = []
        seen_dirs: set[str] = set()
        for obj in objects:
            key = obj["Key"]
            vpath = self._virtual_path(key)
            rel = vpath[len("/" + prefix) :] if prefix else vpath[1:]
            if "/" in rel:
                dir_name = rel.split("/")[0]
                dir_path = "/" + prefix + dir_name + "/"
                if dir_path not in seen_dirs:
                    seen_dirs.add(dir_path)
                    results.append({"path": dir_path, "is_dir": True})
            else:
                results.append(
                    {
                        "path": vpath,
                        "is_dir": False,
                        "size": obj.get("Size", 0),
                        "modified_at": (
                            obj["LastModified"].isoformat()
                            if "LastModified" in obj
                            else None
                        ),
                    }
                )
        results.sort(key=lambda x: x.get("path", ""))
        return results

    def read(
        self, file_path: str, offset: int = 0, limit: int = 2000
    ) -> ReadResult:
        return run_async_safely(self.aread(file_path, offset, limit))

    async def aread(
        self, file_path: str, offset: int = 0, limit: int = 2000
    ) -> ReadResult:
        data = await self._get_file_data(file_path)
        if data is None:
            return ReadResult(error=f"File '{file_path}' not found")
        lines = data.get("content", [])
        if not lines:
            return ReadResult(file_data=FileData(content="", encoding="utf-8"))
        if offset >= len(lines):
            return ReadResult(
                error=f"Line offset {offset} exceeds file length ({len(lines)} lines)"
            )
        selected = lines[offset : offset + limit]
        # IMPORTANT: return RAW (unformatted) content — see the matching,
        # more detailed comment in azure_blob_backend/backend.py's aread()
        # for why applying format_content_with_line_numbers() at this layer
        # is a bug (line-number formatting belongs to the deepagents
        # read_file TOOL middleware, not the backend's read()/aread()).
        content_out = "\n".join(selected)
        return ReadResult(
            file_data=FileData(content=content_out, encoding="utf-8")
        )

    def write(self, file_path: str, content: str) -> WriteResult:
        return run_async_safely(self.awrite(file_path, content))

    async def awrite(self, file_path: str, content: str) -> WriteResult:
        if await self._exists(file_path):
            return WriteResult(
                error=f"Cannot write to {file_path} because it already exists. "
                "Read and then make an edit, or write to a new path."
            )
        now = datetime.now(timezone.utc).isoformat()
        data = {
            "content": content.splitlines(),
            "created_at": now,
            "modified_at": now,
        }
        try:
            await self._put_file_data(file_path, data, update_modified=False)
            return WriteResult(path=file_path)
        except Exception as e:
            return WriteResult(error=f"Error writing file '{file_path}': {e}")

    def edit(
        self,
        file_path: str,
        old_string: str,
        new_string: str,
        replace_all: bool = False,
    ) -> EditResult:
        return run_async_safely(
            self.aedit(file_path, old_string, new_string, replace_all)
        )

    async def aedit(
        self,
        file_path: str,
        old_string: str,
        new_string: str,
        replace_all: bool = False,
    ) -> EditResult:
        data = await self._get_file_data(file_path)
        if data is None:
            return EditResult(error=f"Error: File '{file_path}' not found")
        content = "\n".join(data.get("content", []))
        result = perform_string_replacement(
            content, old_string, new_string, replace_all
        )
        if isinstance(result, str):
            return EditResult(error=result)
        new_content, occurrences = result
        data["content"] = new_content.splitlines()
        try:
            await self._put_file_data(file_path, data)
            return EditResult(path=file_path, occurrences=int(occurrences))
        except Exception as e:
            return EditResult(error=f"Error editing file '{file_path}': {e}")

    def grep_raw(
        self, pattern: str, path: str | None = None, glob: str | None = None
    ) -> list[GrepMatch] | str:
        return run_async_safely(self.agrep_raw(pattern, path, glob))

    async def agrep_raw(
        self, pattern: str, path: str | None = None, glob: str | None = None
    ) -> list[GrepMatch] | str:
        try:
            regex = re.compile(pattern)
        except re.error as e:
            return f"Invalid regex pattern: {e}"
        search_prefix = (path or "/").lstrip("/")
        objects = await self._list_keys(search_prefix)
        matches: list[GrepMatch] = []
        for obj in objects:
            vpath = self._virtual_path(obj["Key"])
            filename = PurePosixPath(vpath).name
            if glob and not wcglob.globmatch(
                filename, glob, flags=wcglob.BRACE
            ):
                continue
            data = await self._get_file_data(vpath)
            if data is None:
                continue
            for line_num, line in enumerate(data.get("content", []), 1):
                if regex.search(line):
                    matches.append(
                        {"path": vpath, "line": line_num, "text": line}
                    )
        return matches

    def glob(self, pattern: str, path: str = "/") -> list[FileInfo]:
        return run_async_safely(self.aglob_info(pattern, path))

    async def aglob_info(
        self, pattern: str, path: str = "/"
    ) -> list[FileInfo]:
        search_prefix = path.lstrip("/")
        objects = await self._list_keys(search_prefix)
        results: list[FileInfo] = []
        for obj in objects:
            vpath = self._virtual_path(obj["Key"])
            rel_path = (
                vpath[len(path) :].lstrip("/") if path != "/" else vpath[1:]
            )
            if fnmatch.fnmatch(rel_path, pattern) or fnmatch.fnmatch(
                vpath, pattern
            ):
                results.append(
                    {
                        "path": vpath,
                        "is_dir": False,
                        "size": obj.get("Size", 0),
                        "modified_at": (
                            obj["LastModified"].isoformat()
                            if "LastModified" in obj
                            else None
                        ),
                    }
                )
        results.sort(key=lambda x: x.get("path", ""))
        return results

    def upload_files(
        self, files: list[tuple[str, bytes]]
    ) -> list[FileUploadResponse]:
        return run_async_safely(self.aupload_files(files))

    async def aupload_files(
        self, files: list[tuple[str, bytes]]
    ) -> list[FileUploadResponse]:
        responses: list[FileUploadResponse] = []
        async with self._client() as client:
            for path, content in files:
                try:
                    key = self._s3_key(path)
                    await client.put_object(
                        Bucket=self._bucket, Key=key, Body=content
                    )
                    responses.append(FileUploadResponse(path=path, error=None))
                except ClientError as e:
                    code = e.response["Error"]["Code"]
                    err = (
                        "permission_denied"
                        if code == "AccessDenied"
                        else "invalid_path"
                    )
                    responses.append(FileUploadResponse(path=path, error=err))
                except Exception:
                    responses.append(
                        FileUploadResponse(path=path, error="invalid_path")
                    )
        return responses

    def download_files(self, paths: list[str]) -> list[FileDownloadResponse]:
        return run_async_safely(self.adownload_files(paths))

    async def adownload_files(
        self, paths: list[str]
    ) -> list[FileDownloadResponse]:
        responses: list[FileDownloadResponse] = []
        async with self._client() as client:
            for path in paths:
                try:
                    key = self._s3_key(path)
                    response = await client.get_object(
                        Bucket=self._bucket, Key=key
                    )
                    async with response["Body"] as stream:
                        content = await stream.read()
                    responses.append(
                        FileDownloadResponse(
                            path=path, content=content, error=None
                        )
                    )
                except ClientError as e:
                    code = e.response["Error"]["Code"]
                    if code == "NoSuchKey":
                        err = "file_not_found"
                    elif code == "AccessDenied":
                        err = "permission_denied"
                    else:
                        err = "invalid_path"
                    responses.append(
                        FileDownloadResponse(
                            path=path, content=None, error=err
                        )
                    )
        return responses



================================================
FILE: src/azure_blob_backend/__init__.py
================================================
from .backend import AzureBlobBackend
from .config import AzureBlobConfig

__all__ = ["AzureBlobBackend", "AzureBlobConfig"]



================================================
FILE: src/azure_blob_backend/_path.py
================================================
"""Path normalization utilities for Azure Blob Storage backend."""

from __future__ import annotations

from deepagents.backends.utils import validate_path


def normalize_path(path: str) -> str:
    if path == "":
        return ""
    normalized = validate_path(path)
    return "" if normalized == "/" else normalized.lstrip("/")


def to_blob_key(prefix: str, path: str) -> str:
    normalized = normalize_path(path)
    if not prefix:
        return normalized
    p = prefix if prefix.endswith("/") else prefix + "/"
    return p + normalized


def from_blob_key(prefix: str, blob_name: str) -> str:
    if prefix:
        p = prefix if prefix.endswith("/") else prefix + "/"
        if blob_name.startswith(p):
            blob_name = blob_name[len(p) :]
    return "/" + blob_name if blob_name else "/"


def get_prefix_for_path(prefix: str, path: str) -> str:
    normalized = normalize_path(path)
    if not prefix and not normalized:
        return ""
    if not prefix:
        return normalized + "/" if normalized else ""
    p = prefix if prefix.endswith("/") else prefix + "/"
    if not normalized:
        return p
    return p + normalized + "/"



================================================
FILE: src/azure_blob_backend/_utils.py
================================================
"""Internal helpers for building FileInfo."""

from __future__ import annotations

from deepagents.backends.protocol import FileInfo


def build_file_info(
    path: str,
    *,
    is_dir: bool = False,
    size: int = 0,
    modified_at: str = "",
) -> FileInfo:
    return {
        "path": path,
        "is_dir": is_dir,
        "size": size,
        "modified_at": modified_at,
    }



================================================
FILE: src/azure_blob_backend/backend.py
================================================
"""Azure Blob Storage backend for Deep Agents."""

from __future__ import annotations

import asyncio
import inspect
import logging
from datetime import datetime, timezone
from types import SimpleNamespace
from typing import Any, Optional

import wcmatch.glob as wcglob
from azure.core.credentials import AzureSasCredential
from azure.core.exceptions import AzureError, ResourceExistsError, ResourceNotFoundError
from azure.storage.blob.aio import BlobServiceClient, ContainerClient
from deepagents.backends.protocol import (
    BackendProtocol,
    EditResult,
    FileData,
    FileDownloadResponse,
    FileInfo,
    FileUploadResponse,
    GlobResult,
    GrepMatch,
    GrepResult,
    LsResult,
    ReadResult,
    WriteResult,
)
from deepagents.backends.utils import (
    perform_string_replacement,
    validate_path,
)

from ._path import from_blob_key, get_prefix_for_path, to_blob_key
from ._utils import build_file_info
from .config import AzureBlobConfig

logger = logging.getLogger(__name__)


class AzureBlobBackend(BackendProtocol):
    """Azure Blob Storage filesystem backend for Deep Agents.

    Implements BackendProtocol using Azure Blob Storage as the persistence
    layer. All file content is stored as raw UTF-8 text in blob bodies, with
    timestamps in blob metadata. Directories are synthesized from blob key
    prefixes.
    """

    def __init__(self, config: AzureBlobConfig) -> None:
        self._config = config
        self._client: Optional[BlobServiceClient] = None
        self._container: Optional[ContainerClient] = None
        self._credential: Optional[Any] = None
        self._init_lock: Optional[asyncio.Lock] = None

    def _get_lock(self) -> asyncio.Lock:
        if self._init_lock is None:
            self._init_lock = asyncio.Lock()
        return self._init_lock

    async def _get_container(self) -> ContainerClient:
        if self._container is not None:
            return self._container

        async with self._get_lock():
            if self._container is not None:
                return self._container

            kwargs: dict[str, Any] = {}
            if self._config.api_version:
                kwargs["api_version"] = self._config.api_version

            if self._config.connection_string:
                self._client = BlobServiceClient.from_connection_string(
                    self._config.connection_string,
                    **kwargs,
                )
            elif self._config.account_key:
                self._client = BlobServiceClient(
                    account_url=self._config.account_url,
                    credential=self._config.account_key,
                    **kwargs,
                )
            elif self._config.sas_token:
                credential = AzureSasCredential(self._config.sas_token)
                self._client = BlobServiceClient(
                    account_url=self._config.account_url,
                    credential=credential,
                    **kwargs,
                )
            elif self._config.credential is not None:
                credential = self._config.credential
                if hasattr(
                    credential, "close"
                ) and inspect.iscoroutinefunction(credential.close):
                    self._credential = credential
                self._client = BlobServiceClient(
                    account_url=self._config.account_url,
                    credential=credential,
                    **kwargs,
                )
            else:
                from azure.identity.aio import DefaultAzureCredential

                credential = DefaultAzureCredential()
                self._credential = credential
                self._client = BlobServiceClient(
                    account_url=self._config.account_url,
                    credential=credential,
                    **kwargs,
                )
            self._container = self._client.get_container_client(
                self._config.container_name,
            )

            # Ensure the container exists (auto-create on first use) so a fresh
            # storage account works without manual provisioning.
            try:
                await self._container.create_container()
                logging.getLogger("azure_blob_backend").info(
                    "Created blob container '%s'", self._config.container_name
                )
            except ResourceExistsError:
                pass
            except AzureError as exc:  # e.g. insufficient permission to create
                logging.getLogger("azure_blob_backend").warning(
                    "Could not ensure container '%s' exists: %s",
                    self._config.container_name,
                    exc,
                )

            return self._container

    async def close(self) -> None:
        if self._client is not None:
            await self._client.close()
            self._client = None
            self._container = None
        if self._credential is not None:
            await self._credential.close()
            self._credential = None

    # ------------------------------------------------------------------
    # Helpers
    # ------------------------------------------------------------------

    def _blob_key(self, path: str) -> str:
        return to_blob_key(self._config.prefix, path)

    def _virtual_path(self, blob_name: str) -> str:
        return from_blob_key(self._config.prefix, blob_name)

    def _validate_file_path(self, path: str) -> str:
        return validate_path(path)

    def _validate_search_path(self, path: str | None) -> str:
        return validate_path(path or "/")

    def _relative_path(self, virtual_path: str, base_path: str) -> str | None:
        if base_path == "/":
            return virtual_path[1:]
        prefix_with_slash = base_path + "/"
        if virtual_path.startswith(prefix_with_slash):
            return virtual_path[len(prefix_with_slash) :]
        if virtual_path == base_path:
            return virtual_path.split("/")[-1]
        return None

    async def _get_listed_blob(
        self, container: ContainerClient, blob_key: str
    ) -> Any | None:
        blob = container.get_blob_client(blob_key)
        try:
            props = await blob.get_blob_properties()
        except ResourceNotFoundError:
            return None
        metadata = dict(props.metadata) if props.metadata else None
        return SimpleNamespace(
            name=blob_key, size=getattr(props, "size", 0), metadata=metadata
        )

    async def _list_target_blobs(
        self, container: ContainerClient, path: str
    ) -> list[Any]:
        if path == "/":
            return await self._list_blobs(
                container, to_blob_key(self._config.prefix, "/")
            )
        exact_blob = await self._get_listed_blob(
            container, self._blob_key(path)
        )
        if exact_blob is not None:
            return [exact_blob]
        return await self._list_blobs(
            container, get_prefix_for_path(self._config.prefix, path)
        )

    async def _blob_exists(
        self, container: ContainerClient, blob_key: str
    ) -> bool:
        blob = container.get_blob_client(blob_key)
        return await blob.exists()

    async def _read_blob(
        self, container: ContainerClient, blob_key: str
    ) -> tuple[str, dict[str, str]]:
        # Download raw bytes (no in-flight decode) so we can handle files saved
        # in non-UTF-8 encodings — common in legacy enterprise codebases (COBOL
        # files touched by Windows editors often contain cp1252 bytes like 0xA0
        # non-breaking space). Passing encoding= to download_blob forces the SDK
        # to decode server-side and crashes on the first invalid byte.
        blob = container.get_blob_client(blob_key)
        stream = await blob.download_blob()
        raw = await stream.readall()
        if isinstance(raw, str):
            content = raw
        else:
            primary = self._config.encoding or "utf-8"
            content = None
            for enc in (primary, "utf-8-sig", "cp1252", "latin-1"):
                try:
                    content = raw.decode(enc)
                    break
                except UnicodeDecodeError:
                    continue
            if content is None:
                # latin-1 maps every byte, so this branch is defensive only.
                content = raw.decode(primary, errors="replace")
        props = await blob.get_blob_properties()
        metadata: dict[str, str] = (
            dict(props.metadata) if props.metadata else {}
        )
        return content, metadata

    async def _register_document_to_db(self, file_path: str) -> None:
        """Record a written blob into the ``fe_agent_document`` table.

        Awaited inline on the write path's live loop so the upsert reliably
        completes before the write returns (a prior fire-and-forget
        ``loop.create_task`` was cancelled when the worker loop closed). Best-effort
        — any failure is logged and swallowed. Called from every blob write path
        (``awrite`` / ``aedit`` / ``aupload_files``).
        """
        from pathlib import Path

        try:
            # ``prefix`` is the blob path segment (may be composite,
            # ``<workspace_id>/<session_id>``); the DB row is keyed by the real
            # session id, falling back to prefix for backward compatibility.
            url_prefix = self._config.prefix  # e.g. "267" or "12/session_ab..."
            db_session_id = self._config.session_id or self._config.prefix
            if not db_session_id or db_session_id.strip() == "":
                return

            document_name = file_path.replace("\\", "/").lstrip("/")
            parts = Path(document_name).parts

            # Only register genuine *generated* documents (the agent's workspace
            # output artifacts). Skip the deep-agent's internal/seed folders and
            # intake inputs, otherwise the fe_agent_document table fills up with
            # noise like ``large_tool_results/toolu_...`` instead of HLD.md/TSD.md.
            INTERNAL_FOLDERS = (
                "skills", "memory", "global", "large_tool_results",
                "codebase", "intake", "conversation_history",
            )
            if (
                any(part in INTERNAL_FOLDERS for part in parts)
                or document_name == "event_history.json"
            ):
                return

            # Only document-like artifacts (guards against stray non-doc writes).
            DOC_SUFFIXES = (".md", ".markdown", ".pdf", ".docx", ".txt", ".csv", ".json", ".html", ".cls", ".xml", ".example", ".py", ".java")
            if Path(document_name).suffix.lower() not in DOC_SUFFIXES:
                return

            # Derive Azure Blob URL from configured account/container.
            from core.config import settings

            container = (
                self._config.container_name
                or settings.azure.BLOB_CONTAINER_NAME
                or "workspaces-deepagent"
            )
            account_name = settings.azure.BLOB_ACCOUNT_NAME
            if not account_name:
                return
            document_url = (
                f"https://{account_name}.blob.core.windows.net/"
                f"{container}/{url_prefix}/{document_name}"
            )

            # The producing agent is the folder immediately under ``workspace/``
            # (e.g. ``workspace/dev/HLD.md`` -> ``dev``). This lets the
            # UI label/group each shared workflow output by the agent that made it.
            agent_folder = None
            if len(parts) >= 3 and parts[0] == "workspace":
                agent_folder = parts[1]
            elif len(parts) >= 2:
                agent_folder = parts[0]

            from services import documents

            try:
                await documents.upsert_document(
                    session_id=db_session_id,
                    document_name=document_name,
                    document_url=document_url,
                    workspace_id=self._config.workspace_id,
                    workflow_job_id=self._config.workflow_job_id,
                    scope=self._config.scope,
                    agent_folder=agent_folder,
                )
                logger.info(
                    "Registered document %s in fe_agent_document.",
                    document_name,
                )
            except Exception as exc:  # noqa: BLE001
                logger.error(
                    "Failed to upsert document %s: %s", document_name, exc
                )
        except Exception as e:  # noqa: BLE001
            logger.warning(
                "Failed to register document to DB for %s: %s", file_path, e
            )

    async def _write_blob(
        self,
        container: ContainerClient,
        blob_key: str,
        content: str,
        *,
        created_at: Optional[str] = None,
        overwrite: bool = True,
    ) -> None:
        now = datetime.now(timezone.utc).isoformat()
        metadata = {"created_at": created_at or now, "modified_at": now}
        blob = container.get_blob_client(blob_key)
        await blob.upload_blob(
            content.encode(self._config.encoding),
            overwrite=overwrite,
            metadata=metadata,
        )

    async def _list_blobs(
        self, container: ContainerClient, prefix: str
    ) -> list[Any]:
        blobs = []
        async for blob in container.list_blobs(
            name_starts_with=prefix or None,
            include=["metadata"],
        ):
            blobs.append(blob)
        return blobs

    # ------------------------------------------------------------------
    # Sync wrappers
    # ------------------------------------------------------------------

    def _isolated(self) -> "AzureBlobBackend":
        """A clone sharing this backend's config but owning its own clients."""
        return AzureBlobBackend(self._config)

    def _run_async(self, make_coro: Any) -> Any:
        """Run an async operation from sync context.

        `make_coro` is a callable taking a backend instance and returning the
        coroutine to await — NOT a coroutine. That indirection matters: the
        operation must run against a backend whose Azure client was created on
        the loop that will await it.

        The previous implementation took an already-bound coroutine and worked
        around the cross-loop problem by nulling `self._client` / `self._container`
        / `self._init_lock`, running the coroutine on a throwaway loop, then
        restoring the saved values in a `finally`. That save/clear/restore
        sequence mutates state shared by every caller, and the eight Phase 2
        codegen sub-agents call this concurrently: thread B would snapshot the
        `None`s that thread A had just installed, and whichever thread finished
        last would restore a client belonging to a loop that was already closed.
        Every subsequent blob call on that instance then failed or hung —
        surfacing as sporadic "cannot get URL" errors, reads that returned
        nothing for files that existed, and codegen agents stalling on a file.

        Each call now runs on a private clone, so no state is shared and no
        restore is needed.
        """
        try:
            loop = asyncio.get_running_loop()
        except RuntimeError:
            loop = None

        async def _run_and_cleanup(backend: "AzureBlobBackend"):
            try:
                return await make_coro(backend)
            finally:
                try:
                    await backend.close()
                except Exception:
                    pass

        if loop is not None and loop.is_running():
            import concurrent.futures

            with concurrent.futures.ThreadPoolExecutor(max_workers=1) as pool:
                future = pool.submit(
                    asyncio.run, _run_and_cleanup(self._isolated())
                )
                return future.result()

        return asyncio.run(_run_and_cleanup(self._isolated()))

    # ------------------------------------------------------------------
    # ls
    # ------------------------------------------------------------------

    def ls(self, path: str) -> LsResult:
        return LsResult(entries=self._run_async(lambda b: b.als_info(path)))

    async def als(self, path: str) -> LsResult:
        return LsResult(entries=await self.als_info(path))

    async def als_info(self, path: str) -> list[FileInfo]:
        try:
            normalized_root = self._validate_search_path(path)
        except ValueError:
            return []

        container = await self._get_container()
        blobs = await self._list_blobs(
            container,
            get_prefix_for_path(self._config.prefix, normalized_root),
        )
        if not blobs:
            return []

        infos: list[FileInfo] = []
        subdirs: set[str] = set()
        normalized_path = (
            normalized_root
            if normalized_root.endswith("/")
            else normalized_root + "/"
        )

        for blob in blobs:
            virtual = self._virtual_path(blob.name)
            if not virtual.startswith(normalized_path):
                continue
            relative = virtual[len(normalized_path) :]
            if not relative:
                continue
            if "/" in relative:
                subdir_name = relative.split("/")[0]
                subdirs.add(normalized_path + subdir_name + "/")
            else:
                modified_at = ""
                if blob.metadata:
                    modified_at = blob.metadata.get("modified_at", "")
                infos.append(
                    build_file_info(
                        path=virtual,
                        is_dir=False,
                        size=blob.size or 0,
                        modified_at=modified_at,
                    )
                )

        for subdir in sorted(subdirs):
            infos.append(build_file_info(path=subdir, is_dir=True, size=0))

        infos.sort(key=lambda x: x.get("path", ""))
        return infos

    # ------------------------------------------------------------------
    # read
    # ------------------------------------------------------------------

    def read(
        self, file_path: str, offset: int = 0, limit: int = 2000
    ) -> ReadResult:
        return self._run_async(lambda b: b.aread(file_path, offset, limit))

    async def aread(
        self, file_path: str, offset: int = 0, limit: int = 2000
    ) -> ReadResult:
        try:
            file_path = self._validate_file_path(file_path)
        except ValueError as exc:
            return ReadResult(error=f"Invalid path '{file_path}': {exc}")

        container = await self._get_container()
        blob_key = self._blob_key(file_path)

        try:
            content, _metadata = await self._read_blob(container, blob_key)
        except ResourceNotFoundError:
            return ReadResult(error=f"File '{file_path}' not found")

        if not content or content.strip() == "":
            return ReadResult(file_data=FileData(content="", encoding="utf-8"))

        lines = content.split("\n")
        if lines and lines[-1] == "":
            lines = lines[:-1]

        if offset >= len(lines):
            return ReadResult(
                error=f"Line offset {offset} exceeds file length ({len(lines)} lines)"
            )

        selected = lines[offset : offset + limit]
        # IMPORTANT: return RAW (unformatted) content here — this matches
        # deepagents' own FilesystemBackend.read() contract ("Returns:
        # ReadResult with raw (unformatted) content... Line-number
        # formatting is applied by the middleware."). Applying
        # format_content_with_line_numbers() at this layer was a bug: the
        # deepagents read_file TOOL (middleware/filesystem.py) already
        # applies that formatting itself for the LLM's benefit, so doing
        # it here too caused every read_file call to show DOUBLE line
        # numbers (e.g. "     1\t     1\t..."), and — much worse — every
        # other caller of aread()/read() that has nothing to do with the
        # LLM's read_file tool (this project's own run_manifest.json /
        # event_history.json reads, and the /api/pipeline/{run_id}/artifacts
        # and /artifact/{key} REST endpoints that serve raw file content
        # directly to the frontend UI for display/download) ALSO got these
        # line-number prefixes baked into what they received — which is
        # exactly why generated .md design docs appeared corrupted in the
        # UI and in downloaded files, even when the LLM never wrote a bad
        # line into the file itself.
        content_out = "\n".join(selected)
        return ReadResult(
            file_data=FileData(content=content_out, encoding="utf-8")
        )

    # ------------------------------------------------------------------
    # write
    # ------------------------------------------------------------------

    def write(self, file_path: str, content: str) -> WriteResult:
        return self._run_async(lambda b: b.awrite(file_path, content))

    async def awrite(self, file_path: str, content: str) -> WriteResult:
        try:
            file_path = self._validate_file_path(file_path)
        except ValueError as exc:
            return WriteResult(error=f"Invalid path '{file_path}': {exc}")

        container = await self._get_container()
        blob_key = self._blob_key(file_path)

        await self._write_blob(container, blob_key, content, overwrite=True)
        await self._register_document_to_db(file_path)
        return WriteResult(path=file_path)

    # ------------------------------------------------------------------
    # edit
    # ------------------------------------------------------------------

    def edit(
        self,
        file_path: str,
        old_string: str,
        new_string: str,
        replace_all: bool = False,
    ) -> EditResult:
        return self._run_async(
            lambda b: b.aedit(file_path, old_string, new_string, replace_all)
        )

    async def aedit(
        self,
        file_path: str,
        old_string: str,
        new_string: str,
        replace_all: bool = False,
    ) -> EditResult:
        try:
            file_path = self._validate_file_path(file_path)
        except ValueError as exc:
            return EditResult(error=f"Invalid path '{file_path}': {exc}")

        container = await self._get_container()
        blob_key = self._blob_key(file_path)

        try:
            content, metadata = await self._read_blob(container, blob_key)
        except ResourceNotFoundError:
            return EditResult(error=f"Error: File '{file_path}' not found")

        result = perform_string_replacement(
            content, old_string, new_string, replace_all
        )
        if isinstance(result, str):
            return EditResult(error=result)

        new_content, occurrences = result
        created_at = metadata.get("created_at")
        await self._write_blob(
            container, blob_key, new_content, created_at=created_at
        )
        await self._register_document_to_db(file_path)
        return EditResult(path=file_path, occurrences=int(occurrences))

    # ------------------------------------------------------------------
    # glob
    # ------------------------------------------------------------------

    def glob(self, pattern: str, path: str = "/") -> GlobResult:
        return GlobResult(
            matches=self._run_async(lambda b: b.aglob_info(pattern, path))
        )

    async def aglob(self, pattern: str, path: str = "/") -> GlobResult:
        return GlobResult(matches=await self.aglob_info(pattern, path))

    async def aglob_info(
        self, pattern: str, path: str = "/"
    ) -> list[FileInfo]:
        try:
            normalized_path = self._validate_search_path(path)
        except ValueError:
            return []

        container = await self._get_container()
        blobs = await self._list_target_blobs(container, normalized_path)
        if not blobs:
            return []

        infos: list[FileInfo] = []
        for blob in blobs:
            virtual = self._virtual_path(blob.name)
            relative = self._relative_path(virtual, normalized_path)
            if relative is None:
                continue
            if wcglob.globmatch(
                relative, pattern, flags=wcglob.BRACE | wcglob.GLOBSTAR
            ):
                modified_at = ""
                if blob.metadata:
                    modified_at = blob.metadata.get("modified_at", "")
                infos.append(
                    build_file_info(
                        path=virtual,
                        is_dir=False,
                        size=blob.size or 0,
                        modified_at=modified_at,
                    )
                )
        return infos

    # ------------------------------------------------------------------
    # grep_raw
    # ------------------------------------------------------------------

    def grep_raw(
        self, pattern: str, path: str | None = None, glob: str | None = None
    ) -> list[GrepMatch] | str:
        return self._run_async(lambda b: b.agrep_raw(pattern, path, glob))

    async def agrep_raw(
        self, pattern: str, path: str | None = None, glob: str | None = None
    ) -> list[GrepMatch] | str:
        try:
            search_path = self._validate_search_path(path)
        except ValueError as exc:
            invalid_path = path if path is not None else "/"
            return f"Error: Invalid path '{invalid_path}': {exc}"

        container = await self._get_container()
        blobs = await self._list_target_blobs(container, search_path)
        if not blobs:
            return []

        blob_candidates: list[tuple[Any, str]] = []
        for blob in blobs:
            virtual = self._virtual_path(blob.name)
            relative = self._relative_path(virtual, search_path)
            if relative is None:
                continue
            blob_candidates.append((blob, relative))

        if glob:
            blob_candidates = [
                (blob, relative)
                for blob, relative in blob_candidates
                if wcglob.globmatch(
                    relative, glob, flags=wcglob.BRACE | wcglob.GLOBSTAR
                )
            ]

        matches: list[GrepMatch] = []
        failed_blobs: list[str] = []
        semaphore = asyncio.Semaphore(self._config.max_concurrency)

        async def search_blob(blob: Any) -> list[GrepMatch]:
            async with semaphore:
                try:
                    blob_client = container.get_blob_client(blob.name)
                    stream = await blob_client.download_blob(
                        encoding=self._config.encoding
                    )
                    content = str(await stream.readall())
                except (AzureError, UnicodeError) as exc:
                    logger.warning(
                        "Failed to read blob %s for grep: %s", blob.name, exc
                    )
                    failed_blobs.append(self._virtual_path(blob.name))
                    return []

                virtual = self._virtual_path(blob.name)
                blob_matches: list[GrepMatch] = []
                for line_num, line in enumerate(content.split("\n"), 1):
                    if pattern in line:
                        blob_matches.append(
                            {"path": virtual, "line": line_num, "text": line}
                        )
                return blob_matches

        results = await asyncio.gather(
            *(search_blob(blob) for blob, _ in blob_candidates)
        )
        for blob_matches in results:
            matches.extend(blob_matches)

        if failed_blobs:
            failed_blobs.sort()
            sample = ", ".join(failed_blobs[:3])
            remainder = len(failed_blobs) - min(len(failed_blobs), 3)
            suffix = f", and {remainder} more" if remainder else ""
            return f"Error: grep could not read {len(failed_blobs)} file(s): {sample}{suffix}"

        return matches

    def grep(
        self, pattern: str, path: str | None = None, glob: str | None = None
    ) -> GrepResult:
        result = self._run_async(lambda b: b.agrep_raw(pattern, path, glob))
        if isinstance(result, str):
            return GrepResult(error=result)
        return GrepResult(matches=result)

    async def agrep(
        self, pattern: str, path: str | None = None, glob: str | None = None
    ) -> GrepResult:
        result = await self.agrep_raw(pattern, path, glob)
        if isinstance(result, str):
            return GrepResult(error=result)
        return GrepResult(matches=result)

    # ------------------------------------------------------------------
    # upload_files / download_files
    # ------------------------------------------------------------------

    def upload_files(
        self, files: list[tuple[str, bytes]]
    ) -> list[FileUploadResponse]:
        return self._run_async(lambda b: b.aupload_files(files))

    async def aupload_files(
        self, files: list[tuple[str, bytes]]
    ) -> list[FileUploadResponse]:
        container = await self._get_container()
        responses: list[FileUploadResponse] = []
        for file_path, content in files:
            try:
                file_path = self._validate_file_path(file_path)
            except ValueError:
                responses.append(
                    FileUploadResponse(path=file_path, error="invalid_path")
                )
                continue
            blob_key = self._blob_key(file_path)
            now = datetime.now(timezone.utc).isoformat()
            metadata = {"created_at": now, "modified_at": now}
            try:
                blob = container.get_blob_client(blob_key)
                await blob.upload_blob(
                    content, overwrite=True, metadata=metadata
                )
                await self._register_document_to_db(file_path)
                responses.append(
                    FileUploadResponse(path=file_path, error=None)
                )
            except Exception as exc:
                logger.error("Failed to upload %s: %s", file_path, exc)
                responses.append(
                    FileUploadResponse(
                        path=file_path, error="permission_denied"
                    )
                )
        return responses

    def download_files(self, paths: list[str]) -> list[FileDownloadResponse]:
        return self._run_async(lambda b: b.adownload_files(paths))

    async def adownload_files(
        self, paths: list[str]
    ) -> list[FileDownloadResponse]:
        container = await self._get_container()
        responses: list[FileDownloadResponse] = []
        for file_path in paths:
            try:
                file_path = self._validate_file_path(file_path)
            except ValueError:
                responses.append(
                    FileDownloadResponse(
                        path=file_path, content=None, error="invalid_path"
                    )
                )
                continue
            blob_key = self._blob_key(file_path)
            try:
                blob = container.get_blob_client(blob_key)
                stream = await blob.download_blob()
                raw = await stream.readall()
                content_bytes = (
                    raw if isinstance(raw, bytes) else raw.encode("utf-8")
                )
                responses.append(
                    FileDownloadResponse(
                        path=file_path, content=content_bytes, error=None
                    )
                )
            except ResourceNotFoundError:
                responses.append(
                    FileDownloadResponse(
                        path=file_path, content=None, error="file_not_found"
                    )
                )
        return responses



================================================
FILE: src/azure_blob_backend/config.py
================================================
"""Configuration for the Azure Blob Storage backend."""

from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Optional


@dataclass
class AzureBlobConfig:
    """Configuration for AzureBlobBackend.

    Five authentication methods are supported (mutually exclusive):

    1. **Connection string** -- set ``connection_string`` only.
    2. **Account key** -- set ``account_url`` + ``account_key``.
    3. **SAS token** -- set ``account_url`` + ``sas_token``.
    4. **Credential object** -- set ``account_url`` + ``credential``.
    5. **Default (AAD)** -- set ``account_url`` only.
    """

    account_url: str = ""
    container_name: str = ""
    prefix: str = ""
    # The real per-conversation session id. Decoupled from ``prefix`` (which may
    # be a composite ``<workspace_id>/<session_id>`` blob path) so document rows
    # in ``fe_agent_document`` are still keyed by the true session id.
    session_id: str = ""
    # Run-restructure metadata carried onto each registered document row so the
    # UI can group shared workflow outputs and scope individual-agent outputs.
    workspace_id: Optional[str] = None
    workflow_job_id: Optional[str] = None
    scope: Optional[str] = None
    credential: Any = None
    account_key: Optional[str] = None
    sas_token: Optional[str] = None
    max_concurrency: int = 8
    encoding: str = "utf-8"
    connection_string: Optional[str] = None
    api_version: str = "2025-11-05"

    def __post_init__(self) -> None:
        for field_name in ("connection_string", "account_key", "sas_token"):
            value = getattr(self, field_name)
            if value is not None and not value.strip():
                raise ValueError(
                    f"{field_name} must be None or a non-empty string, got empty string."
                )

        if self.sas_token is not None:
            normalized = self.sas_token.strip().lstrip("?")
            if not normalized:
                raise ValueError("sas_token contains only '?' characters.")
            self.sas_token = normalized

        cred_sources = [
            ("connection_string", self.connection_string is not None),
            ("account_key", self.account_key is not None),
            ("sas_token", self.sas_token is not None),
            ("credential", self.credential is not None),
        ]
        active = [name for name, is_set in cred_sources if is_set]

        if len(active) > 1:
            raise ValueError(
                f"Only one authentication method may be set, got: {', '.join(active)}."
            )

        if self.connection_string and self.account_url.strip():
            raise ValueError(
                "connection_string and account_url are mutually exclusive."
            )

        if not self.connection_string and not self.account_url.strip():
            raise ValueError(
                "account_url is required unless connection_string is provided."
            )



================================================
FILE: src/core/__init__.py
================================================
"""
Core module for the REST API
Contains shared utilities, configuration, and base classes
"""
# Auth module is for Azure Functions only - not used in FastAPI
# from .auth import decode_and_verify_token, extract_bearer_token, require_auth
from .config import get_secret, load_secrets_to_settings, settings
from .database import (
    DatabaseManager,
    get_db_manager,  # Use the lazy getter
    get_mongo_collection,
    get_workflow_collection,
    setup_mongodb_collections,
)
from .exceptions import (
    AgentException,
    APIException,
    AuthenticationException,
    AuthorizationException,
    BusinessLogicException,
    ConfigurationException,
    ConflictException,
    DatabaseErrorHandler,
    DatabaseException,
    ErrorResponse,
    ExternalServiceException,
    MCPException,
    NotFoundException,
    RateLimitException,
    TimeoutException,
    ValidationException,
    handle_azure_function_exceptions,
)
from .logging import (
    Logger,
    log_debug,
    log_error,
    log_info,
    log_warning,
    logger,
    setup_logging,
)
# Middleware module is for Azure Functions only - not used in FastAPI
# from .middleware import AzureFunctionMiddleware, azure_function_middleware

__all__ = [
    # Config
    "settings",
    "get_secret",
    "load_secrets_to_settings",
    # Logging
    "Logger",
    "setup_logging",
    "logger",
    "log_info",
    "log_error",
    "log_warning",
    "log_debug",
    # Database
    "get_db_manager",  # Lazy database manager getter
    "DatabaseManager",
    "get_mongo_collection",
    "get_workflow_collection",
    "setup_mongodb_collections",
    # Exceptions
    "APIException",
    "ValidationException",
    "AuthenticationException",
    "AuthorizationException",
    "NotFoundException",
    "ConflictException",
    "RateLimitException",
    "DatabaseException",
    "ExternalServiceException",
    "MCPException",
    "ConfigurationException",
    "AgentException",
    "BusinessLogicException",
    "TimeoutException",
    "ErrorResponse",
    "handle_azure_function_exceptions",
    "DatabaseErrorHandler",
    # Auth - commented out, Azure Functions only
    # "require_auth",
    # "decode_and_verify_token",
    # "extract_bearer_token",
]


# Module-level __getattr__ for backward compatibility with lazy db_manager
def __getattr__(name):
    """Lazy module-level attribute access for backward compatibility."""
    if name == "db_manager":
        # Import from database module, which has its own __getattr__
        from .database import db_manager
        return db_manager
    raise AttributeError(f"module '{__name__}' has no attribute '{name}'")



================================================
FILE: src/core/config.py
================================================
"""
Configuration management for the REST API
Handles environment variables, secrets, and application settings
"""
import os
from typing import List

from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
from pydantic import Field, validator
from pydantic_settings import BaseSettings


def _account_name_from_connection_string(conn_str: str) -> str:
    """Extract ``AccountName`` from an Azure Storage connection string.

    The connection string embeds ``AccountName=<name>`` among ``;``-separated
    key=value pairs. Used as a fallback so document URLs can be built even when a
    standalone ``BLOB_ACCOUNT_NAME`` env var isn't set.
    """
    for part in (conn_str or "").split(";"):
        key, _, value = part.partition("=")
        if key.strip().lower() == "accountname" and value.strip():
            return value.strip()
    return ""


class DatabaseSettings(BaseSettings):
    """Database configuration settings"""

    # SQL Database settings
    SQL_SERVER: str = Field(default="", env="SQL_SERVER")
    SQL_DATABASE: str = Field(default="", env="SQL_DATABASE")
    SQL_USERNAME: str = Field(default="", env="SQL_USERNAME")
    SQL_PASSWORD: str = Field(default="", env="SQL_PASSWORD")
    SQL_DRIVER: str = Field(default="ODBC Driver 18 for SQL Server", env="SQL_DRIVER")

    # MongoDB settings
    MONGODB_DATABASE_URI: str = Field(default="", env="MONGODB_DATABASE_URI")
    DEV_DB_NAME: str = Field(default="", env="DEV_DB_NAME")
    MONGODB_WORKFLOW_DATABASE_NAME: str = Field(
        default="", env="MONGODB_WORKFLOW_DATABASE_NAME"
    )
    DEV_CHAT_COLLECTION: str = Field(default="", env="DEV_CHAT_COLLECTION")
    DEV_SESSION_CONTEXT: str = Field(default="", env="DEV_SESSION_CONTEXT")

    # Durable live-streaming event log (persists SSE/WebSocket events per job so a
    # client can replay after a process restart or across instances).
    DEV_STREAM_EVENTS_COLLECTION: str = Field(
        default="devdynamic_stream_events", env="DEV_STREAM_EVENTS_COLLECTION"
    )
    # Long-term agent memory (durable per-session/user notes the agent reads/writes
    # across conversations; distinct from the design-snapshot store).
    DEV_AGENT_MEMORY_COLLECTION: str = Field(
        default="devdynamic_agent_memory", env="DEV_AGENT_MEMORY_COLLECTION"
    )
    # LangGraph durable checkpointer collections (graph state per thread_id).
    DEV_CHECKPOINTS_COLLECTION: str = Field(
        default="devdynamic_checkpoints", env="DEV_CHECKPOINTS_COLLECTION"
    )
    DEV_CHECKPOINT_WRITES_COLLECTION: str = Field(
        default="devdynamic_checkpoint_writes", env="DEV_CHECKPOINT_WRITES_COLLECTION"
    )

    # Connection pool settings
    MAX_POOL_SIZE: int = Field(default=20, env="DB_MAX_POOL_SIZE")
    MIN_POOL_SIZE: int = Field(default=5, env="DB_MIN_POOL_SIZE")
    POOL_TIMEOUT: int = Field(default=30, env="DB_POOL_TIMEOUT")

    @property
    def sql_connection_string(self) -> str:
        """Build SQL Server connection string"""
        if not all([self.SQL_SERVER, self.SQL_DATABASE]):
            return ""

        conn_str = (
            f"mssql+pyodbc://{self.SQL_USERNAME}:{self.SQL_PASSWORD}"
            f"@{self.SQL_SERVER}/{self.SQL_DATABASE}"
            f"?driver={self.SQL_DRIVER.replace(' ', '+')}"
            f"&timeout={self.POOL_TIMEOUT}"
        )
        return conn_str


class PostgresSettings(BaseSettings):
    """PostgreSQL registry configuration (agent / workspace / user tables).

    Read-only lookups over the ForgeX registry. Table names are config-driven
    (like the Mongo collection names) so environments can point at differently
    named tables. Defaults reflect the live schema discovered in ``ForgeX_V2``.
    """

    POSTGRESQL_DATABASE_HOST: str = Field(default="", env="POSTGRESQL_DATABASE_HOST")
    POSTGRESQL_DATABASE_PORT: int = Field(default=5432, env="POSTGRESQL_DATABASE_PORT")
    POSTGRESQL_DATABASE_DATABASE: str = Field(
        default="", env="POSTGRESQL_DATABASE_DATABASE"
    )
    POSTGRESQL_DATABASE_USER: str = Field(default="", env="POSTGRESQL_DATABASE_USER")
    POSTGRESQL_DATABASE_PASSWORD: str = Field(
        default="", env="POSTGRESQL_DATABASE_PASSWORD"
    )

    # Registry table names. Defaults match the actual tables in the live DB
    # (note: the physical names are pluralized vs. the informal names).
    PG_AGENT_DETAILS_TABLE: str = Field(
        default="agents_details", env="POSTGRESQL_DATABASE_AGENT_DETAILS_TABLE"
    )
    PG_AGENT_CMS_TABLE: str = Field(
        default="agents_cms", env="POSTGRESQL_DATABASE_AGENT_CMS_TABLE"
    )
    PG_WORKSPACE_TABLE: str = Field(
        default="workspace_master", env="POSTGRESQL_DATABASE_WORKSPACE_TABLE"
    )
    PG_WORKSPACE_AGENT_MAPPING_TABLE: str = Field(
        default="workspace_agents_mapping_2",
        env="POSTGRESQL_DATABASE_WORKSPACE_AGENT_MAPPING_TABLE",
    )
    PG_WORKSPACE_USER_MAPPING_TABLE: str = Field(
        default="workspace_users_mapping",
        env="POSTGRESQL_DATABASE_WORKSPACE_USER_MAPPING_TABLE",
    )
    PG_USER_TABLE: str = Field(
        default="users", env="POSTGRESQL_DATABASE_USER_TABLE"
    )
    PG_DOCUMENT_TABLE: str = Field(
        default="fe_agent_document", env="POSTGRESQL_DATABASE_DOCUMENT_TABLE"
    )

    # Connection pool sizing (mirrors DatabaseSettings knobs).
    PG_MIN_POOL_SIZE: int = Field(default=1, env="POSTGRESQL_MIN_POOL_SIZE")
    PG_MAX_POOL_SIZE: int = Field(default=10, env="POSTGRESQL_MAX_POOL_SIZE")
    PG_CONNECT_TIMEOUT: int = Field(default=30, env="POSTGRESQL_CONNECT_TIMEOUT")

    @property
    def dsn(self) -> str:
        """Async DSN for asyncpg. Empty string when host is not configured."""
        if not all(
            [
                self.POSTGRESQL_DATABASE_HOST,
                self.POSTGRESQL_DATABASE_DATABASE,
                self.POSTGRESQL_DATABASE_USER,
            ]
        ):
            return ""
        return (
            f"postgresql://{self.POSTGRESQL_DATABASE_USER}:{self.POSTGRESQL_DATABASE_PASSWORD}"
            f"@{self.POSTGRESQL_DATABASE_HOST}:{self.POSTGRESQL_DATABASE_PORT}"
            f"/{self.POSTGRESQL_DATABASE_DATABASE}"
        )


class AzureSettings(BaseSettings):
    """Azure-specific configuration settings"""

    # Azure Functions settings
    AZURE_STORAGE_CONNECTION_STRING: str = Field(default="", env="AzureWebJobsStorage")

    # Azure Key Vault settings
    KEYVAULT_URL: str = Field(default="", env="KEYVAULT_URL")

    # Azure Service Bus settings
    SERVICE_BUS_CONNECTION_STRING: str = Field(
        default="", env="SERVICE_BUS_CONNECTION_STRING"
    )

    # Azure Application Insights
    APPINSIGHTS_INSTRUMENTATION_KEY: str = Field(
        default="", env="APPINSIGHTS_INSTRUMENTATION_KEY"
    )

    # Azure Blob Storage
    BLOB_STORAGE_CONNECTION_STRING: str = Field(
        default=(
            os.getenv("BLOB_STORAGE_CONNECTION_STRING")
            or os.getenv("AZURE_BLOB_STORAGE_CONNECTION_STRING")
            or ""
        ),
        env="BLOB_STORAGE_CONNECTION_STRING",
    )
    BLOB_CONTAINER_NAME: str = Field(
        default=(
            os.getenv("BLOB_CONTAINER_NAME")
            or os.getenv("AZURE_BLOB_STORAGE_CONTAINER_NAME")
            or "documents"
        ),
        env="BLOB_CONTAINER_NAME",
    )
    # Account-name/key auth (alternative to connection string) for the Azure Blob
    # deep-agent backend, mirroring the ForwardEngineering reference.
    BLOB_ACCOUNT_NAME: str = Field(
        default=(
            os.getenv("BLOB_ACCOUNT_NAME")
            or _account_name_from_connection_string(
                os.getenv("BLOB_STORAGE_CONNECTION_STRING")
                or os.getenv("AZURE_BLOB_STORAGE_CONNECTION_STRING")
                or ""
            )
        ),
        env="BLOB_ACCOUNT_NAME",
    )
    BLOB_ACCOUNT_KEY: str = Field(
        default=(os.getenv("BLOB_ACCOUNT_KEY") or ""),
        env="BLOB_ACCOUNT_KEY",
    )

    # --- Skill blob storage (per SKILL_BLOB_STORAGE.md) ---------------------
    # Containers are configured with underscores in settings but Azure normalizes
    # them to hyphens in actual container names.
    SKILL_BLOB_DEV_CONTAINER: str = Field(
        default=(os.getenv("SKILL_BLOB_DEV_CONTAINER") or "skill_dev"),
        env="SKILL_BLOB_DEV_CONTAINER",
    )
    SKILL_BLOB_STAGE_CONTAINER: str = Field(
        default=(os.getenv("SKILL_BLOB_STAGE_CONTAINER") or "skill_stage"),
        env="SKILL_BLOB_STAGE_CONTAINER",
    )
    # Optional override to force a single container across envs.
    SKILL_BLOB_CONTAINER_NAME: str = Field(
        default=(os.getenv("SKILL_BLOB_CONTAINER_NAME") or ""),
        env="SKILL_BLOB_CONTAINER_NAME",
    )
    ARTIFACT_BLOB_CONTAINER_NAME: str = Field(
        default="devdynamic-artifacts", env="ARTIFACT_BLOB_CONTAINER_NAME"
    )
    ARTIFACT_BLOB_PATH_PREFIX: str = Field(
        default="devdynamic-zips", env="ARTIFACT_BLOB_PATH_PREFIX"
    )
    ARTIFACT_BLOB_URL_EXPIRY_MINUTES: int = Field(
        default=5, env="ARTIFACT_BLOB_URL_EXPIRY_MINUTES"
    )


class RenderSettings(BaseSettings):
    """Mermaid render configuration (identical keys to the MCP server)"""

    PUBLIC_BASE_URL: str = Field(default="", env="PUBLIC_BASE_URL")
    PUBLIC_ASSET_DIR: str = Field(default="public/diagrams", env="PUBLIC_ASSET_DIR")
    CHROME_EXECUTABLE_PATH: str = Field(default="", env="CHROME_EXECUTABLE_PATH")


class MCPSettings(BaseSettings):
    """MCP (Model Context Protocol) client configuration.

    The REST API connects to the Dev MCP server (FastMCP over the
    Streamable HTTP transport) as a client and proxies tool calls. Connections
    are cached per ``(session_id, tab_id)`` inside a warm Functions instance;
    see ``agent.mcp_client``.
    """

    # Base URL of the MCP server, e.g. "https://arch-mcp.example.com".
    # The transport path below is appended to it.
    MCP_SERVER_URL: str = Field(default="", env="MCP_SERVER_URL")
    # Streamable HTTP transport path mounted by the server (see mcp/.../main.py).
    MCP_ENDPOINT_PATH: str = Field(default="/mcp", env="MCP_ENDPOINT_PATH")

    # Seconds to wait for a brand-new session to connect + initialize.
    MCP_CONNECT_TIMEOUT: int = Field(default=30, env="MCP_CONNECT_TIMEOUT")
    # Seconds to wait for a single tool call to return. ``send_message`` polls
    # the agent internally, so this must comfortably exceed the agent runtime.
    MCP_TOOL_TIMEOUT: int = Field(default=230, env="MCP_TOOL_TIMEOUT")
    # Idle time after which a cached connection is closed and evicted. Keep this
    # well under the JWT lifetime so a cached session never outlives its token.
    MCP_IDLE_TTL: int = Field(default=300, env="MCP_IDLE_TTL")
    # Hard cap on simultaneously cached connections per instance (back-pressure).
    MCP_MAX_CONNECTIONS: int = Field(default=200, env="MCP_MAX_CONNECTIONS")

    @property
    def endpoint_url(self) -> str:
        """Full Streamable HTTP endpoint, e.g. ``https://host/mcp``."""
        if not self.MCP_SERVER_URL:
            return ""
        return f"{self.MCP_SERVER_URL.rstrip('/')}/{self.MCP_ENDPOINT_PATH.lstrip('/')}"


class WorkflowSettings(BaseSettings):
    """Workflow and dev persistence settings."""

    WORKFLOW_COLLECTION_NAME: str = Field(
        default="workflow_stg_jobs", env="WORKFLOW_COLLECTION_NAME"
    )
    WORKFLOW_ARTIFACT_COLLECTION_NAME: str = Field(
        default="workflow_stg_artifacts", env="WORKFLOW_ARTIFACT_COLLECTION_NAME"
    )
    DEV_DESIGN_COLLECTION: str = Field(
        default="devdynamic_design_snapshots", env="DEV_DESIGN_COLLECTION"
    )


class DevSettings(BaseSettings):
    """Dev service runtime settings."""

    ARCH_EXPORT_DIR: str = Field(default="", env="ARCH_EXPORT_DIR")


class LLMRouterSettings(BaseSettings):
    """LLM router defaults used when request context is missing."""

    DEFAULT_WORKSPACE_ID: int = Field(default=1, env="DEFAULT_WORKSPACE_ID")
    DEFAULT_AGENT_ID: int = Field(default=1, env="DEFAULT_AGENT_ID")


class ProgressSettings(BaseSettings):
    """Progress reporting and WebSocket relay configuration."""

    # Progress backend selection
    # Options: "local_relay", "azure_service_bus", "aws_eventbridge", "log", "none", "auto"
    PROGRESS_BACKEND: str = Field(default="auto", env="PROGRESS_BACKEND")

    # Local relay configuration (development)
    PROGRESS_LOCAL_RELAY_URL: str = Field(
        default="http://127.0.0.1:8090/publish", env="PROGRESS_LOCAL_RELAY_URL"
    )

    # Azure Service Bus configuration (production)
    PROGRESS_TOPIC: str = Field(
        default="devdynamic-progress", env="PROGRESS_TOPIC"
    )
    PROGRESS_QUEUE: str = Field(default="", env="PROGRESS_QUEUE")  # Optional fallback

    # AWS EventBridge configuration (AWS production)
    PROGRESS_EVENT_BUS: str = Field(default="default", env="PROGRESS_EVENT_BUS")

    # WebSocket relay configuration
    RELAY_PORT: int = Field(default=8092, env="RELAY_PORT")
    RELAY_SUBSCRIPTION_NAME: str = Field(
        default="relay-sub", env="RELAY_SUBSCRIPTION_NAME"
    )
    BROADCAST_MODE: str = Field(
        default="session", env="BROADCAST_MODE"
    )  # "session" or "tab"


class SecuritySettings(BaseSettings):
    """Security and authentication settings"""

    # JWT settings
    JWT_SECRET_KEY: str = Field(
        default="your-secret-key-change-in-production", env="JWT_SECRET_KEY"
    )
    JWT_ALGORITHM: str = Field(default="HS256", env="JWT_ALGORITHM")
    JWT_EXPIRATION_MINUTES: int = Field(default=60, env="JWT_EXPIRATION_MINUTES")

    # API Key settings
    API_KEY_HEADER: str = Field(default="X-API-Key", env="API_KEY_HEADER")
    ALLOWED_API_KEYS: List[str] = Field(default_factory=list, env="ALLOWED_API_KEYS")

    # CORS settings
    CORS_ORIGINS: List[str] = Field(default=["*"], env="CORS_ORIGINS")
    CORS_METHODS: List[str] = Field(
        default=["GET", "POST", "PUT", "DELETE"], env="CORS_METHODS"
    )

    @validator("ALLOWED_API_KEYS", pre=True)
    def parse_api_keys(cls, v):
        if isinstance(v, str):
            return [key.strip() for key in v.split(",") if key.strip()]
        return v

    @validator("CORS_ORIGINS", pre=True)
    def parse_cors_origins(cls, v):
        if isinstance(v, str):
            return [origin.strip() for origin in v.split(",") if origin.strip()]
        return v


class DeepAgentSettings(BaseSettings):
    """Deep Agent framework configuration"""

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        case_sensitive = False
        extra = "ignore"

    # Cloud storage provider (filesystem | azure | s3)
    CLOUD_STORAGE_PROVIDER: str = Field(default="azure", env="CLOUD_STORAGE_PROVIDER")

    # Workspace root (for filesystem backend)
    WORKSPACE_ROOT: str = Field(default="./workspaces", env="WORKSPACE_ROOT")

    # Azure Blob backend (inherits from AzureSettings BLOB_STORAGE_CONNECTION_STRING)
    SKILL_BLOB_CONTAINER: str = Field(default="skills", env="SKILL_BLOB_CONTAINER")
    ARTIFACT_BLOB_CONTAINER: str = Field(default="artifacts", env="ARTIFACT_BLOB_CONTAINER")
    # Optional override for the container that holds per-session deep-agent
    # workspaces (outputs/inputs). When empty, the Azure backend uses
    # AzureSettings.BLOB_CONTAINER_NAME (env AZURE_BLOB_STORAGE_CONTAINER_NAME).
    # Each session is stored under a blob prefix = session id.
    WORKSPACE_BLOB_CONTAINER: str = Field(default="", env="WORKSPACE_BLOB_CONTAINER")

    # AWS S3 backend
    AWS_S3_BUCKET_NAME: str = Field(default="", env="AWS_S3_BUCKET_NAME")
    AWS_ACCESS_KEY_ID: str = Field(default="", env="AWS_ACCESS_KEY_ID")
    AWS_SECRET_ACCESS_KEY: str = Field(default="", env="AWS_SECRET_ACCESS_KEY")
    AWS_DEFAULT_REGION: str = Field(default="us-east-1", env="AWS_DEFAULT_REGION")

    # LLM configuration
    LLM_OPTION_NAME_UTILS_AGENTIC: str = Field(default="azure-openai", env="LLM_OPTION_NAME_UTILS_AGENTIC")
    MODEL_USED: str = Field(default="openai", env="MODEL_USED")
    DEFAULT_WORKSPACE_ID: int = Field(default=1, env="DEFAULT_WORKSPACE_ID")
    DEFAULT_AGENT_ID: int = Field(default=1, env="DEFAULT_AGENT_ID")

    # Azure OpenAI API configuration
    AZURE_OPENAI_API_KEY: str = Field(default="", env="AZURE_OPENAI_API_KEY")
    AZURE_OPENAI_BASE_URL: str = Field(default="", env="AZURE_OPENAI_BASE_URL")
    AZURE_OPENAI_MODEL: str = Field(default="", env="AZURE_OPENAI_MODEL")

    # Anthropic API configuration (via Azure Foundry)
    ANTHROPIC_FOUNDRY_API_KEY: str = Field(default="", env="ANTHROPIC_FOUNDRY_API_KEY")
    ANTHROPIC_DEFAULT_SONNET_MODEL: str = Field(default="claude-sonnet-4-5-forgex-rnd", env="ANTHROPIC_DEFAULT_SONNET_MODEL")
    ANTHROPIC_FOUNDRY_BASE_URL: str = Field(default="", env="ANTHROPIC_FOUNDRY_BASE_URL")

    # Shell backend (for local development)
    ENABLE_SHELL_BACKEND: bool = Field(default=False, env="ENABLE_SHELL_BACKEND")


class KnowledgeBaseSettings(BaseSettings):
    """LightRAG / KBCurator knowledge-base configuration."""

    KBCURATOR_URL: str = Field(default="", env="KBCURATOR_URL")

    # Optional explicit query endpoint. When empty, it is derived from
    # KBCURATOR_URL as "<base>/api/query-rag".
    KBCURATOR_QUERY_ENDPOINT: str = Field(default="", env="KBCURATOR_QUERY_ENDPOINT")

    KB_QUERY_TIMEOUT: int = Field(default=30, env="KB_QUERY_TIMEOUT")


class Settings(BaseSettings):
    """Main application settings"""

    # Application settings
    APP_NAME: str = Field(default="forgex-rest-api", env="APP_NAME")
    VERSION: str = Field(default="0.1.0", env="APP_VERSION")
    ENVIRONMENT: str = Field(default="development", env="ENVIRONMENT")
    # Some environments incorrectly set DEBUG=INFO/WARN/etc. Keep parsing lenient
    # so settings initialization never fails at import time.
    DEBUG: bool = Field(default=True, env="DEBUG")

    # Server settings
    HOST: str = Field(default="0.0.0.0", env="HOST")
    PORT: int = Field(default=8000, env="PORT")
    # Auto-reload is decoupled from DEBUG. On Windows the uvicorn reloader tears
    # the process down with `OSError: [WinError 10038]` and kills in-flight SSE
    # streams mid-generation, so it defaults OFF even in DEBUG. Set RELOAD=true
    # explicitly (non-Windows dev) if you want hot-reload.
    RELOAD: bool = Field(default=False, env="RELOAD")

    # Logging settings
    LOG_LEVEL: str = Field(default="INFO", env="LOG_LEVEL")
    LOG_FORMAT: str = Field(default="json", env="LOG_FORMAT")

    # Request settings
    REQUEST_TIMEOUT: int = Field(default=30, env="REQUEST_TIMEOUT")
    MAX_REQUEST_SIZE: int = Field(default=10485760, env="MAX_REQUEST_SIZE")  # 10MB

    # Rate limiting
    RATE_LIMIT_REQUESTS: int = Field(default=100, env="RATE_LIMIT_REQUESTS")
    RATE_LIMIT_WINDOW: int = Field(default=60, env="RATE_LIMIT_WINDOW")  # seconds

    # Agent architecture settings
    AGENT_ENDPOINT: str = Field(default="", env="AGENT_ENDPOINT")
    MCP_SERVER_URL: str = Field(default="", env="MCP_SERVER_URL")

    # External service settings
    EXTERNAL_API_TIMEOUT: int = Field(default=30, env="EXTERNAL_API_TIMEOUT")
    EXTERNAL_API_RETRIES: int = Field(default=3, env="EXTERNAL_API_RETRIES")

    # Nested settings
    database: DatabaseSettings = DatabaseSettings()
    postgres: PostgresSettings = PostgresSettings()
    azure: AzureSettings = AzureSettings()
    security: SecuritySettings = SecuritySettings()
    render: RenderSettings = RenderSettings()
    mcp: MCPSettings = MCPSettings()
    workflow: WorkflowSettings = WorkflowSettings()
    dev: DevSettings = DevSettings()
    llm_router: LLMRouterSettings = LLMRouterSettings()
    progress: ProgressSettings = ProgressSettings()
    deep_agent: DeepAgentSettings = DeepAgentSettings()
    knowledge_base: KnowledgeBaseSettings = KnowledgeBaseSettings()

    @validator("LOG_LEVEL")
    def validate_log_level(cls, v):
        valid_levels = ["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]
        if v.upper() not in valid_levels:
            raise ValueError(f"LOG_LEVEL must be one of {valid_levels}")
        return v.upper()

    @validator("DEBUG", pre=True)
    def validate_debug(cls, v):
        if isinstance(v, bool):
            return v
        if v is None:
            return True
        s = str(v).strip().lower()
        if s in {"1", "true", "t", "yes", "y", "on"}:
            return True
        if s in {"0", "false", "f", "no", "n", "off"}:
            return False
        # Treat garbage as False (safer default) rather than failing startup.
        return False

    @validator("ENVIRONMENT")
    def validate_environment(cls, v):
        valid_envs = ["development", "staging", "production"]
        if v.lower() not in valid_envs:
            raise ValueError(f"ENVIRONMENT must be one of {valid_envs}")
        return v.lower()

    @property
    def is_production(self) -> bool:
        """Check if running in production environment"""
        return self.ENVIRONMENT == "production"

    @property
    def is_development(self) -> bool:
        """Check if running in development environment"""
        return self.ENVIRONMENT == "development"

    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        case_sensitive = False

        # Tests / dev environments may have extra keys in their environment (or
        # local .env) from other services. Ignore unknown keys rather than
        # crashing import-time settings initialization.
        extra = "ignore"


class KeyVaultManager:
    """Manages Azure Key Vault operations for secrets"""

    def __init__(self, keyvault_url: str):
        self.keyvault_url = keyvault_url
        self.credential = DefaultAzureCredential()
        self.client = (
            SecretClient(vault_url=keyvault_url, credential=self.credential)
            if keyvault_url
            else None
        )

    def get_secret(self, secret_name: str, default_value: str = "") -> str:
        """Retrieve secret from Key Vault"""
        if not self.client:
            return default_value

        try:
            secret = self.client.get_secret(secret_name)
            return secret.value
        except Exception:
            # Log error and return default
            return default_value

    def set_secret(self, secret_name: str, secret_value: str) -> bool:
        """Set secret in Key Vault"""
        if not self.client:
            return False

        try:
            self.client.set_secret(secret_name, secret_value)
            return True
        except Exception:
            return False


# Initialize settings
settings = Settings()

# Initialize Key Vault manager if URL is provided
keyvault_manager = (
    KeyVaultManager(settings.azure.KEYVAULT_URL)
    if settings.azure.KEYVAULT_URL
    else None
)


def get_secret(secret_name: str, default_value: str = "") -> str:
    """Get secret from Key Vault or environment variable"""
    if keyvault_manager:
        return keyvault_manager.get_secret(secret_name, default_value)
    return os.getenv(secret_name, default_value)


def load_secrets_to_settings():
    """Load secrets from Key Vault and update settings"""
    if not keyvault_manager:
        return

    # Load database secrets
    if not settings.database.SQL_PASSWORD:
        settings.database.SQL_PASSWORD = get_secret("SQL-PASSWORD")

    if not settings.database.MONGODB_DATABASE_URI:
        settings.database.MONGODB_DATABASE_URI = get_secret("MONGODB-DATABASE-URI")

    # Load JWT secret
    if settings.security.JWT_SECRET_KEY == "your-secret-key-change-in-production":
        settings.security.JWT_SECRET_KEY = get_secret(
            "JWT-SECRET-KEY", settings.security.JWT_SECRET_KEY
        )


# Load secrets on import
load_secrets_to_settings()



================================================
FILE: src/core/database.py
================================================
"""
Database configuration and connection management.

The REST API is MongoDB-only (via Motor, the async driver). `DatabaseManager`
owns a single async client and exposes the dev/workflow databases plus a
collection accessor. SQL support was removed as it is unused.
"""
import certifi
import motor.motor_asyncio
from pymongo.server_api import ServerApi

from typing import Optional

from .config import settings
from .logging import Logger

logger = Logger("database")


class DatabaseManager:
    """Central manager for the async MongoDB connection."""

    def __init__(self):
        self.mongo_client = None
        self.mongo_db = None
        self.workflow_mongo_db = None
        self._initialized = False

    async def initialize(self):
        """Initialize the MongoDB connection (idempotent)."""
        if self._initialized:
            return

        try:
            if settings.database.MONGODB_DATABASE_URI:
                await self._initialize_mongodb()

            self._initialized = True
            logger.info("Database connections initialized successfully")

        except Exception as e:
            logger.error("Failed to initialize database connections", error=e)
            raise

    async def _initialize_mongodb(self):
        """Initialize MongoDB connection"""
        try:
            self.mongo_client = motor.motor_asyncio.AsyncIOMotorClient(
                settings.database.MONGODB_DATABASE_URI,
                server_api=ServerApi("1"),
                tlsCAFile=certifi.where(),
                maxPoolSize=settings.database.MAX_POOL_SIZE,
                minPoolSize=settings.database.MIN_POOL_SIZE,
                serverSelectionTimeoutMS=settings.database.POOL_TIMEOUT * 1000,
            )

            self.mongo_db = self.mongo_client[settings.database.DEV_DB_NAME]
            self.workflow_mongo_db = self.mongo_client[
                settings.database.MONGODB_WORKFLOW_DATABASE_NAME
            ]

            # Test connection
            await self.mongo_client.admin.command("ping")

            logger.info("MongoDB connection initialized")

        except Exception as e:
            logger.error("Failed to initialize MongoDB connection", error=e)
            raise

    def get_mongo_collection(self, collection_name: str):
        """Get a collection from the dev database."""
        if self.mongo_db is None:
            raise RuntimeError("MongoDB not initialized")
        return self.mongo_db[collection_name]

    def get_workflow_collection(self, collection_name: str):
        """Get a collection from the workflow database."""
        if self.workflow_mongo_db is None:
            raise RuntimeError("MongoDB not initialized")
        return self.workflow_mongo_db[collection_name]

    async def close_connections(self):
        """Close the MongoDB connection."""
        if self.mongo_client:
            self.mongo_client.close()

        self.workflow_mongo_db = None
        self._initialized = False
        logger.info("Database connections closed")


# Global database manager instance (lazy initialization)
_db_manager: Optional[DatabaseManager] = None


def get_db_manager() -> DatabaseManager:
    """Get or create the global database manager instance (lazy initialization)."""
    global _db_manager
    if _db_manager is None:
        _db_manager = DatabaseManager()
    return _db_manager


def _mongo_collection_indexes() -> dict:
    """Index specs keyed by the real, config-driven collection names.

    The collection names come from settings (``DEV_CHAT_COLLECTION`` /
    ``DEV_SESSION_CONTEXT``) — the same values consumed by the session
    managers — so the indexes always land on the collections the app actually
    reads and writes. Index keys mirror the query patterns in
    ``session_managers.py``; see ``mongo_models.py`` for the document schemas.
    """
    return {
        # dev_chat_collection (ChatMessage): see SessionHistoryManager.
        settings.database.DEV_CHAT_COLLECTION: {
            "indexes": [
                # load_history / delete_session: filter by the session triple,
                # sorted by timestamp.
                [
                    ("workspace_id", 1),
                    ("user_id", 1),
                    ("session_id", 1),
                    ("timestamp", 1),
                ],
                # get_recent_sessions: filter by (workspace, user, job_id),
                # newest first.
                [("workspace_id", 1), ("user_id", 1), ("job_id", 1), ("timestamp", -1)],
            ]
        },
        # dev_session_context (SessionContext): see SessionContextManager.
        settings.database.DEV_SESSION_CONTEXT: {
            "indexes": [
                # set_context upserts on this triple, and get/delete query it.
                {
                    "keys": [("workspace_id", 1), ("user_id", 1), ("session_id", 1)],
                    "unique": True,
                },
            ]
        },
        # dev_jobs (send_message async job store): see services.dev_jobs.
        "dev_jobs": {
            "indexes": [
                # Every job lookup / update is keyed by job_id.
                {"keys": [("job_id", 1)], "unique": True},
                # Conversation-level cancel queries active jobs by this field.
                [("request.conversation_id", 1), ("status", 1)],
            ]
        },
        # dev_stream_events (durable SSE/WebSocket event log): see
        # services.stream_events. Replay reads events for a job ordered by seq.
        settings.database.DEV_STREAM_EVENTS_COLLECTION: {
            "indexes": [
                # Replay: fetch a job's events in emission order.
                {"keys": [("job_id", 1), ("seq", 1)], "unique": True},
            ]
        },
        # dev_agent_memory (durable long-term agent memory): see
        # services.agent_memory. Keyed by the memory scope + key.
        settings.database.DEV_AGENT_MEMORY_COLLECTION: {
            "indexes": [
                {
                    "keys": [
                        ("workspace_id", 1),
                        ("user_id", 1),
                        ("conversation_id", 1),
                        ("key", 1),
                    ],
                    "unique": True,
                },
            ]
        },
    }


# Module-level __getattr__ to make db_manager lazy
def __getattr__(name):
    """Lazy module-level attribute access for backward compatibility."""
    if name == "db_manager":
        return get_db_manager()
    raise AttributeError(f"module '{__name__}' has no attribute '{name}'")


async def setup_mongodb_collections():
    """Setup MongoDB collections and indexes"""
    manager = get_db_manager()
    if manager.mongo_db is None:
        await manager.initialize()

    for collection_name, schema in _mongo_collection_indexes().items():
        if not collection_name:
            # Collection name not configured; skip rather than create a
            # garbage empty-named collection.
            continue
        collection = manager.get_mongo_collection(collection_name)

        # Create indexes
        for index_spec in schema.get("indexes", []):
            if isinstance(index_spec, dict):
                keys = index_spec["keys"]
                options = {k: v for k, v in index_spec.items() if k != "keys"}
                await collection.create_index(keys, **options)
            else:
                await collection.create_index(index_spec)

    logger.info("MongoDB collections and indexes created")


# Helper functions
def get_mongo_collection(collection_name: str):
    """Get MongoDB collection - convenience function"""
    manager = get_db_manager()
    return manager.get_mongo_collection(collection_name)


def get_workflow_collection(collection_name: str):
    """Get workflow MongoDB collection - convenience function"""
    manager = get_db_manager()
    return manager.get_workflow_collection(collection_name)


async def init_db():
    """Initialize database connections (FastAPI lifespan wrapper)."""
    manager = get_db_manager()
    try:
        await manager.initialize()
        # Setup collections and indexes (optional - will create on first use if not exists)
        try:
            await setup_mongodb_collections()
        except Exception as e:
            logger.warning(f"Failed to setup MongoDB collections (will be created on demand): {e}")
    except Exception as e:
        logger.error(f"Failed to initialize database: {e}")
        # Don't fail startup - allow server to run even if DB is unavailable
        logger.warning("Server starting without database connection")


async def close_db():
    """Close database connections (FastAPI lifespan wrapper)."""
    manager = get_db_manager()
    await manager.close_connections()



================================================
FILE: src/core/exceptions.py
================================================
"""
Custom exceptions and error handling for the REST API
"""
import traceback
from datetime import datetime
from typing import Any, Dict, Optional

from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

from .logging import Logger

logger = Logger("exceptions")


class ErrorResponse(BaseModel):
    """Standard error response model"""

    error: str
    message: str
    details: Optional[Dict[str, Any]] = None
    timestamp: datetime
    correlation_id: Optional[str] = None
    error_code: Optional[str] = None


class APIException(Exception):
    """Base API exception class"""

    def __init__(
        self,
        message: str,
        status_code: int = 500,
        error_code: Optional[str] = None,
        details: Optional[Dict[str, Any]] = None,
    ):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code or self.__class__.__name__
        self.details = details or {}
        super().__init__(self.message)


class ValidationException(APIException):
    """Exception for validation errors"""

    def __init__(
        self,
        message: str = "Validation failed",
        details: Optional[Dict[str, Any]] = None,
    ):
        super().__init__(
            message=message,
            status_code=400,
            error_code="VALIDATION_ERROR",
            details=details,
        )


class AuthenticationException(APIException):
    """Exception for authentication errors"""

    def __init__(self, message: str = "Authentication failed"):
        super().__init__(
            message=message, status_code=401, error_code="AUTHENTICATION_ERROR"
        )


class AuthorizationException(APIException):
    """Exception for authorization errors"""

    def __init__(self, message: str = "Access forbidden"):
        super().__init__(
            message=message, status_code=403, error_code="AUTHORIZATION_ERROR"
        )


class NotFoundException(APIException):
    """Exception for resource not found errors"""

    def __init__(
        self, message: str = "Resource not found", resource_type: str = "resource"
    ):
        super().__init__(
            message=message,
            status_code=404,
            error_code="NOT_FOUND",
            details={"resource_type": resource_type},
        )


class ConflictException(APIException):
    """Exception for conflict errors"""

    def __init__(
        self, message: str = "Resource conflict", resource_type: str = "resource"
    ):
        super().__init__(
            message=message,
            status_code=409,
            error_code="CONFLICT",
            details={"resource_type": resource_type},
        )


class RateLimitException(APIException):
    """Exception for rate limiting errors"""

    def __init__(self, message: str = "Rate limit exceeded"):
        super().__init__(
            message=message, status_code=429, error_code="RATE_LIMIT_EXCEEDED"
        )


class DatabaseException(APIException):
    """Exception for database errors"""

    def __init__(self, message: str = "Database error", operation: str = "unknown"):
        super().__init__(
            message=message,
            status_code=500,
            error_code="DATABASE_ERROR",
            details={"operation": operation},
        )


class ExternalServiceException(APIException):
    """Exception for external service errors"""

    def __init__(
        self, message: str = "External service error", service: str = "unknown"
    ):
        super().__init__(
            message=message,
            status_code=502,
            error_code="EXTERNAL_SERVICE_ERROR",
            details={"service": service},
        )


class MCPException(APIException):
    """Exception for errors talking to the Dev MCP server.

    Raised by the MCP client (``agent.mcp_client``) on connection failures,
    tool-call timeouts, or tool-call errors. ``tool_name`` is recorded in
    ``details`` when the failure relates to a specific tool invocation.
    """

    def __init__(
        self,
        message: str = "MCP error",
        tool_name: Optional[str] = None,
    ):
        details = {}
        if tool_name:
            details["tool_name"] = tool_name
        super().__init__(
            message=message,
            status_code=502,
            error_code="MCP_ERROR",
            details=details,
        )


class ConfigurationException(APIException):
    """Exception for configuration errors"""

    def __init__(
        self, message: str = "Configuration error", config_key: str = "unknown"
    ):
        super().__init__(
            message=message,
            status_code=500,
            error_code="CONFIGURATION_ERROR",
            details={"config_key": config_key},
        )


class AgentException(APIException):
    """Exception for agent-related errors"""

    def __init__(
        self,
        message: str = "Agent error",
        agent_id: Optional[str] = None,
        agent_type: Optional[str] = None,
    ):
        details = {}
        if agent_id:
            details["agent_id"] = agent_id
        if agent_type:
            details["agent_type"] = agent_type

        super().__init__(
            message=message, status_code=500, error_code="AGENT_ERROR", details=details
        )


class BusinessLogicException(APIException):
    """Exception for business logic errors"""

    def __init__(self, message: str = "Business logic error", rule: str = "unknown"):
        super().__init__(
            message=message,
            status_code=422,
            error_code="BUSINESS_LOGIC_ERROR",
            details={"rule": rule},
        )


class TimeoutException(APIException):
    """Exception for timeout errors"""

    def __init__(self, message: str = "Operation timed out", timeout_seconds: int = 0):
        super().__init__(
            message=message,
            status_code=408,
            error_code="TIMEOUT",
            details={"timeout_seconds": timeout_seconds},
        )


async def api_exception_handler(request: Request, exc: APIException) -> JSONResponse:
    """Handle API exceptions"""
    correlation_id = getattr(request.state, "correlation_id", None)

    # Log the error
    logger.error(
        f"API Exception: {exc.message}",
        error=exc,
        status_code=exc.status_code,
        error_code=exc.error_code,
        correlation_id=correlation_id,
        path=str(request.url.path),
        method=request.method,
        details=exc.details,
    )

    # Create error response
    error_response = ErrorResponse(
        error=exc.error_code,
        message=exc.message,
        details=exc.details,
        timestamp=datetime.utcnow(),
        correlation_id=correlation_id,
        error_code=exc.error_code,
    )

    return JSONResponse(status_code=exc.status_code, content=error_response.dict())


async def http_exception_handler(request: Request, exc: HTTPException) -> JSONResponse:
    """Handle HTTP exceptions"""
    correlation_id = getattr(request.state, "correlation_id", None)

    # Log the error
    logger.error(
        f"HTTP Exception: {exc.detail}",
        status_code=exc.status_code,
        correlation_id=correlation_id,
        path=str(request.url.path),
        method=request.method,
    )

    # Create error response
    error_response = ErrorResponse(
        error="HTTP_ERROR",
        message=str(exc.detail),
        timestamp=datetime.utcnow(),
        correlation_id=correlation_id,
        error_code=f"HTTP_{exc.status_code}",
    )

    return JSONResponse(status_code=exc.status_code, content=error_response.dict())


async def validation_exception_handler(
    request: Request, exc: Exception
) -> JSONResponse:
    """Handle Pydantic validation exceptions"""
    correlation_id = getattr(request.state, "correlation_id", None)

    # Extract validation errors
    if hasattr(exc, "errors"):
        validation_errors = []
        for error in exc.errors():
            validation_errors.append(
                {
                    "field": ".".join(str(x) for x in error["loc"]),
                    "message": error["msg"],
                    "type": error["type"],
                }
            )
        details = {"validation_errors": validation_errors}
    else:
        details = {"error": [{"message": str(exc)}]}

    # Log the error
    logger.error(
        "Validation Exception",
        error=exc,
        correlation_id=correlation_id,
        path=str(request.url.path),
        method=request.method,
        details=details,
    )

    # Create error response
    error_response = ErrorResponse(
        error="VALIDATION_ERROR",
        message="Request validation failed",
        details=details,
        timestamp=datetime.utcnow(),
        correlation_id=correlation_id,
        error_code="VALIDATION_ERROR",
    )

    return JSONResponse(status_code=422, content=error_response.dict())


async def general_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    """Handle general exceptions"""
    correlation_id = getattr(request.state, "correlation_id", None)

    # Log the error with full traceback
    logger.error(
        f"Unhandled Exception: {str(exc)}",
        error=exc,
        correlation_id=correlation_id,
        path=str(request.url.path),
        method=request.method,
        traceback=traceback.format_exc(),
    )

    # Create error response (don't expose internal error details in production)
    error_response = ErrorResponse(
        error="INTERNAL_SERVER_ERROR",
        message="An internal server error occurred",
        timestamp=datetime.utcnow(),
        correlation_id=correlation_id,
        error_code="INTERNAL_SERVER_ERROR",
    )

    return JSONResponse(status_code=500, content=error_response.dict())


def setup_exception_handlers(app):
    """Setup exception handlers for FastAPI app"""
    app.add_exception_handler(APIException, api_exception_handler)
    app.add_exception_handler(HTTPException, http_exception_handler)
    app.add_exception_handler(ValueError, validation_exception_handler)
    app.add_exception_handler(Exception, general_exception_handler)


# Decorator for handling exceptions in Azure Functions
def handle_azure_function_exceptions(func):
    """Decorator to handle exceptions in Azure Functions"""

    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except APIException as e:
            logger.error(f"Azure Function API Exception: {e.message}", error=e)
            return {
                "statusCode": e.status_code,
                "body": {
                    "error": e.error_code,
                    "message": e.message,
                    "details": e.details,
                    "timestamp": datetime.utcnow().isoformat(),
                },
                "headers": {"Content-Type": "application/json"},
            }
        except Exception as e:
            logger.error(f"Azure Function Unhandled Exception: {str(e)}", error=e)
            return {
                "statusCode": 500,
                "body": {
                    "error": "INTERNAL_SERVER_ERROR",
                    "message": "An internal server error occurred",
                    "timestamp": datetime.utcnow().isoformat(),
                },
                "headers": {"Content-Type": "application/json"},
            }

    return wrapper


# Context manager for database operations
class DatabaseErrorHandler:
    """Context manager for handling database errors"""

    def __init__(self, operation: str = "database operation"):
        self.operation = operation

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            # Convert database-specific exceptions to our custom exceptions
            if "connection" in str(exc_val).lower():
                raise DatabaseException(
                    message="Database connection failed", operation=self.operation
                )
            elif "timeout" in str(exc_val).lower():
                raise TimeoutException(
                    message="Database operation timed out", timeout_seconds=30
                )
            else:
                raise DatabaseException(
                    message=f"Database error during {self.operation}",
                    operation=self.operation,
                )
        return False



================================================
FILE: src/core/logging.py
================================================
"""
Centralized logging system for the REST API
"""
import logging
import sys
from datetime import datetime
from typing import Any, Dict, Optional

import structlog

from .config import settings


def setup_logging() -> None:
    """Configure structured logging for the application"""
    # Configure structlog
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer(),
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )

    # Configure standard library logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, settings.LOG_LEVEL.upper(), logging.INFO),
    )


class Logger:
    """Centralized logger class with structured logging capabilities"""

    def __init__(self, name: str = None):
        self.logger = structlog.get_logger(name or "rest-api")

    def _add_context(self, **kwargs) -> Dict[str, Any]:
        """Add common context to all log messages"""
        return {
            "app": settings.APP_NAME,
            "environment": settings.ENVIRONMENT,
            "version": "0.1.0",
            "timestamp": datetime.utcnow().isoformat(),
            **kwargs,
        }

    def info(self, message: str, **kwargs) -> None:
        """Log info level message"""
        self.logger.info(message, **self._add_context(**kwargs))

    def warning(self, message: str, **kwargs) -> None:
        """Log warning level message"""
        self.logger.warning(message, **self._add_context(**kwargs))

    def error(self, message: str, error: Optional[Exception] = None, **kwargs) -> None:
        """Log error level message"""
        context = self._add_context(**kwargs)
        if error:
            context.update(
                {
                    "error_message": str(error),
                    "error_type": type(error).__name__,
                    "error_module": getattr(error, "__module__", None),
                }
            )
        self.logger.error(message, **context)

    def debug(self, message: str, **kwargs) -> None:
        """Log debug level message"""
        self.logger.debug(message, **self._add_context(**kwargs))

    def critical(self, message: str, **kwargs) -> None:
        """Log critical level message"""
        self.logger.critical(message, **self._add_context(**kwargs))

    def log_request(
        self, method: str, path: str, user_id: Optional[str] = None, **kwargs
    ) -> None:
        """Log HTTP request"""
        self.info(
            "HTTP Request",
            request_method=method,
            request_path=path,
            user_id=user_id,
            **kwargs,
        )

    def log_response(self, status_code: int, response_time_ms: float, **kwargs) -> None:
        """Log HTTP response"""
        self.info(
            "HTTP Response",
            status_code=status_code,
            response_time_ms=response_time_ms,
            **kwargs,
        )

    def log_database_operation(
        self, operation: str, table: str, duration_ms: float, **kwargs
    ) -> None:
        """Log database operations"""
        self.info(
            "Database Operation",
            operation=operation,
            table=table,
            duration_ms=duration_ms,
            **kwargs,
        )

    def log_external_api_call(
        self,
        service: str,
        endpoint: str,
        status_code: int,
        duration_ms: float,
        **kwargs,
    ) -> None:
        """Log external API calls"""
        self.info(
            "External API Call",
            service=service,
            endpoint=endpoint,
            status_code=status_code,
            duration_ms=duration_ms,
            **kwargs,
        )


# Global logger instance
logger = Logger(__name__)


# Convenience functions for backward compatibility
def log_info(message: str, **kwargs) -> None:
    """Log info message - backward compatibility"""
    logger.info(message, **kwargs)


def log_error(message: str, error: Exception = None, **kwargs) -> None:
    """Log error message - backward compatibility"""
    logger.error(message, error=error, **kwargs)


def log_warning(message: str, **kwargs) -> None:
    """Log warning message - backward compatibility"""
    logger.warning(message, **kwargs)


def log_debug(message: str, **kwargs) -> None:
    """Log debug message - backward compatibility"""
    logger.debug(message, **kwargs)



================================================
FILE: src/core/postgres.py
================================================
"""PostgreSQL connection management (asyncpg pool).

Read-only registry access to the ForgeX Postgres DB (agent / workspace / user
tables). Mirrors the ``DatabaseManager`` pattern in ``core.database`` (lazy
singleton, idempotent async ``initialize()``, resilient lifespan wrappers) but
backed by an ``asyncpg`` connection pool instead of Motor.

Azure Database for PostgreSQL requires TLS; asyncpg negotiates SSL when
``ssl="require"`` is passed. Startup failures are swallowed by ``init_pg()`` so
the API still boots when Postgres is unreachable (the registry lookups then
degrade gracefully to their callers' fallbacks).
"""
import asyncio
import socket
from typing import Any, List, Optional

import asyncpg

from .config import settings
from .logging import Logger

logger = Logger("postgres")


async def _resolve_host(host: str) -> str:
    """Resolve a hostname to an IPv4 address using the OS resolver.

    asyncpg's event-loop ``getaddrinfo`` can intermittently fail on Windows
    where the blocking OS resolver succeeds. Resolving up-front in a thread and
    connecting by IP sidesteps that (TLS uses ``ssl="require"`` which does not
    verify the hostname, so an IP host is fine). Falls back to the original
    hostname on any resolver error.
    """
    try:
        loop = asyncio.get_running_loop()
        return await loop.run_in_executor(None, socket.gethostbyname, host)
    except Exception:  # noqa: BLE001
        return host


class PostgresManager:
    """Central manager for the async PostgreSQL connection pool."""

    def __init__(self):
        self.pool: Optional[asyncpg.Pool] = None
        self._initialized = False
        # The loop the pool was created on. asyncpg connections are bound to
        # their creating loop, so a caller running on a *different* loop (e.g.
        # the Azure blob backend registers documents from a throwaway loop in a
        # worker thread via ``asyncio.run``) cannot use this pool — acquiring
        # from it raises "attached to a different loop". Such callers fall back
        # to a transient standalone connection created on the current loop.
        self._loop: Optional[asyncio.AbstractEventLoop] = None
        self._resolved_host: Optional[str] = None

    async def initialize(self):
        """Create the connection pool (idempotent)."""
        if self._initialized:
            return

        pg = settings.postgres
        if not pg.dsn:
            logger.warning("PostgreSQL not configured (missing host/db/user); skipping")
            self._initialized = True
            return

        try:
            host = await _resolve_host(pg.POSTGRESQL_DATABASE_HOST)
            self._resolved_host = host
            self.pool = await asyncpg.create_pool(
                host=host,
                port=pg.POSTGRESQL_DATABASE_PORT,
                user=pg.POSTGRESQL_DATABASE_USER,
                password=pg.POSTGRESQL_DATABASE_PASSWORD,
                database=pg.POSTGRESQL_DATABASE_DATABASE,
                min_size=pg.PG_MIN_POOL_SIZE,
                max_size=pg.PG_MAX_POOL_SIZE,
                timeout=pg.PG_CONNECT_TIMEOUT,
                command_timeout=pg.PG_CONNECT_TIMEOUT,
                ssl="require",
            )
            self._loop = asyncio.get_running_loop()
            # Verify connectivity.
            async with self.pool.acquire() as conn:
                await conn.fetchval("SELECT 1")

            self._initialized = True
            logger.info(
                f"PostgreSQL pool initialized: {settings.postgres.POSTGRESQL_DATABASE_DATABASE}"
            )
        except Exception as e:
            logger.error("Failed to initialize PostgreSQL pool", error=e)
            raise

    def is_available(self) -> bool:
        return self.pool is not None

    def _on_pool_loop(self) -> bool:
        """True when the caller runs on the loop the pool belongs to.

        When False the pool is unusable for this caller and a standalone
        connection must be opened on the current loop instead.
        """
        if self.pool is None or self._loop is None:
            return False
        try:
            return asyncio.get_running_loop() is self._loop
        except RuntimeError:
            return False

    async def _connect_standalone(self) -> asyncpg.Connection:
        """Open a one-off connection on the current loop (same params as pool)."""
        pg = settings.postgres
        host = self._resolved_host or await _resolve_host(pg.POSTGRESQL_DATABASE_HOST)
        return await asyncpg.connect(
            host=host,
            port=pg.POSTGRESQL_DATABASE_PORT,
            user=pg.POSTGRESQL_DATABASE_USER,
            password=pg.POSTGRESQL_DATABASE_PASSWORD,
            database=pg.POSTGRESQL_DATABASE_DATABASE,
            timeout=pg.PG_CONNECT_TIMEOUT,
            command_timeout=pg.PG_CONNECT_TIMEOUT,
            ssl="require",
        )

    async def _run(self, op: str, query: str, *args: Any):
        """Dispatch a query via the pool, or a standalone connection off-loop.

        ``op`` is one of ``fetch`` / ``fetchrow`` / ``fetchval`` / ``execute``.
        """
        if self.pool is None:
            return None
        if self._on_pool_loop():
            async with self.pool.acquire() as conn:
                return await getattr(conn, op)(query, *args)
        # Cross-loop caller (e.g. blob backend's throwaway loop): a transient
        # connection created here is bound to the caller's loop and closes with it.
        conn = await self._connect_standalone()
        try:
            return await getattr(conn, op)(query, *args)
        finally:
            try:
                await conn.close()
            except Exception:  # noqa: BLE001
                pass

    async def fetch(self, query: str, *args: Any) -> List[asyncpg.Record]:
        """Run a query returning all rows (empty list if pool unavailable)."""
        if self.pool is None:
            return []
        return await self._run("fetch", query, *args)

    async def fetchrow(self, query: str, *args: Any) -> Optional[asyncpg.Record]:
        """Run a query returning a single row (or None)."""
        return await self._run("fetchrow", query, *args)

    async def fetchval(self, query: str, *args: Any) -> Any:
        """Run a query returning a single scalar (or None)."""
        return await self._run("fetchval", query, *args)

    async def execute(self, query: str, *args: Any) -> Optional[str]:
        """Run a write statement (INSERT/UPDATE/DDL); None if pool unavailable.

        Returns asyncpg's command status string (e.g. ``"INSERT 0 1"``).
        """
        return await self._run("execute", query, *args)

    async def close(self):
        """Close the connection pool."""
        if self.pool is not None:
            await self.pool.close()
            self.pool = None
        self._initialized = False
        logger.info("PostgreSQL pool closed")


# Global manager instance (lazy initialization).
_pg_manager: Optional[PostgresManager] = None


def get_pg_manager() -> PostgresManager:
    """Get or create the global Postgres manager (lazy)."""
    global _pg_manager
    if _pg_manager is None:
        _pg_manager = PostgresManager()
    return _pg_manager


def __getattr__(name):
    """Lazy module-level ``pg_manager`` accessor (mirrors core.database)."""
    if name == "pg_manager":
        return get_pg_manager()
    raise AttributeError(f"module '{__name__}' has no attribute '{name}'")


async def init_pg():
    """Initialize the Postgres pool (FastAPI lifespan wrapper, resilient)."""
    manager = get_pg_manager()
    try:
        await manager.initialize()
    except Exception as e:
        logger.error(f"Failed to initialize PostgreSQL: {e}")
        logger.warning("Server starting without PostgreSQL connection")


async def close_pg():
    """Close the Postgres pool (FastAPI lifespan wrapper)."""
    manager = get_pg_manager()
    await manager.close()



================================================
FILE: src/integrations/__init__.py
================================================
"""External service integrations for agent tools."""



================================================
FILE: src/integrations/github_tools.py
================================================
"""GitHub REST tools for the Dev agent.

Discovery (connection/repos/branches) plus write/push tools. Pushable/pullable
documents are resolved dynamically from whatever the agent actually saved this
session (via ``workspace_tools.resolve_pushable_document`` / ``list_output_files``)
instead of a hardcoded filename — so any artifact saved under this session's
blob prefix (``workspace/<agent_folder>/*``) can be pushed, and the agent can
list the options for the user to choose from.
"""

import base64
import re
from pathlib import Path

import httpx
from dotenv import load_dotenv
from langchain_core.tools import tool

load_dotenv()

_API = "https://api.github.com"


def _headers(config: dict | None = None) -> dict:
    from os import getenv

    config = config or {}
    token = str(config.get("github_token") or getenv("GITHUB_TOKEN", "")).strip()
    if not token:
        raise ValueError("GITHUB_TOKEN (or github_token) must be configured")
    return {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28",
    }


def _repo(repo_url: str = "", config: dict | None = None) -> tuple[str, str]:
    from os import getenv

    config = config or {}
    repo_name = config.get("github_repo_full_name")
    url = (repo_url or repo_name or getenv("GITHUB_REPO_URL", "")).strip().rstrip("/")
    if repo_name and "/" in url and "github.com" not in url:
        return tuple(url.split("/", 1))
    url = re.sub(r"\.git$", "", url)
    match = re.search(r"github\.com[/:]([^/]+)/([^/]+)$", url)
    if not match:
        raise ValueError("GITHUB_REPO_URL must be a GitHub URL such as https://github.com/org/repo")
    return match.group(1), match.group(2)


def _branch(branch: str = "", config: dict | None = None) -> str:
    from os import getenv

    config = config or {}
    return branch.strip() or str(config.get("github_branch") or getenv("GITHUB_DEFAULT_BRANCH", "main")).strip() or "main"


def _raise_detailed(response: httpx.Response) -> None:
    """Raise with GitHub's actual error body folded in, not just the HTTP status.

    ``response.raise_for_status()`` alone only reports the status code and
    discards the response body, which is where GitHub explains the real cause
    (e.g. ``{"message": "Reference already exists"}``).
    """
    if response.is_success:
        return
    detail = (response.text or "").strip()
    try:
        response.raise_for_status()
    except httpx.HTTPStatusError as exc:
        raise httpx.HTTPStatusError(
            f"{exc}: {detail}" if detail else str(exc),
            request=exc.request,
            response=exc.response,
        ) from None


def _file_sha(owner: str, repo: str, path: str, branch: str, headers: dict) -> str:
    with httpx.Client(timeout=20.0) as client:
        response = client.get(f"{_API}/repos/{owner}/{repo}/contents/{path}", headers=headers, params={"ref": branch})
    return response.json().get("sha", "") if response.status_code == 200 else ""


def _download_file(owner: str, repo: str, path: str, branch: str, headers: dict) -> bytes:
    with httpx.Client(timeout=30.0) as client:
        response = client.get(f"{_API}/repos/{owner}/{repo}/contents/{path}", headers=headers, params={"ref": branch})
    _raise_detailed(response)
    data = response.json()
    if data.get("encoding") != "base64":
        raise ValueError(f"GitHub returned unsupported content encoding for {path}")
    return base64.b64decode(data.get("content", ""))


def create_github_tools(
    workspace_dir: str,
    agent_folder: str = "dev",
    config: dict | None = None,
    workspace_id: str | None = None,
    run_location=None,
):
    """Create GitHub discovery + push/pull/branch tools for a Dev workspace."""

    @tool
    def test_github_connection() -> str:
        """Test the configured GitHub token and return the authenticated login."""
        try:
            with httpx.Client(timeout=20.0) as client:
                response = client.get(f"{_API}/user", headers=_headers(config))
            _raise_detailed(response)
            return f"Connected to GitHub as {response.json().get('login', '?')}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub connection failed: {exc}"

    @tool
    def get_github_repos() -> str:
        """List repositories accessible with the configured GitHub token."""
        try:
            with httpx.Client(timeout=20.0) as client:
                response = client.get(f"{_API}/user/repos", headers=_headers(config), params={"per_page": 30, "sort": "updated"})
            _raise_detailed(response)
            return "\n".join(
                f"{repo.get('full_name', '?')} ({repo.get('default_branch', 'main')})"
                for repo in response.json()
            ) or "No GitHub repositories found"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub repository lookup failed: {exc}"

    @tool
    def get_github_branches(repo_url: str = "") -> str:
        """List branches available in the configured GitHub repository."""
        try:
            owner, repo = _repo(repo_url, config)
            with httpx.Client(timeout=20.0) as client:
                response = client.get(f"{_API}/repos/{owner}/{repo}/branches", headers=_headers(config), params={"per_page": 100})
            _raise_detailed(response)
            return "\n".join(branch.get("name", "?") for branch in response.json()) or "No branches found"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub branch lookup failed: {exc}"

    @tool
    def list_pushable_documents() -> str:
        """List every generated document available in this session to push to GitHub or Jira.

        Call this first when the user asks to push something but doesn't name a
        specific file — show them the list and let them pick, instead of guessing
        a filename.
        """
        from services import workspace_tools

        session_id = Path(workspace_dir).name
        files = workspace_tools.list_output_files(session_id, workspace_id, run_location)
        if not files:
            return "No generated documents found in this session yet."
        return "Documents available to push:\n" + "\n".join(f"- {f}" for f in files)

    @tool
    def push_document_to_github(
        filename: str = "",
        branch: str = "",
        commit_message: str = "Add generated document",
        repo_url: str = "",
    ) -> str:
        """Push a generated document from this session to docs/<file> in GitHub.

        Args:
            filename: Which generated document to push (any artifact saved this
                session, e.g. USER_STORIES.md, TSD.md). Omit it to auto-pick when
                only one document exists; if several exist, this returns the list
                so you can show it to the user and retry with their pick.
            branch: Target branch. Defaults to the configured default branch.
                Use create_github_branch first if you want to push to a new branch.
        """
        from services import workspace_tools

        try:
            owner, repo = _repo(repo_url, config)
            target = _branch(branch, config)
            session_id = Path(workspace_dir).name
            name, text, error = workspace_tools.resolve_pushable_document(
                session_id, filename, workspace_id, run_location
            )
            if error:
                return error
            headers = _headers(config)
            remote = f"docs/{name}"
            payload = {
                "message": commit_message,
                "content": base64.b64encode(text.encode("utf-8")).decode(),
                "branch": target,
            }
            sha = _file_sha(owner, repo, remote, target, headers)
            if sha:
                payload["sha"] = sha
            with httpx.Client(timeout=30.0) as client:
                response = client.put(f"{_API}/repos/{owner}/{repo}/contents/{remote}", json=payload, headers=headers)
            _raise_detailed(response)
            return f"Pushed {name} to {owner}/{repo} on {target}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub document push failed: {exc}"

    @tool
    def pull_document_from_github(filename: str, branch: str = "", repo_url: str = "") -> str:
        """Pull docs/<filename> from GitHub into this agent's own output folder."""
        from services import workspace_tools

        try:
            owner, repo = _repo(repo_url, config)
            target = _branch(branch, config)
            headers = _headers(config)
            safe = Path(filename).name
            content = _download_file(owner, repo, f"docs/{safe}", target, headers).decode("utf-8")
            rel_path = f"workspace/{agent_folder}/{safe}"
            if run_location is not None:
                from services.workspace_tools import _backend_for_workspace, _write_via_backend

                backend = _backend_for_workspace(workspace_dir, run_location)
                error = _write_via_backend(backend, rel_path, content)
                if error:
                    return f"Failed to save pulled document: {error}"
            else:
                # DISABLED: pulled artifacts must be written through Blob/S3.
                # path = Path(workspace_dir) / "workspace" / agent_folder / safe
                # path.parent.mkdir(parents=True, exist_ok=True)
                # path.write_text(content, encoding="utf-8")
                return "Cannot save the GitHub document: cloud run location is required."
            return f"Pulled {safe} into {agent_folder}/{safe} from {owner}/{repo} on {target}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub document pull failed: {exc}"

    @tool
    def push_code_to_github(
        branch: str = "",
        commit_message: str = "Add generated code",
        repo_url: str = "",
        extensions: str = "",
    ) -> str:
        """Push every generated code file from this agent's own output folder to src/ in GitHub.

        Discovers files dynamically from wherever they were actually saved this
        session (``workspace/<agent_folder>/*`` in the active backend) — it does
        NOT require a fixed ``outputs/code/`` location.

        Args:
            extensions: Comma-separated file extensions to include (e.g. "py,ts").
                Leave empty to push every generated file that isn't a .md document.
        """
        from services import workspace_tools

        try:
            owner, repo = _repo(repo_url, config)
            target = _branch(branch, config)
            session_id = Path(workspace_dir).name
            files = workspace_tools.list_output_files(session_id, workspace_id, run_location)
            prefix = f"{agent_folder}/"
            own_files = [f for f in files if f.startswith(prefix)]
            wanted_ext = {e.strip().lstrip(".").lower() for e in extensions.split(",") if e.strip()}
            code_files = [
                f for f in own_files
                if not f.lower().endswith(".md")
                and (not wanted_ext or f.rsplit(".", 1)[-1].lower() in wanted_ext)
            ]
            if not code_files:
                available = ", ".join(own_files) if own_files else "none yet"
                return f"No generated code files found to push. Generated documents in this session: {available}"
            headers = _headers(config)
            pushed = []
            for rel in code_files:
                name = rel[len(prefix):]
                text = workspace_tools.read_output_file(session_id, name, workspace_id, run_location)
                if text is None:
                    continue
                remote = f"src/{name}"
                payload = {
                    "message": f"{commit_message}: {name}",
                    "content": base64.b64encode(text.encode("utf-8")).decode(),
                    "branch": target,
                }
                sha = _file_sha(owner, repo, remote, target, headers)
                if sha:
                    payload["sha"] = sha
                with httpx.Client(timeout=30.0) as client:
                    response = client.put(f"{_API}/repos/{owner}/{repo}/contents/{remote}", json=payload, headers=headers)
                _raise_detailed(response)
                pushed.append(name)
            return f"Pushed {len(pushed)} generated file(s) to {owner}/{repo} on {target}: {', '.join(pushed)}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub code push failed: {exc}"

    @tool
    def pull_code_from_github(branch: str = "", repo_url: str = "") -> str:
        """Pull every file under src/ from GitHub into this agent's own output folder."""
        try:
            owner, repo = _repo(repo_url, config)
            target = _branch(branch, config)
            headers = _headers(config)
            with httpx.Client(timeout=30.0) as client:
                response = client.get(f"{_API}/repos/{owner}/{repo}/git/trees/{target}", headers=headers, params={"recursive": "1"})
            _raise_detailed(response)
            entries = [
                item["path"] for item in response.json().get("tree", [])
                if item.get("type") == "blob" and item.get("path", "").startswith("src/")
            ]
            if not entries:
                return "No files found under src/"
            backend = None
            if run_location is not None:
                from services.workspace_tools import _backend_for_workspace

                backend = _backend_for_workspace(workspace_dir, run_location)
            pulled = []
            for remote in entries:
                name = remote.removeprefix("src/")
                content = _download_file(owner, repo, remote, target, headers).decode("utf-8", errors="replace")
                rel_path = f"workspace/{agent_folder}/{name}"
                if backend is not None:
                    from services.workspace_tools import _write_via_backend

                    error = _write_via_backend(backend, rel_path, content)
                    if error:
                        continue
                else:
                    # DISABLED: pulled artifacts must be written through Blob/S3.
                    # path = Path(workspace_dir) / "workspace" / agent_folder / name
                    # path.parent.mkdir(parents=True, exist_ok=True)
                    # path.write_text(content, encoding="utf-8")
                    continue
                pulled.append(name)
            return f"Pulled {len(pulled)} code file(s) into {agent_folder}/ from {owner}/{repo} on {target}: {', '.join(pulled)}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub code pull failed: {exc}"

    @tool
    def create_github_branch(branch_name: str, base_branch: str = "", repo_url: str = "") -> str:
        """Create a new branch in the configured GitHub repository.

        Args:
            branch_name: Name of the new branch to create.
            base_branch: Branch to fork from (defaults to the configured default branch).
        """
        try:
            owner, repo = _repo(repo_url, config)
            base = _branch(base_branch, config)
            headers = _headers(config)
            with httpx.Client(timeout=20.0) as client:
                base_ref = client.get(f"{_API}/repos/{owner}/{repo}/git/ref/heads/{base}", headers=headers)
                _raise_detailed(base_ref)
                base_sha = base_ref.json().get("object", {}).get("sha")
                response = client.post(
                    f"{_API}/repos/{owner}/{repo}/git/refs",
                    json={"ref": f"refs/heads/{branch_name}", "sha": base_sha},
                    headers=headers,
                )
                if response.status_code == 422 and "already exists" in response.text.lower():
                    return f"Branch {branch_name} already exists in {owner}/{repo}"
                _raise_detailed(response)
            return f"Created branch {branch_name} from {base} in {owner}/{repo}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub branch creation failed: {exc}"

    @tool
    def create_branch_and_push(
        branch_name: str,
        push_type: str = "code",
        filename: str = "",
        base_branch: str = "",
        commit_message: str = "Add generated content",
        repo_url: str = "",
    ) -> str:
        """Create a new branch (if it doesn't exist) and push generated content to it in one step.

        Args:
            branch_name: Name of the new (or existing) branch to push to.
            push_type: "code" to push every generated code file to src/ (default),
                or "document" to push a single generated document to docs/.
            filename: Only used when push_type="document" — which document to
                push. Omit to auto-pick when only one exists.
            base_branch: Branch to fork the new branch from (defaults to the
                configured default branch).
        """
        branch_result = create_github_branch.invoke(
            {"branch_name": branch_name, "base_branch": base_branch, "repo_url": repo_url}
        )
        if "failed" in branch_result.lower():
            return branch_result
        if push_type == "document":
            push_result = push_document_to_github.invoke(
                {
                    "filename": filename,
                    "branch": branch_name,
                    "commit_message": commit_message,
                    "repo_url": repo_url,
                }
            )
        else:
            push_result = push_code_to_github.invoke(
                {
                    "branch": branch_name,
                    "commit_message": commit_message,
                    "repo_url": repo_url,
                }
            )
        return f"{branch_result}\n{push_result}"

    @tool
    def create_github_pr(title: str, body: str = "", head_branch: str = "", base_branch: str = "", repo_url: str = "") -> str:
        """Create a GitHub pull request from head_branch into base_branch."""
        try:
            owner, repo = _repo(repo_url, config)
            payload = {"title": title, "body": body, "head": _branch(head_branch, config), "base": base_branch.strip() or "main"}
            with httpx.Client(timeout=20.0) as client:
                response = client.post(f"{_API}/repos/{owner}/{repo}/pulls", json=payload, headers=_headers(config))
            _raise_detailed(response)
            data = response.json()
            return f"Created PR #{data.get('number', '?')}: {data.get('html_url', '?')}"
        except Exception as exc:  # noqa: BLE001
            return f"GitHub PR creation failed: {exc}"

    return [
        test_github_connection,
        get_github_repos,
        get_github_branches,
        list_pushable_documents,
        push_document_to_github,
        pull_document_from_github,
        push_code_to_github,
        pull_code_from_github,
        create_github_branch,
        create_branch_and_push,
        create_github_pr,
    ]



================================================
FILE: src/integrations/jira_tools.py
================================================
"""Jira REST tools shared by the PO, Dev, and QE agents."""

import base64
import re
from pathlib import Path

import httpx
from dotenv import load_dotenv
from langchain_core.tools import tool

load_dotenv()


def _headers(config: dict | None = None) -> dict:
    from os import getenv

    config = config or {}
    email = str(config.get("jira_email") or config.get("email") or getenv("JIRA_EMAIL", "")).strip()
    token = str(config.get("jira_access_token") or config.get("jira_api_token") or getenv("JIRA_API_TOKEN", "")).strip()
    if not email or not token:
        raise ValueError("JIRA_EMAIL (or jira_email) and JIRA_API_TOKEN (or jira_access_token) must be configured")
    encoded = base64.b64encode(f"{email}:{token}".encode()).decode()
    return {
        "Authorization": f"Basic {encoded}",
        "Accept": "application/json",
        "Content-Type": "application/json",
    }


def _base_url(config: dict | None = None) -> str:
    from os import getenv

    config = config or {}
    url = str(config.get("jira_url") or config.get("project_url") or getenv("JIRA_PROJECT_URL", "")).strip().rstrip("/")
    if not url:
        raise ValueError("JIRA_PROJECT_URL (or jira_url) must be configured")
    return url


def _raise_detailed(response: httpx.Response) -> None:
    """Like response.raise_for_status(), but folds Jira's error body into the
    exception message — the raw response body (e.g. field-not-on-screen
    errors) is what actually explains a 400, and raise_for_status() alone
    only reports the status code."""
    try:
        response.raise_for_status()
    except httpx.HTTPStatusError as exc:
        detail = response.text.strip()
        raise httpx.HTTPStatusError(f"{exc}: {detail}", request=exc.request, response=exc.response) from None


def _resolve_issue_type(client: httpx.Client, project_key: str, requested: str, config: dict | None) -> str:
    """Pick an issue type name this project actually accepts.

    Team-managed (Kanban-style) projects often don't have a "Story" type at
    all — only e.g. Task/Bug/Epic — so hardcoding "Story" gets a 400 ("Specify
    a valid issue type"). Looks up the project's real issue types and falls
    back to the closest match instead of failing outright.
    """
    try:
        response = client.get(f"{_base_url(config)}/rest/api/3/project/{project_key}", headers=_headers(config))
        response.raise_for_status()
        issue_types = [
            t.get("name", "") for t in response.json().get("issueTypes", []) if not t.get("subtask")
        ]
    except Exception:  # noqa: BLE001
        return requested
    if not issue_types:
        return requested
    names_lower = {n.lower(): n for n in issue_types}
    if requested.lower() in names_lower:
        return names_lower[requested.lower()]
    for fallback in ("story", "task"):
        if fallback in names_lower:
            return names_lower[fallback]
    return issue_types[0]


def _to_adf(text: str) -> dict:
    content = [
        {"type": "paragraph", "content": [{"type": "text", "text": line.strip()}]}
        for line in text.strip().splitlines()
        if line.strip()
    ]
    return {
        "version": 1,
        "type": "doc",
        "content": content or [{"type": "paragraph", "content": [{"type": "text", "text": text}]}],
    }


def _parse_stories(content: str) -> list[dict]:
    stories = []
    for block in re.split(r"\n(?=#{2,3}\s+)", content):
        match = re.match(r"^#{2,3}\s+(?:US-\d+\s*[:\-]?\s*)?(.+)", block.strip())
        if not match:
            continue
        summary = match.group(1).strip()
        if summary.lower() in {"user stories", "epics", "overview", "summary"}:
            continue
        description = "\n".join(block.strip().splitlines()[1:]).strip()
        stories.append({"summary": summary[:255], "description": description[:3000]})
    return stories


def create_jira_tools(
    workspace_dir: str,
    agent_folder: str = "dev",
    config: dict | None = None,
    workspace_id: str | None = None,
    run_location=None,
):
    """Create Jira connection and story-management tools for a workspace."""

    backend = None
    if run_location is not None:
        from services.workspace_tools import _backend_for_workspace

        backend = _backend_for_workspace(workspace_dir, run_location)

    @tool
    def test_jira_connection() -> str:
        """Test Jira credentials and return the authenticated account name."""
        try:
            with httpx.Client(timeout=20.0) as client:
                response = client.get(f"{_base_url(config)}/rest/api/3/myself", headers=_headers(config))
            response.raise_for_status()
            data = response.json()
            return f"Connected to Jira as {data.get('displayName') or data.get('emailAddress', '?')}"
        except Exception as exc:  # noqa: BLE001
            return f"Jira connection failed: {exc}"

    @tool
    def get_jira_projects() -> str:
        """List Jira projects accessible with the configured credentials."""
        try:
            with httpx.Client(timeout=20.0) as client:
                response = client.get(f"{_base_url(config)}/rest/api/3/project", headers=_headers(config))
            response.raise_for_status()
            projects = response.json()
            return "\n".join(f"{p.get('key', '?')}: {p.get('name', '?')}" for p in projects) or "No Jira projects found"
        except Exception as exc:  # noqa: BLE001
            return f"Jira project lookup failed: {exc}"

    @tool
    def get_jira_issues(project_key: str = "", jql: str = "", max_results: int = 50) -> str:
        """List Jira issues from a project or custom JQL query."""
        from os import getenv

        key = project_key.strip() or str((config or {}).get("jira_project_key") or getenv("JIRA_PROJECT_KEY", "")).strip()
        query = jql.strip() or (f'project = "{key}" ORDER BY created DESC' if key else "ORDER BY created DESC")
        try:
            with httpx.Client(timeout=30.0) as client:
                response = client.get(
                    # /rest/api/3/search was removed by Atlassian (returns 410
                    # Gone); /rest/api/3/search/jql is the current replacement.
                    f"{_base_url(config)}/rest/api/3/search/jql",
                    headers=_headers(config),
                    params={"jql": query, "maxResults": min(max(max_results, 1), 100), "fields": "summary,description,status,priority"},
                )
            _raise_detailed(response)
            issues = response.json().get("issues", [])
            if not issues:
                return "No Jira issues found"
            return "\n".join(
                f"{issue.get('key', '?')}: {issue.get('fields', {}).get('summary', '?')}"
                for issue in issues
            )
        except Exception as exc:  # noqa: BLE001
            return f"Jira issue lookup failed: {exc}"

    @tool
    def create_jira_story(summary: str, description: str = "", project_key: str = "", priority: str = "Medium", issue_type: str = "Story") -> str:
        """Create one Jira issue (Story by default) using the supplied or configured project key."""
        from os import getenv

        key = project_key.strip() or str((config or {}).get("jira_project_key") or getenv("JIRA_PROJECT_KEY", "")).strip()
        if not key:
            return "project_key is required or JIRA_PROJECT_KEY must be configured"
        try:
            with httpx.Client(timeout=30.0) as client:
                resolved_type = _resolve_issue_type(client, key, issue_type or "Story", config)
                payload = {
                    "fields": {
                        "project": {"key": key},
                        "summary": summary[:255],
                        "issuetype": {"name": resolved_type},
                        "priority": {"name": priority},
                    }
                }
                if description:
                    payload["fields"]["description"] = _to_adf(description)
                response = client.post(f"{_base_url(config)}/rest/api/3/issue", json=payload, headers=_headers(config))
                if response.status_code == 400 and "priority" in payload["fields"]:
                    # Common on team-managed/Kanban projects: Priority isn't on
                    # the issue's create screen, so Jira rejects the field with
                    # a 400. Retry once without it rather than failing outright.
                    payload["fields"].pop("priority", None)
                    response = client.post(f"{_base_url(config)}/rest/api/3/issue", json=payload, headers=_headers(config))
            _raise_detailed(response)
            issue_key = response.json().get("key", "?")
            return f"Created Jira issue {issue_key}: {_base_url(config)}/browse/{issue_key}"
        except Exception as exc:  # noqa: BLE001
            return f"Jira story creation failed: {exc}"

    @tool
    def push_user_stories_to_jira(project_key: str = "", filename: str = "") -> str:
        """Create Jira Stories from a generated document in this session.

        Args:
            project_key: Jira project key (falls back to the configured default).
            filename: Which generated document to push (e.g. USER_STORIES.md).
                Omit it to auto-pick when only one document exists; if several
                exist, this returns the list so you can ask the user to choose
                and retry with their pick.
        """
        from os import getenv
        from services import workspace_tools

        key = project_key.strip() or str((config or {}).get("jira_project_key") or getenv("JIRA_PROJECT_KEY", "")).strip()
        if not key:
            return "project_key is required or JIRA_PROJECT_KEY must be configured"
        session_id = Path(workspace_dir).name
        name, text, error = workspace_tools.resolve_pushable_document(
            session_id, filename, workspace_id, run_location
        )
        if error:
            return error
        stories = _parse_stories(text)
        if not stories:
            return f"No user stories could be parsed from {name}"
        created = []
        for story in stories:
            result = create_jira_story.invoke({**story, "project_key": key})
            created.append(result)
        return f"Processed {len(created)} Jira stories from {name} for project {key}:\n" + "\n".join(created)

    @tool
    def pull_jira_stories_to_workspace(project_key: str = "", max_results: int = 50) -> str:
        """Pull Jira issues into USER_STORIES.md in this agent's own output folder."""
        from os import getenv

        key = project_key.strip() or str((config or {}).get("jira_project_key") or getenv("JIRA_PROJECT_KEY", "")).strip()
        query = f'project = "{key}" ORDER BY created ASC' if key else "ORDER BY created ASC"
        try:
            with httpx.Client(timeout=30.0) as client:
                response = client.get(
                    f"{_base_url(config)}/rest/api/3/search/jql",
                    headers=_headers(config),
                    params={"jql": query, "maxResults": min(max(max_results, 1), 100), "fields": "summary,description,status,priority"},
                )
            _raise_detailed(response)
            issues = response.json().get("issues", [])
            if not issues:
                return "No Jira issues found"
            output = ["# User Stories", ""]
            for issue in issues:
                fields = issue.get("fields", {})
                output.extend([f"## {issue.get('key', '?')}: {fields.get('summary', '')}", "", f"Status: {fields.get('status', {}).get('name', 'Unknown')}", "", "---", ""])
            text = "\n".join(output)
            rel_path = f"workspace/{agent_folder}/USER_STORIES.md"
            if backend is not None:
                from services.workspace_tools import _write_via_backend

                error = _write_via_backend(backend, rel_path, text)
                if error:
                    return f"Failed to save pulled stories: {error}"
            else:
                # DISABLED: pulled artifacts must be written through Blob/S3.
                # path = Path(workspace_dir) / "workspace" / agent_folder / "USER_STORIES.md"
                # path.parent.mkdir(parents=True, exist_ok=True)
                # path.write_text(text, encoding="utf-8")
                return "Cannot save Jira stories: cloud run location is required."
            return f"Pulled {len(issues)} Jira issue(s) into {agent_folder}/USER_STORIES.md"
        except Exception as exc:  # noqa: BLE001
            return f"Jira story pull failed: {exc}"

    return [test_jira_connection, get_jira_projects, get_jira_issues, create_jira_story, push_user_stories_to_jira, pull_jira_stories_to_workspace]



================================================
FILE: src/services/__init__.py
================================================
"""
Dev domain package.

The Dev-specific business logic, split out of ``core`` so it sits apart
from both the platform infrastructure (``core``) and the agent runtime
(``agent``). Distinct from the vendored ``dev`` package, which holds
verbatim copies of the MCP server's helper modules.

- ``dev_service``  — ``send_message`` / ``update_design_hld`` orchestration
- ``dev_jobs``     — Mongo-backed async job store for those flows
- ``design_snapshot``    — async per-conversation design snapshot store
- ``mermaid``            — shared Mermaid rendering pipeline
- ``session_managers``   — chat history / session context managers
- ``mongo_models``       — MongoDB document schemas (``ChatMessage``, ``SessionContext``)
"""



================================================
FILE: src/services/agent_memory.py
================================================
"""Durable long-term agent memory (Motor-backed).

Persists free-form notes the agent chooses to remember across conversations,
keyed by ``(workspace_id, user_id, conversation_id, key)``. This is distinct
from:
  * chat history (``session_managers``) — the raw turn-by-turn transcript, and
  * design snapshots (``design_snapshot``) — the last generated design.

Collection name comes from ``DEV_AGENT_MEMORY_COLLECTION``
(default ``dev_agent_memory``). Use ``conversation_id=""`` for memory that
should be shared across all of a user's conversations.
"""
from datetime import datetime
from typing import Any, Dict, List

from core.config import settings
from core.database import db_manager
from core.logging import logger


def _collection():
    return db_manager.get_mongo_collection(
        settings.database.DEV_AGENT_MEMORY_COLLECTION
    )


async def remember(
    workspace_id: str,
    user_id: str,
    conversation_id: str,
    key: str,
    content: str,
) -> None:
    """Upsert a single memory entry (best-effort; memory is non-critical)."""
    try:
        await db_manager.initialize()
        await _collection().update_one(
            {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "conversation_id": conversation_id,
                "key": key,
            },
            {"$set": {"content": content, "updated_at": datetime.utcnow()}},
            upsert=True,
        )
    except Exception as e:  # noqa: BLE001
        logger.error("Failed to save agent memory", error=e)


async def recall(
    workspace_id: str,
    user_id: str,
    conversation_id: str,
) -> List[Dict[str, Any]]:
    """Return memory entries for this conversation scope (newest first)."""
    try:
        await db_manager.initialize()
        cursor = (
            _collection()
            .find(
                {
                    "workspace_id": workspace_id,
                    "user_id": user_id,
                    "conversation_id": conversation_id,
                },
                {"_id": 0},
            )
            .sort("updated_at", -1)
        )
        return [doc async for doc in cursor]
    except Exception as e:  # noqa: BLE001
        logger.error("Failed to recall agent memory", error=e)
        return []



================================================
FILE: src/services/design_snapshot.py
================================================
"""
Async design-snapshot store (REST-native, Motor-backed).

Mirrors the MCP server's ``DesignSnapshotStore`` (which is sync pymongo) but uses
REST's shared async ``db_manager`` so the ported ``send_message`` flow keeps the
last generated/updated design per conversation for resilience across restarts.

Document key: ``(workspace_id, user_id, conversation_id)``.
Fields: ``last_hld_struct`` (dict), ``last_hld_plaintext`` (str), ``updated_at``.
Collection name comes from ``DEV_DESIGN_COLLECTION``
(default ``dev_design_snapshots``) to match the MCP server.
"""
from datetime import datetime
from typing import Any, Dict, Optional

from core.config import settings
from core.database import db_manager
from core.logging import logger


def _collection():
    return db_manager.get_mongo_collection(
        settings.workflow.DEV_DESIGN_COLLECTION
    )


async def upsert_snapshot(
    workspace_id: str,
    user_id: str,
    conversation_id: str,
    last_hld_struct: Optional[Dict[str, Any]],
    last_hld_plaintext: Optional[str],
) -> None:
    """Best-effort upsert of the latest design snapshot for a conversation."""
    try:
        await db_manager.initialize()
        await _collection().update_one(
            {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "conversation_id": conversation_id,
            },
            {
                "$set": {
                    "last_hld_struct": last_hld_struct,
                    "last_hld_plaintext": last_hld_plaintext,
                    "updated_at": datetime.utcnow(),
                }
            },
            upsert=True,
        )
    except Exception as e:  # noqa: BLE001 - snapshots are non-critical
        logger.error("Failed to upsert design snapshot", error=e)


async def get_snapshot(
    workspace_id: str, user_id: str, conversation_id: str
) -> Optional[Dict[str, Any]]:
    """Return the stored snapshot (without ``_id``) or ``None``."""
    try:
        await db_manager.initialize()
        return await _collection().find_one(
            {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "conversation_id": conversation_id,
            },
            {"_id": 0},
        )
    except Exception as e:  # noqa: BLE001
        logger.error("Failed to fetch design snapshot", error=e)
        return None



================================================
FILE: src/services/dev_jobs.py
================================================
"""
Mongo-backed job store for the asynchronous ``send_message`` flow.

Azure Functions invocations are short-lived and may run on different instances,
so job state must live in durable storage rather than process memory. The POST
handler creates a ``queued`` job and enqueues a Storage Queue message; the
queue-triggered worker flips it to ``running`` then ``completed``/``failed``; the
status endpoint reads it back. All access is async (Motor) via the shared
``db_manager``.
"""
from datetime import datetime
from typing import Any, Dict, Optional

from core.database import db_manager
from core.logging import logger

# Job lifecycle states.
STATUS_QUEUED = "queued"
STATUS_RUNNING = "running"
STATUS_COMPLETED = "completed"
STATUS_FAILED = "failed"
STATUS_CANCELLED = "cancelled"

_COLLECTION_NAME = "dev_jobs"


def _collection():
    return db_manager.get_mongo_collection(_COLLECTION_NAME)


async def create_job(job_id: str, request: Dict[str, Any]) -> Dict[str, Any]:
    """Insert a new ``queued`` job document for ``job_id``."""
    await db_manager.initialize()
    now = datetime.utcnow()
    doc = {
        "job_id": job_id,
        "status": STATUS_QUEUED,
        "request": request,
        "result": None,
        "error": None,
        "created_at": now,
        "updated_at": now,
    }
    await _collection().insert_one(dict(doc))
    return doc


async def mark_running(job_id: str) -> None:
    """Transition a job to ``running``."""
    await db_manager.initialize()
    await _collection().update_one(
        {"job_id": job_id},
        {"$set": {"status": STATUS_RUNNING, "updated_at": datetime.utcnow()}},
    )


async def complete_job(job_id: str, result: Dict[str, Any]) -> None:
    """Store the tool result and mark the job ``completed``."""
    await db_manager.initialize()
    await _collection().update_one(
        {"job_id": job_id},
        {
            "$set": {
                "status": STATUS_COMPLETED,
                "result": result,
                "updated_at": datetime.utcnow(),
            }
        },
    )


async def fail_job(
    job_id: str, error: str, result: Optional[Dict[str, Any]] = None
) -> None:
    """Record an error and mark the job ``failed`` (optionally keep a result)."""
    await db_manager.initialize()
    update: Dict[str, Any] = {
        "status": STATUS_FAILED,
        "error": error,
        "updated_at": datetime.utcnow(),
    }
    if result is not None:
        update["result"] = result
    await _collection().update_one({"job_id": job_id}, {"$set": update})
    logger.error("send_message job failed", job_id=job_id, error_detail=error)


async def cancel_job(job_id: str) -> bool:
    """Best-effort cancellation request.

    Marks a job as ``cancelled`` unless it already finished.

    Returns:
        True if the status was transitioned to cancelled, False if the job was
        already in a terminal state (completed/failed/cancelled) or missing.
    """
    await db_manager.initialize()
    existing = await _collection().find_one({"job_id": job_id}, {"_id": 0, "status": 1})
    if not existing:
        return False

    status = existing.get("status")
    if status in {STATUS_COMPLETED, STATUS_FAILED, STATUS_CANCELLED}:
        return False

    await _collection().update_one(
        {"job_id": job_id},
        {
            "$set": {
                "status": STATUS_CANCELLED,
                "error": "cancelled",
                "updated_at": datetime.utcnow(),
            }
        },
    )
    return True


async def is_cancelled(job_id: str) -> bool:
    """Return True if the job is currently marked as cancelled."""
    await db_manager.initialize()
    doc = await _collection().find_one({"job_id": job_id}, {"_id": 0, "status": 1})
    return bool(doc and doc.get("status") == STATUS_CANCELLED)


async def cancel_jobs_by_conversation(conversation_id: str) -> int:
    """Cancel all non-terminal jobs for a conversation.

    Returns:
        Number of jobs updated to ``cancelled``.
    """
    await db_manager.initialize()
    result = await _collection().update_many(
        {
            "request.conversation_id": conversation_id,
            "status": {"$nin": [STATUS_COMPLETED, STATUS_FAILED, STATUS_CANCELLED]},
        },
        {
            "$set": {
                "status": STATUS_CANCELLED,
                "error": "cancelled",
                "updated_at": datetime.utcnow(),
            }
        },
    )
    return int(result.modified_count or 0)


async def get_job(job_id: str) -> Optional[Dict[str, Any]]:
    """Return the job document (without Mongo ``_id``) or ``None``."""
    await db_manager.initialize()
    doc = await _collection().find_one({"job_id": job_id}, {"_id": 0})
    return doc


# Referenced by core.database._mongo_collection_indexes() so the unique index on
# job_id is created alongside the existing collections on startup.
JOB_COLLECTION_NAME = _COLLECTION_NAME



================================================
FILE: src/services/documents.py
================================================
"""Session document registry (``fe_agent_document``).

Records metadata for documents generated during an agent session so the UI can
list them per session. Backed by the ForgeX Postgres DB via ``core.postgres``.

Follows the best-effort, parameterized pattern in ``services.registry``: the
table name comes from ``settings.postgres.PG_DOCUMENT_TABLE`` (trusted config,
not user input); all row values are parameterized (``$1``). Every function is
best-effort — on any error (or when Postgres is unavailable) writes return
``False`` and reads return ``[]`` so callers degrade gracefully.
"""
from typing import Any, Dict, List, Optional

import asyncpg

from core.config import settings
from core.postgres import get_pg_manager
from core.logging import logger


def _pg():
    return get_pg_manager()


_SELECT_COLS = (
    "id, session_id, document_name, document_url, created_at, created_by, "
    "workspace_id, workflow_job_id, scope, agent_folder"
)

# Legacy column set for DBs where the run-scope migration
# (2026_08_27_fe_agent_document_run_scope.sql) has not been applied yet. Used as
# an automatic fallback so document registration/listing keeps working before the
# migration runs (the new columns simply aren't populated/returned).
_SELECT_COLS_LEGACY = "id, session_id, document_name, document_url, created_at, created_by"

# asyncpg raises this when a query references a column the table doesn't have.
_MISSING_COL_ERRORS = (asyncpg.UndefinedColumnError,)


async def upsert_document(
    session_id: str,
    document_name: str,
    document_url: str,
    created_by: str = "System",
    workspace_id: Optional[str] = None,
    workflow_job_id: Optional[str] = None,
    scope: Optional[str] = None,
    agent_folder: Optional[str] = None,
) -> bool:
    """Insert or update a document row for a session (idempotent).

    Relies on the UNIQUE (session_id, document_name) constraint for the upsert.
    The run-restructure columns (``workspace_id``/``workflow_job_id``/``scope``/
    ``agent_folder``) let the UI group shared workflow outputs and label each
    document by the agent that produced it. Returns True on success, False on any
    error / when Postgres is unavailable.
    """
    t = settings.postgres.PG_DOCUMENT_TABLE
    query = f'''
        INSERT INTO "{t}" (
            session_id, document_name, document_url, created_by, created_at,
            workspace_id, workflow_job_id, scope, agent_folder
        )
        VALUES ($1, $2, $3, $4, now(), $5, $6, $7, $8)
        ON CONFLICT (session_id, document_name)
        DO UPDATE SET
            document_url = EXCLUDED.document_url,
            created_at = now(),
            workspace_id = EXCLUDED.workspace_id,
            workflow_job_id = EXCLUDED.workflow_job_id,
            scope = EXCLUDED.scope,
            agent_folder = EXCLUDED.agent_folder
    '''
    legacy_query = f'''
        INSERT INTO "{t}" (
            session_id, document_name, document_url, created_by, created_at
        )
        VALUES ($1, $2, $3, $4, now())
        ON CONFLICT (session_id, document_name)
        DO UPDATE SET
            document_url = EXCLUDED.document_url,
            created_at = now()
    '''
    try:
        result = await _pg().execute(
            query, session_id, document_name, document_url, created_by,
            workspace_id, workflow_job_id, scope, agent_folder,
        )
        return result is not None
    except _MISSING_COL_ERRORS:
        # Run-scope migration not applied yet — register with the legacy columns
        # so the document still appears in the UI Output panel.
        logger.warning(
            "documents.upsert_document: run-scope columns missing; "
            "falling back to legacy insert (run migration "
            "2026_08_27_fe_agent_document_run_scope.sql)"
        )
        try:
            result = await _pg().execute(
                legacy_query, session_id, document_name, document_url, created_by,
            )
            return result is not None
        except Exception as e:  # noqa: BLE001
            logger.error("documents.upsert_document legacy fallback failed", error=e)
            return False
    except Exception as e:  # noqa: BLE001
        logger.error("documents.upsert_document failed", error=e)
        return False


async def _fetch_docs(
    where: str,
    order: str,
    *args: Any,
    label: str,
    legacy_fallback: Optional[Dict[str, Any]] = None,
) -> List[Dict[str, Any]]:
    """Fetch document rows with an automatic legacy-column fallback.

    Tries the full run-scope column set first; if those columns don't exist yet
    (migration not applied), retries with the legacy columns so the UI still gets
    its documents.

    Fallback selection when the run-scope columns are missing:
    - If ``legacy_fallback`` is given, it explicitly describes the pre-migration
      retry: ``{"where": ..., "order": ..., "args": (...)}``. This lets callers
      whose *primary* ``where`` names a run-scope column (e.g. filtering on
      ``workflow_job_id``) still resolve via an equivalent legacy query — dynamic
      docs are stored with ``session_id = workflow_job_id``, so a
      ``WHERE session_id = $1`` legacy query returns them.
    - Otherwise the primary ``where``/``order`` are reused, but only when they
      reference no run-scope column (a plain session query); else there is nothing
      to return on a pre-migration DB.
    """
    t = settings.postgres.PG_DOCUMENT_TABLE
    query = f'SELECT {_SELECT_COLS} FROM "{t}" {where} {order}'
    try:
        rows = await _pg().fetch(query, *args)
        return [dict(r) for r in rows]
    except _MISSING_COL_ERRORS:
        logger.warning(
            f"documents.{label}: run-scope columns missing; falling back to "
            "legacy columns (run migration 2026_08_27_fe_agent_document_run_scope.sql)"
        )
        if legacy_fallback is not None:
            legacy_where = legacy_fallback["where"]
            legacy_order = legacy_fallback.get("order", order)
            legacy_args = legacy_fallback.get("args", args)
        elif "workflow_job_id" in where or "agent_folder" in where or "scope" in where:
            # Primary query filters on a missing column and no explicit legacy form
            # was supplied — nothing resolvable on a pre-migration DB.
            return []
        else:
            legacy_where, legacy_order, legacy_args = where, order, args
        legacy = f'SELECT {_SELECT_COLS_LEGACY} FROM "{t}" {legacy_where} {legacy_order}'
        try:
            rows = await _pg().fetch(legacy, *legacy_args)
            return [dict(r) for r in rows]
        except Exception as e:  # noqa: BLE001
            logger.error(f"documents.{label} legacy fallback failed", error=e)
            return []
    except Exception as e:  # noqa: BLE001
        logger.error(f"documents.{label} failed", error=e)
        return []


async def list_session_documents(session_id: str) -> List[Dict[str, Any]]:
    """Return all documents recorded for a session, oldest first."""
    return await _fetch_docs(
        "WHERE session_id = $1", "ORDER BY created_at", session_id,
        label="list_session_documents",
    )


async def list_workflow_documents(workflow_job_id: str) -> List[Dict[str, Any]]:
    """All documents produced across every agent in a shared dynamic workflow.

    Uses the run-scope ``workflow_job_id`` column when present. On a pre-migration
    DB it falls back to ``WHERE session_id = $1`` with the same id — dynamic docs
    are registered with ``session_id = workflow_job_id`` — so the workflow Output
    panel still lists documents before the migration is applied.
    """
    return await _fetch_docs(
        "WHERE workflow_job_id = $1", "ORDER BY agent_folder, created_at",
        workflow_job_id, label="list_workflow_documents",
        legacy_fallback={
            "where": "WHERE session_id = $1",
            "order": "ORDER BY created_at",
            "args": (workflow_job_id,),
        },
    )


async def list_agent_documents(
    workspace_id: str, agent_folder: str
) -> List[Dict[str, Any]]:
    """All documents an individual agent produced within a workspace.

    Requires the run-scope columns; on a pre-migration DB this returns [] (the
    columns it filters by don't exist).
    """
    return await _fetch_docs(
        "WHERE workspace_id = $1 AND agent_folder = $2 AND scope = 'individual'",
        "ORDER BY created_at", workspace_id, agent_folder,
        label="list_agent_documents",
    )



================================================
FILE: src/services/intake_parsing.py
================================================
"""Server-side text extraction for uploaded intake documents.

The browser cannot reliably read binary Office/PDF formats, so intake documents
are uploaded as raw bytes and parsed here. Plain-text formats are decoded
directly; ``.docx`` uses ``python-docx`` (already a dependency, mirroring
``workflow_context.py``); ``.pdf`` uses ``pypdf``.
"""
from __future__ import annotations

import io
from pathlib import Path
from typing import Tuple

# Extensions we can decode as UTF-8 text directly.
TEXT_EXTENSIONS = {
    "txt", "md", "markdown", "csv", "tsv", "json", "log",
    "yaml", "yml", "xml", "html", "htm", "ini", "conf", "rtf",
}


class UnsupportedIntakeType(Exception):
    """Raised when a file's type cannot be parsed into text."""


def _extension(filename: str) -> str:
    suffix = Path(filename or "").suffix
    return suffix[1:].lower() if suffix else ""


def _extract_docx(data: bytes) -> str:
    from docx import Document  # python-docx

    doc = Document(io.BytesIO(data))
    parts = []
    for paragraph in doc.paragraphs:
        if paragraph.text.strip():
            parts.append(paragraph.text.strip())
    for table in doc.tables:
        for row in table.rows:
            for cell in row.cells:
                if cell.text.strip():
                    parts.append(cell.text.strip())
    return "\n".join(parts)


def _extract_pdf(data: bytes) -> str:
    from pypdf import PdfReader

    reader = PdfReader(io.BytesIO(data))
    parts = []
    for page in reader.pages:
        text = (page.extract_text() or "").strip()
        if text:
            parts.append(text)
    return "\n\n".join(parts)


def extract_intake_text(filename: str, data: bytes) -> Tuple[str, str]:
    """Extract text from an uploaded intake file, normalized to Markdown.

    Returns ``(output_name, text)``. Every intake document is stored under a
    ``.md`` name so the agent's ``read_input`` always decodes clean, readable
    Markdown text — never raw binary. Binary formats (``.docx``/``.pdf``) are
    parsed to text; already-``.md`` files keep their name; other plain-text
    formats are re-labelled ``.md`` (their content is already text).
    Raises :class:`UnsupportedIntakeType` for formats we can't parse.
    """
    ext = _extension(filename)
    stem = Path(filename or "document").name
    md_name = f"{Path(stem).stem}.md"

    if ext in ("md", "markdown"):
        return md_name, data.decode("utf-8", errors="replace")

    if ext in TEXT_EXTENSIONS or ext == "":
        return md_name, data.decode("utf-8", errors="replace")

    if ext == "docx":
        return md_name, _extract_docx(data)

    if ext == "pdf":
        return md_name, _extract_pdf(data)

    raise UnsupportedIntakeType(
        f"Unsupported intake file type: .{ext} ({stem}). "
        f"Supported: {', '.join(sorted(TEXT_EXTENSIONS))}, docx, pdf."
    )



================================================
FILE: src/services/kb_client.py
================================================
"""LightRAG / KBCurator knowledge-base client (async, httpx).

Thin I/O layer over the KBCurator ``/api/query-rag`` endpoint. The Dev agent's
``kb_search`` tool calls :func:`lightrag_query`; all prompt shaping lives in the
skill markdown, not here.

Endpoint resolution mirrors the legacy wrapper:
  1. ``KBCURATOR_QUERY_ENDPOINT`` (explicit), else
  2. derived from ``KBCURATOR_URL`` by trimming a trailing transport segment and
     appending ``/api/query-rag``.
"""
from __future__ import annotations

import json
from urllib.parse import urlparse

import httpx

from core.config import settings
from core.logging import logger


def _derive_kb_http_endpoint(base_url: str) -> str:
    """Resolve KB query endpoint from configuration."""
    explicit = (settings.knowledge_base.KBCURATOR_QUERY_ENDPOINT or "").strip()
    if explicit:
        # If explicit endpoint is absolute (starts with http/https), use as-is
        if explicit.startswith(("http://", "https://")):
            return explicit
        # If it's a relative path, join with base URL
        if base_url:
            base_clean = base_url.rstrip("/")
            endpoint_clean = explicit.lstrip("/")
            return f"{base_clean}/{endpoint_clean}"
        # No base URL and relative path → cannot resolve
        return ""

    if not base_url:
        return ""

    clean = base_url.rstrip("/")
    parsed = urlparse(clean)
    if not parsed.scheme or not parsed.netloc:
        return ""

    path = parsed.path.strip("/")
    if path and "/" not in path and not path.startswith("api"):
        clean = f"{parsed.scheme}://{parsed.netloc}"

    return f"{clean}/api/v2/query-kb"


def _extract_rag_context(resp_json: dict) -> str:
    """Best-effort normalization for different KB response envelopes."""
    if not isinstance(resp_json, dict):
        return "No relevant context found."

    # Common envelopes
    if isinstance(resp_json.get("LightRAG"), str):
        content = resp_json.get("LightRAG", "")
        refs = resp_json.get("sources")
        if refs is not None:
            return f"{content}\n{json.dumps(refs)}"
        return content or "No relevant context found."

    data = resp_json.get("data")
    if isinstance(data, dict):
        if isinstance(data.get("LightRAG"), str):
            content = data.get("LightRAG", "")
            refs = data.get("sources")
            if refs is not None:
                return f"{content}\n{json.dumps(refs)}"
            return content or "No relevant context found."
        if isinstance(data.get("content"), str):
            return data.get("content") or "No relevant context found."

    if isinstance(resp_json.get("content"), str):
        return resp_json.get("content") or "No relevant context found."

    return "No relevant context found."


async def lightrag_query(
    prompt: str,
    namespace: dict | None = None,
    user_prompt: str = "",
    history: list | None = None,
    auth_header: str = "",
) -> str:
    """Query the LightRAG knowledge base over HTTP.

    Args:
        prompt: The user's question / retrieval query.
        namespace: KB namespace selector (merged into the request body).
        user_prompt: Optional extra instruction passed through to the KB.
        history: Prior turns to give the KB conversational context.
        auth_header: Optional bearer/authorization header to forward.

    Returns:
        Context string from the knowledge base, or a fallback message on errors.
    """
    endpoint = _derive_kb_http_endpoint(settings.knowledge_base.KBCURATOR_URL)
    if not endpoint:
        logger.warning("KBCURATOR URL/endpoint is not configured")
        return "No relevant context found."

    payload = {"query": prompt, "history": history or [], **(namespace or {})}

    if user_prompt:
        payload["user_prompt"] = user_prompt

    headers = {"Content-Type": "application/json"}
    if auth_header:
        headers["Authorization"] = auth_header

    timeout = float(settings.knowledge_base.KB_QUERY_TIMEOUT or 30)
    logger.info(f"[KB_Client] Auth Header: {auth_header}")
    logger.info(f"[KB_Client] Payload: {payload}")
    logger.info(f"[KB_Client] Endpoint: {endpoint}")
    try:
        async with httpx.AsyncClient(timeout=timeout) as client:
            response = await client.post(endpoint, json=payload, headers=headers)
            logger.info(f"[KB Client] Response: {response}")
            response.raise_for_status()
            resp_json = response.json()
        context = _extract_rag_context(resp_json)
        if "[no-context]" in context:
            return "No relevant context found."
        return context
    except httpx.TimeoutException:
        logger.warning("LightRAG HTTP query timed out")
        return "Knowledge Base service is unreachable. Please try again later."
    except httpx.HTTPStatusError as e:
        # Log the response body for 400 errors to see what's wrong
        logger.error(
            "KB endpoint returned error",
            status=e.response.status_code,
            response_body=e.response.text[:500],  # First 500 chars
            sent_payload=payload,
        )
        return "No relevant context found."
    except Exception as e:  # noqa: BLE001
        logger.error("Error querying LightRAG KB over HTTP", error=str(e))
        return "No relevant context found."



================================================
FILE: src/services/mermaid.py
================================================
"""
Shared Mermaid rendering pipeline.

Extracted from the ``render_mermaid`` HTTP function so both that endpoint and the
ported ``send_message`` flow (which inlines diagrams into the generated TSD) use
one implementation. Logic is unchanged from the original function: sanitize →
normalize → render in headless Chromium (Mermaid v11, falling back to v10.9.1) →
write SVG to the public asset dir → return a public URL.
"""
import base64
import hashlib
import html
import os
import re
import uuid
from urllib.parse import quote

from core.logging import logger

try:
    from playwright.async_api import async_playwright

    PLAYWRIGHT_AVAILABLE = True
except ImportError:  # pragma: no cover - environment dependent
    PLAYWRIGHT_AVAILABLE = False


def sha_preview(s: str) -> str:
    """Short digest for debugging to ensure identical text reaches the browser."""
    h = hashlib.sha256(s.encode("utf-8")).hexdigest()
    return f"{len(s)} bytes, sha256[:12]={h[:12]}"


def sanitize_mermaid(code: str) -> str:
    """Normalize Mermaid code (BOM/NBSP, HTML entities, newlines, smart quotes)."""
    if not code:
        return code

    code = code.replace("﻿", "")  # BOM
    code = code.replace(" ", " ")  # NBSP -> space
    code = code.replace("\t", "    ")  # tabs -> spaces
    code = code.replace("\r\n", "\n").replace("\r", "\n")

    code = html.unescape(html.unescape(code))
    code = code.replace("&gt;", ">").replace("&lt;", "<").replace("&amp;", "&")

    replacements = {
        "--&amp;gt;": "-->",
        "-.-&amp;gt;": "-.->",
        "&amp;lt;--": "<--",
        "&amp;lt;-.-": "<-.-",
        "&amp;lt;--&amp;gt;": "<-->",
        "&amp;lt;-.-&amp;gt;": "<-.->",
    }
    for k, v in replacements.items():
        code = code.replace(k, v)

    code = code.replace("“", '"').replace("”", '"').replace("’", "'").replace("‘", "'")
    code = "\n".join(ln.rstrip() for ln in code.split("\n"))
    return code.strip()


def normalize_header_and_split(code: str) -> str:
    """Ensure 'graph/flowchart <dir>' header is alone; normalize to 'flowchart'."""
    out: list[str] = []
    header_rx = re.compile(
        r"^\s*(graph|flowchart)\s+([A-Za-z]+)\s*(.*)$", re.IGNORECASE
    )
    for ln in code.splitlines():
        m = header_rx.match(ln)
        if m:
            direction = m.group(2)
            rest = (m.group(3) or "").strip()
            out.append(f"flowchart {direction}")
            if rest:
                out.append(rest)
        else:
            out.append(ln)
    return "\n".join(out)


def normalize_labels_in_nodes(code: str) -> str:
    """Remove parentheses inside node labels (avoids v11 tokenizer edge cases)."""

    def repl(m):
        inside = m.group(1)
        cleaned = re.sub(r"\(([^)]*)\)", r" \1", inside)
        cleaned = re.sub(r"\s{2,}", " ", cleaned).strip()
        return f"[{cleaned}]"

    return re.sub(r"\[([^\]]+)\]", repl, code)


def force_one_statement_per_line(code: str) -> str:
    """Enforce one Mermaid statement per line."""
    code = re.sub(r"\]\s+(?=[A-Za-z_][\w-]*\s*\[)", "]\n", code)
    code = re.sub(
        r"(?m)(?<!^)\s+(?=(click|style|linkStyle|subgraph|end)\b)", "\n", code
    )
    fixed = []
    for ln in code.splitlines():
        if not ln.strip():
            fixed.append(ln)
            continue
        indent = len(ln) - len(ln.lstrip(" "))
        body = re.sub(r"\s{2,}", " ", ln.lstrip(" "))
        fixed.append(" " * indent + body)
    return "\n".join(fixed)


def extract_mermaid_from_content(content: str) -> str | None:
    """Extract Mermaid code from a message string (fenced blocks first)."""
    if not content:
        return None

    fenced = re.compile(
        r"```(?:\s*)(?:mermaid|flowchart)(?:[^\n]*)\n([\s\S]+?)```",
        flags=re.IGNORECASE | re.MULTILINE,
    )
    matches = fenced.findall(content)
    if matches:
        return matches[-1].strip()

    start_re = re.compile(
        r"^(graph\s+\w+|flowchart\s+\w+|sequenceDiagram|classDiagram|stateDiagram(?:-vg)?|erDiagram)\b",
        flags=re.IGNORECASE | re.MULTILINE,
    )
    m = start_re.search(content)
    if not m:
        return None

    tail = content[m.start() :]
    lines = tail.splitlines()
    out: list[str] = []
    for ln in lines:
        if ln.strip() == "" and out:
            break
        out.append(ln)
    return ("\n".join(out)).strip() or None


def prepare_mermaid(raw_code: str) -> str:
    """Full pipeline to prepare a robust, v11-friendly Mermaid string."""
    mermaid_code = sanitize_mermaid(raw_code)
    mermaid_code = normalize_header_and_split(mermaid_code)
    mermaid_code = normalize_labels_in_nodes(mermaid_code)
    mermaid_code = force_one_statement_per_line(mermaid_code)
    return mermaid_code


async def render_in_browser(html_doc: str, chrome_path: str | None) -> str:
    """Launch headless Chromium and return SVG or raise with the browser error."""
    if not PLAYWRIGHT_AVAILABLE:
        raise RuntimeError(
            "playwright is not available. Install it to use browser rendering."
        )

    async with async_playwright() as p:
        launch_kwargs = {
            "headless": True,
            "args": [
                "--no-sandbox",
                "--disable-gpu",
                "--disable-dev-shm-usage",
                "--disable-setuid-sandbox",
            ],
        }

        # Playwright prefers using channel for system Chrome
        if chrome_path and "chrome" in chrome_path.lower():
            launch_kwargs["channel"] = "chrome"
        elif chrome_path:
            launch_kwargs["executable_path"] = chrome_path

        browser = await p.chromium.launch(**launch_kwargs)
        try:
            page = await browser.new_page()
            page.on(
                "console",
                lambda msg: logger.debug(
                    "[mermaid console]", type=msg.type, text=msg.text
                ),
            )
            page.on(
                "pageerror",
                lambda err: logger.debug("[mermaid pageerror]", error=str(err)),
            )

            await page.goto(
                "data:text/html," + quote(html_doc), wait_until="networkidle"
            )
            await page.wait_for_function(
                "window.__MERMAID_DONE__ === true || window.__MERMAID_DONE__ === 'error'",
                timeout=25000,
            )
            done_state = await page.evaluate("window.__MERMAID_DONE__")
            if done_state == "error":
                err_msg = await page.evaluate("window.__MERMAID_ERR__")
                raise RuntimeError(err_msg or "Unknown Mermaid error")
            return await page.evaluate("window.__MERMAID_SVG__")
        finally:
            await browser.close()


def make_html(mermaid_code: str, mermaid_ver: str) -> str:
    """HTML page that decodes base64 code, parses, and renders with Mermaid."""
    code_b64 = base64.b64encode(mermaid_code.encode("utf-8")).decode("ascii")
    code_digest = sha_preview(mermaid_code)
    mermaid_src = (
        f"https://cdn.jsdelivr.net/npm/mermaid@{mermaid_ver}/dist/mermaid.esm.min.mjs"
    )

    return f"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta http-equiv="Content-Security-Policy" content="default-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net data:;">
  <title>Mermaid Render</title>
</head>
<body>
  <script type="module">
    import mermaid from '{mermaid_src}';
    console.log("Mermaid version source:", "{mermaid_ver}");

    const b64 = "{code_b64}";
    const bin = atob(b64);
    const bytes = new Uint8Array(bin.length);
    for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
    const code = new TextDecoder('utf-8').decode(bytes);

    window.__MERMAID_DONE__ = false;
    window.__MERMAID_SVG__ = '';
    window.__MERMAID_ERR__ = '';

    console.log("Mermaid code digest (browser):", "{code_digest}");
    console.log("Header preview:", code.split('\\n')[0]);

    (async () => {{
      try {{
        mermaid.initialize({{
          startOnLoad: false,
          securityLevel: 'strict',
          flowchart: {{ htmlLabels: false }}
        }});

        try {{
          await mermaid.parse(code);
        }} catch (parseErr) {{
          const msg = (parseErr && (parseErr.str || parseErr.message)) ? (parseErr.str || parseErr.message) : String(parseErr);
          console.error('Mermaid parse error:', msg);
          window.__MERMAID_ERR__ = 'error:' + msg;
          window.__MERMAID_DONE__ = 'error';
          return;
        }}

        const {{ svg }} = await mermaid.render('mmd-1', code);
        window.__MERMAID_SVG__ = svg;
        window.__MERMAID_DONE__ = true;
      }} catch (err) {{
        const msg = (err && err.message) ? err.message : String(err);
        console.error('Mermaid render error:', msg);
        window.__MERMAID_ERR__ = 'error:' + msg;
        window.__MERMAID_SVG__ = '';
        window.__MERMAID_DONE__ = 'error';
      }}
    }})();
  </script>
</body>
</html>"""


def write_svg(public_dir: str, file_path: str, svg_data: str) -> None:
    """Blocking filesystem write (offload to a worker thread from async code)."""
    os.makedirs(public_dir, exist_ok=True)
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(svg_data)


async def render_mermaid_to_url(
    raw_code: str,
    public_base_url: str,
    public_dir: str,
    chrome_path: str | None,
) -> str:
    """Render Mermaid to an SVG file and return its public URL.

    Runs the full prepare → render (v11, fallback v10.9.1) → write pipeline.
    Raises ``RuntimeError`` if both Mermaid versions fail or the SVG is empty.
    """
    import asyncio

    mermaid_code = prepare_mermaid(raw_code)

    html_v11 = make_html(mermaid_code, "11")
    try:
        svg_data = await render_in_browser(html_v11, chrome_path)
    except Exception as e_v11:
        logger.info(f"v11 failed, retrying with Mermaid 10.9.1: {e_v11}")
        html_v10 = make_html(mermaid_code, "10.9.1")
        svg_data = await render_in_browser(html_v10, chrome_path)

    if not svg_data:
        raise RuntimeError("Rendered SVG is empty.")

    filename = f"{uuid.uuid4()}.svg"
    file_path = os.path.join(public_dir, filename)
    await asyncio.to_thread(write_svg, public_dir, file_path, svg_data)

    return f"{public_base_url.rstrip('/')}/diagrams/{filename}"



================================================
FILE: src/services/mongo_models.py
================================================
"""
MongoDB document schemas for the Dev chat databases.

These Pydantic models are derived from the live `dev_chatbot_db`
collections, so they match production documents field-for-field:

  - ``dev_chat_collection``  -> :class:`ChatMessage`
  - ``dev_session_context``  -> :class:`SessionContext`

They give the (otherwise schemaless) Mongo layer a single, validated contract:
the write path constructs a model and serializes it with :meth:`to_document`,
and the read path validates raw documents back into models. Keeping both paths
on the same schema is what prevents silent drift between what we write and what
the rest of the code expects to read.
"""
from datetime import datetime
from typing import Any, Literal, Optional

from pydantic import BaseModel, ConfigDict, Field

# Roles observed in production are "user"/"assistant"; "tool" is also accepted
# because the Mermaid history-scan walks tool messages as well.
Role = Literal["user", "assistant", "tool"]


class ChatMessage(BaseModel):
    """A single chat-history message (``dev_chat_collection``)."""

    # Ignore Mongo-managed/unknown fields (e.g. ``_id``) when validating reads.
    model_config = ConfigDict(extra="ignore")

    workspace_id: str
    user_id: str
    session_id: str
    role: Role
    content: str
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    # Optional fields: stored only when present (see ``to_document``). ``job_id``
    # tags messages that belong to an async job; ``file`` carries an attachment.
    job_id: Optional[str] = None
    file: Optional[Any] = None

    def to_document(self) -> dict:
        """Serialize to a Mongo document.

        ``None`` optionals are dropped so the stored shape matches the existing
        collection (e.g. a message without a job has no ``job_id`` key at all,
        rather than ``job_id: null``).
        """
        return self.model_dump(exclude_none=True)


class SessionContext(BaseModel):
    """Per-session context/metadata (``dev_session_context``).

    ``set_context`` may persist arbitrary metadata alongside the known fields,
    so unknown keys are preserved (``extra="allow"``) rather than rejected.
    """

    model_config = ConfigDict(extra="allow")

    workspace_id: str
    user_id: str
    session_id: str
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    last_activity_at: datetime = Field(default_factory=datetime.utcnow)
    message_count: int = 0
    is_deleted: bool = False
    # Human-friendly conversation title shown in the UI synopsis.
    title: Optional[str] = None
    is_custom_title: bool = False



================================================
FILE: src/services/registry.py
================================================
"""Read-only PostgreSQL registry lookups (agent / workspace / user).

Backed by the ForgeX Postgres DB via ``core.postgres``. Resolves the identity
triple that the agent flow currently only passes around as bare ids:

  * ``get_agent``            → agents_details (+ agents_cms) for an agent_id
  * ``get_workspace``        → workspace_master row
  * ``list_workspace_agents``→ agents mapped to a workspace
  * ``list_workspace_users`` → users mapped to a workspace
  * ``user_in_workspace``    → membership check

Table names come from ``settings.postgres.*`` (config-driven). They are trusted
config values, not user input; row filters are always parameterized (``$1``).
All lookups are best-effort: on any error (or when Postgres is unavailable)
they return ``None`` / ``[]`` / ``False`` so callers can fall back gracefully.
"""
from typing import Any, Dict, List, Optional

from core.config import settings
from core.postgres import get_pg_manager
from core.logging import logger


def _pg():
    return get_pg_manager()


def _row_to_dict(row) -> Dict[str, Any]:
    return dict(row) if row is not None else {}


async def get_agent(agent_id: int) -> Optional[Dict[str, Any]]:
    """Return an agent's combined details + CMS record, or None."""
    d = settings.postgres.PG_AGENT_DETAILS_TABLE
    c = settings.postgres.PG_AGENT_CMS_TABLE
    query = f'''
        SELECT
            d.agent_id        AS agent_id,
            d.agent_name      AS agent_name,
            d.agent_category  AS agent_category,
            d.agent_desc      AS agent_desc,
            d.agent_environment AS agent_environment,
            d.is_active       AS is_active,
            c.agent_owner     AS agent_owner,
            c.agent_contact   AS agent_contact,
            c.agent_feature   AS agent_feature,
            c.faqs            AS faqs
        FROM "{d}" d
        LEFT JOIN "{c}" c ON c.agent_id = d.agent_id
        WHERE d.agent_id = $1
        LIMIT 1
    '''
    try:
        row = await _pg().fetchrow(query, agent_id)
        return _row_to_dict(row) or None
    except Exception as e:  # noqa: BLE001
        logger.error("registry.get_agent failed", error=e)
        return None


async def get_workspace(workspace_id: int) -> Optional[Dict[str, Any]]:
    """Return a workspace_master row, or None."""
    w = settings.postgres.PG_WORKSPACE_TABLE
    query = f'''
        SELECT workspace_id, namespace, workspace_name, workspace_desc, is_active
        FROM "{w}"
        WHERE workspace_id = $1
        LIMIT 1
    '''
    try:
        row = await _pg().fetchrow(query, workspace_id)
        return _row_to_dict(row) or None
    except Exception as e:  # noqa: BLE001
        logger.error("registry.get_workspace failed", error=e)
        return None


async def list_workspace_agents(workspace_id: int) -> List[Dict[str, Any]]:
    """Return active agents mapped to a workspace (joined to agents_details)."""
    m = settings.postgres.PG_WORKSPACE_AGENT_MAPPING_TABLE
    d = settings.postgres.PG_AGENT_DETAILS_TABLE
    query = f'''
        SELECT
            d.agent_id       AS agent_id,
            d.agent_name     AS agent_name,
            d.agent_category AS agent_category,
            d.is_active      AS is_active
        FROM "{m}" m
        JOIN "{d}" d ON d.agent_id = m.agent_id
        WHERE m.workspace_id = $1 AND m.is_active = TRUE
        ORDER BY d.agent_name
    '''
    try:
        rows = await _pg().fetch(query, workspace_id)
        return [dict(r) for r in rows]
    except Exception as e:  # noqa: BLE001
        logger.error("registry.list_workspace_agents failed", error=e)
        return []


async def list_workspace_users(workspace_id: int) -> List[Dict[str, Any]]:
    """Return active users mapped to a workspace (joined to the users table)."""
    m = settings.postgres.PG_WORKSPACE_USER_MAPPING_TABLE
    u = settings.postgres.PG_USER_TABLE
    query = f'''
        SELECT
            u.user_id    AS user_id,
            u.email_id   AS email_id,
            u.first_name AS first_name,
            u.last_name  AS last_name,
            m.role_id    AS role_id,
            m.is_active  AS is_active
        FROM "{m}" m
        JOIN "{u}" u ON u.user_id = m.user_id
        WHERE m.workspace_id = $1 AND m.is_active = TRUE
        ORDER BY u.first_name, u.last_name
    '''
    try:
        rows = await _pg().fetch(query, workspace_id)
        return [dict(r) for r in rows]
    except Exception as e:  # noqa: BLE001
        logger.error("registry.list_workspace_users failed", error=e)
        return []


async def user_in_workspace(workspace_id: int, user_id: int) -> bool:
    """True if an active mapping exists for (workspace_id, user_id)."""
    m = settings.postgres.PG_WORKSPACE_USER_MAPPING_TABLE
    query = f'''
        SELECT 1 FROM "{m}"
        WHERE workspace_id = $1 AND user_id = $2 AND is_active = TRUE
        LIMIT 1
    '''
    try:
        val = await _pg().fetchval(query, workspace_id, user_id)
        return val is not None
    except Exception as e:  # noqa: BLE001
        logger.error("registry.user_in_workspace failed", error=e)
        return False



================================================
FILE: src/services/run_events.py
================================================
"""In-memory per-job event channel for SSE streaming.

Mirrors the ForwardEngineering "FEP" deep-agent producer/consumer pattern
(`agentic_routing._push_event` / `_event_generator`). A background task drives
`agent.astream(...)` and pushes events into a per-job `asyncio.Queue`; the SSE
endpoint drains that queue. Events are also appended to `event_history` so a
client that connects (or reconnects) late can replay everything from the start.

This is process-local and non-durable by design; the durable job status/result
still lives in Mongo via `services.dev_jobs`.
"""
from __future__ import annotations

import asyncio
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional

from core.logging import logger

# Terminal events that close an SSE stream.
TERMINAL_EVENTS = ("complete", "error", "stopped")


@dataclass
class RunRecord:
    """Live streaming state for a single job."""

    job_id: str
    event_queue: "asyncio.Queue[Dict[str, Any]]" = field(default_factory=asyncio.Queue)
    event_history: List[Dict[str, Any]] = field(default_factory=list)
    agent_task: Optional[asyncio.Task] = None
    status: str = "queued"
    done: bool = False
    # Monotonic per-job sequence number; used to order durable event replay.
    seq: int = 0


# Process-global registry keyed by job_id.
RUNS: Dict[str, RunRecord] = {}


def get_or_create_record(job_id: str) -> RunRecord:
    record = RUNS.get(job_id)
    if record is None:
        record = RunRecord(job_id=job_id)
        RUNS[job_id] = record
    return record


def get_record(job_id: str) -> Optional[RunRecord]:
    return RUNS.get(job_id)


def push_event(record: RunRecord, event: str, data: Optional[Dict[str, Any]] = None) -> None:
    """Append an event to history, enqueue it, and mirror it to Mongo.

    The in-memory queue drives live subscribers; the durable copy (written
    best-effort as a fire-and-forget task) backs replay after a restart.
    """
    payload = data or {}
    item = {"event": event, "data": payload}
    record.event_history.append(item)
    record.event_queue.put_nowait(item)

    seq = record.seq
    record.seq += 1
    _schedule_persist(record.job_id, seq, event, payload)

    if event in TERMINAL_EVENTS:
        record.done = True
        record.status = event


def _schedule_persist(job_id: str, seq: int, event: str, data: Dict[str, Any]) -> None:
    """Fire-and-forget durable write of an event (no-op if no running loop).

    Imported lazily to avoid a circular import (``stream_events`` imports this
    module for ``TERMINAL_EVENTS``).
    """
    try:
        from services.stream_events import persist_event

        asyncio.get_running_loop().create_task(
            persist_event(job_id, seq, event, data)
        )
    except RuntimeError:
        # No running event loop (e.g. called from sync context) — skip durable
        # persistence; the in-memory record still works for live subscribers.
        pass
    except Exception as e:  # noqa: BLE001
        logger.error("Failed to schedule stream event persistence", error=e)


def drop_record(job_id: str) -> None:
    RUNS.pop(job_id, None)



================================================
FILE: src/services/session_managers.py
================================================
"""
Session managers ported from the MCP server
(`agent_dev.utils.session_history_manager`).

Same Mongo collections, same query/aggregation logic, so the REST API
returns byte-for-byte equivalent results to the MCP tools. No mocks.
"""
import logging
import uuid
from datetime import datetime

from core.config import settings
from core.database import db_manager

from .mongo_models import ChatMessage, SessionContext


def _setting_str(key: str, *, required: bool = False) -> str:
    """Read a string setting from the central config, validating presence."""
    val = getattr(settings.database, key, None)
    if required and (val is None or (isinstance(val, str) and val.strip() == "")):
        raise ValueError(f"Required setting '{key}' is missing or empty.")
    return val


class SessionHistoryManager:
    """Chat-history store backed by the central async DatabaseManager.

    Each async method below calls `await db_manager.initialize()` first. That
    call is idempotent (a cheap boolean check once the per-process client
    exists), so the connection/pool is built exactly once on cold start and
    callers never need to remember to initialize beforehand.
    """

    def __init__(self):
        self.chat_coll_name = _setting_str("DEV_CHAT_COLLECTION", required=True)

    @property
    def chat_collection(self):
        return db_manager.get_mongo_collection(self.chat_coll_name)

    @property
    def context_collection(self):
        return db_manager.get_mongo_collection(
            _setting_str("DEV_SESSION_CONTEXT", required=True)
        )

    @staticmethod
    def create_session() -> str:
        """Generate a new session ID."""
        return f"{datetime.now().strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:8]}"

    async def append_message(
        self, workspace_id, user_id, session_id, role, content, job_id=None, file=None
    ):
        await db_manager.initialize()
        try:
            # Validate + serialize through the schema. ``to_document`` drops
            # unset optionals, so the stored shape matches the collection
            # (no ``job_id``/``file`` keys when they aren't provided).
            message = ChatMessage(
                workspace_id=workspace_id,
                user_id=user_id,
                session_id=session_id,
                role=role,
                content=content,
                job_id=job_id or None,
                file=file or None,
            )
            insert_result = await self.chat_collection.insert_one(message.to_document())

            now = datetime.utcnow()
            title_seed = (content or "").strip()
            should_seed_title = role == "user" and bool(title_seed)

            existing_context = await self.context_collection.find_one(
                {
                    "workspace_id": workspace_id,
                    "user_id": user_id,
                    "session_id": session_id,
                },
                {
                    "_id": 0,
                    "title": 1,
                    "is_custom_title": 1,
                },
            )

            set_update = {
                "updated_at": now,
                "last_activity_at": now,
                "is_deleted": False,
            }
            
            # Determine if we should update title in $set
            will_update_title_in_set = False
            if should_seed_title:
                existing_custom = bool(
                    (existing_context or {}).get("is_custom_title", False)
                )
                # Avoid targeting the same field in both $setOnInsert and $set
                # during a single upsert (Mongo write error code 40).
                if existing_context is not None and not existing_custom:
                    set_update["title"] = title_seed
                    set_update["is_custom_title"] = False
                    will_update_title_in_set = True
            
            # Build $setOnInsert, excluding fields that are in $set to avoid conflicts
            set_on_insert = {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "session_id": session_id,
                "created_at": now,
            }
            if not will_update_title_in_set:
                set_on_insert["title"] = title_seed if should_seed_title else None
                set_on_insert["is_custom_title"] = False

            await self.context_collection.update_one(
                {
                    "workspace_id": workspace_id,
                    "user_id": user_id,
                    "session_id": session_id,
                },
                {
                    "$setOnInsert": set_on_insert,
                    "$set": set_update,
                    "$inc": {"message_count": 1},
                },
                upsert=True,
            )
            return insert_result.inserted_id
        except Exception as e:
            logging.error(f"Error in append_message: {e}")
            raise

    async def get_recent_sessions(self, workspace_id, user_id, limit=1000):
        await db_manager.initialize()
        try:
            query = {"workspace_id": workspace_id, "user_id": user_id, "job_id": None}
            pipeline = [
                {"$match": query},
                {
                    "$group": {
                        "_id": "$session_id",
                        "latest_timestamp": {"$max": "$timestamp"},
                    }
                },
                {"$sort": {"latest_timestamp": -1}},
                {"$limit": limit},
            ]
            sessions = await self.chat_collection.aggregate(pipeline).to_list(
                length=limit
            )
            session_ids = [str(s["_id"]) for s in sessions]
            return session_ids
        except Exception as e:
            logging.error(f"Error in get_recent_sessions: {e}")
            raise

    async def load_history(
        self, workspace_id: str, user_id: str, session_id: str, limit: int | None = None
    ):
        await db_manager.initialize()
        try:
            query = {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "session_id": session_id,
            }
            messages = (
                await self.chat_collection.find(query)
                .sort("timestamp", 1)
                .to_list(length=None)
            )
            if limit:
                messages = messages[-limit:]
            # Validate raw documents back into the schema, then project the
            # fields the API exposes (omitting ``file`` when not present).
            parsed = [ChatMessage.model_validate(m) for m in messages]
            return [
                {
                    "role": m.role,
                    "content": m.content,
                    "timestamp": m.timestamp,
                    **({"file": m.file} if m.file is not None else {}),
                }
                for m in parsed
            ]
        except Exception as e:
            logging.error(f"Error in load_history: {e}")
            raise

    async def delete_session(self, workspace_id, user_id, session_id):
        await db_manager.initialize()
        try:
            query = {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "session_id": session_id,
            }
            delete_result = await self.chat_collection.delete_many(query)
            return {
                "deleted_count": delete_result.deleted_count,
                "status": "success"
                if delete_result.deleted_count > 0
                else "no records found",
            }
        except Exception as e:
            logging.error(f"Error in delete_session: {e}")
            raise


class SessionContextManager:
    """Per-session context/metadata store backed by the central DatabaseManager.

    As with `SessionHistoryManager`, each async method initializes the
    `db_manager` first (idempotent), so callers don't need to do so.
    """

    def __init__(self):
        self.context_coll_name = _setting_str(
            "DEV_SESSION_CONTEXT", required=True
        )

    @property
    def chat_collection(self):
        return db_manager.get_mongo_collection(self.context_coll_name)

    async def set_context(
        self, workspace_id: str, user_id: str, session_id: str, context: dict
    ):
        await db_manager.initialize()
        filter = {
            "workspace_id": workspace_id,
            "user_id": user_id,
            "session_id": session_id,
        }
        now = datetime.utcnow()

        existing = await self.chat_collection.find_one(filter)
        base_doc = {
            "workspace_id": workspace_id,
            "user_id": user_id,
            "session_id": session_id,
            "created_at": (existing or {}).get("created_at", now),
            "updated_at": now,
            "last_activity_at": (existing or {}).get("last_activity_at", now),
            "message_count": int((existing or {}).get("message_count", 0) or 0),
            "is_deleted": bool((existing or {}).get("is_deleted", False)),
            "title": (existing or {}).get("title"),
            "is_custom_title": bool((existing or {}).get("is_custom_title", False)),
        }
        if context is not None:
            base_doc.update(context)

        validated = SessionContext(**base_doc)
        update = {
            "$set": {
                **validated.model_dump(exclude_none=True),
                "updated_at": now,
            }
        }
        try:
            result = await self.chat_collection.update_one(filter, update, upsert=True)
            if result.upserted_id:
                return {
                    "status": "success",
                    "operation": "created",
                    "upserted_id": str(result.upserted_id),
                }
            return {
                "status": "success",
                "operation": "updated",
                "matched_count": result.matched_count,
                "modified_count": result.modified_count,
            }
        except Exception as e:
            logging.error(f"Error in set_context: {e}")
            raise

    async def get_context(self, workspace_id: str, user_id: str, session_id: str):
        await db_manager.initialize()
        try:
            query = {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "session_id": session_id,
            }
            context_doc = await self.chat_collection.find_one(query)
            if context_doc:
                context_doc.pop("_id", None)
                return {"status": "success", **context_doc}
            else:
                return {"status": "success", "context": "Session context is empty"}
        except Exception as e:
            logging.error(f"Error in get_context: {e}")
            raise

    async def list_contexts(self, workspace_id: str, user_id: str, limit: int = 100):
        await db_manager.initialize()
        query = {
            "workspace_id": workspace_id,
            "user_id": user_id,
            "is_deleted": {"$ne": True},
        }
        cursor = (
            self.chat_collection.find(query).sort("last_activity_at", -1).limit(limit)
        )
        docs = await cursor.to_list(length=limit)
        out = []
        for doc in docs:
            doc.pop("_id", None)
            out.append(doc)
        return out

    async def delete_context(self, workspace_id: str, user_id: str, session_id: str):
        await db_manager.initialize()
        try:
            query = {
                "workspace_id": workspace_id,
                "user_id": user_id,
                "session_id": session_id,
            }
            update_result = await self.chat_collection.update_many(
                query,
                {
                    "$set": {
                        "is_deleted": True,
                        "updated_at": datetime.utcnow(),
                    }
                },
            )
            return {
                "deleted_count": update_result.modified_count,
                "status": "success"
                if update_result.modified_count > 0
                else "no records found",
            }
        except Exception as e:
            logging.error(f"Error in delete_context: {e}")
            raise



================================================
FILE: src/services/skills.py
================================================
"""Skill runtime (Azure Blob Storage + Redis cache).

Implements the storage model in SKILL_BLOB_STORAGE.md.

Source of truth: Azure Blob Storage
Hot cache: Redis (best-effort)

Blob path format:
  workspace/{workspace_id}/agents/{agent_id}/{skill_id}__{version}.md

Blob content format:
  <!-- forgex-skill:v1 -->\n<JSON>

Best-effort behavior:
- If Redis is unavailable, it still works via Blob.
- If Blob is unavailable (infra error), the request should fail (do not silently
  fall back to local defaults).
- If a skill is missing in Redis + Blob, seed Blob once from local defaults and
  then re-fetch from Blob/Redis.
"""

from __future__ import annotations

import json
import os
from datetime import datetime
from pathlib import Path
from typing import Any, Dict, List, Optional

from azure.storage.blob import BlobServiceClient
from azure.core.exceptions import (  # type: ignore
    ClientAuthenticationError,
    HttpResponseError,
    ResourceNotFoundError,
    ServiceRequestError,
)

from core.config import settings
from core.logging import logger

try:
    import redis  # type: ignore

    _REDIS_AVAILABLE = True
except Exception:  # noqa: BLE001
    redis = None  # type: ignore
    _REDIS_AVAILABLE = False


def _repo_root() -> Path:
    return Path(__file__).resolve().parents[2]


def _base_prompt_path() -> Path:
    root = _repo_root()
    return root / "src" / "dev" / "skills" / "skill.md"


def _definitions_dir() -> Path:
    root = _repo_root()
    return root / "src" / "dev" / "skills" / "definitions"


def _markdown_seed_path(skill_id: str) -> Path:
    """Optional raw-markdown seed file for a specific skill.

    This supports dynamic workflow orchestration where the local markdown is
    only a seed source and Blob/Redis are the runtime source of truth.
    """

    root = _repo_root()

    # Preferred explicit filename for the Dev dynamic workflow orchestrator seed.
    # The runtime source of truth remains Blob/Redis; this file is seed-only.
    p = root / "dev_dynamic_workflow_skill.md"
    if p.exists():
        return p

    # Fallback: co-locate under the skills directory.
    candidate = root / "src" / "dev" / "skills" / f"{skill_id}.md"
    if candidate.exists():
        return candidate
    
    return p


def _seed_markdown_to_blob(
    *, workspace_id: str, agent_id: int, skill_id: str, version: str
) -> bool:
    """Seed a raw-markdown skill file into Blob once.

    Returns True if a seed upload happened, False otherwise.
    """

    p = _markdown_seed_path(skill_id)
    if not p.exists():
        return False
    try:
        text = p.read_text(encoding="utf-8")
    except Exception:
        return False
    if not isinstance(text, str) or not text.strip():
        return False

    _put_blob_text(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(version),
        text=text,
    )
    return True


def read_base_prompt_text() -> str:
    p = _base_prompt_path()
    if not p.exists():
        return ""
    try:
        return p.read_text(encoding="utf-8").strip()
    except Exception:
        return ""


def _load_definition_files() -> List[Dict[str, Any]]:
    defs_dir = _definitions_dir()
    if not defs_dir.exists() or not defs_dir.is_dir():
        return []

    out: List[Dict[str, Any]] = []
    for p in sorted(defs_dir.glob("*.json")):
        try:
            data = json.loads(p.read_text(encoding="utf-8"))
            if isinstance(data, dict):
                out.append(data)
        except Exception:
            logger.warning("Failed to read skill definition file", path=str(p))
    return out


def _normalize_container_name(name: str) -> str:
    return (name or "").strip().lower().replace("_", "-")


def _skills_container_name() -> str:
    override = (settings.azure.SKILL_BLOB_CONTAINER_NAME or "").strip()
    if override:
        return _normalize_container_name(override)

    env = (settings.ENVIRONMENT or "development").lower()
    if env in {"local", "development"}:
        return _normalize_container_name(settings.azure.SKILL_BLOB_DEV_CONTAINER)
    return _normalize_container_name(settings.azure.SKILL_BLOB_STAGE_CONTAINER)


def _blob_service() -> BlobServiceClient:
    conn_str = settings.azure.BLOB_STORAGE_CONNECTION_STRING
    if not conn_str:
        raise RuntimeError("BLOB_STORAGE_CONNECTION_STRING is not configured")
    return BlobServiceClient.from_connection_string(conn_str)


def _blob_path(*, workspace_id: str, agent_id: int, skill_id: str, version: str) -> str:
    return f"workspace/{workspace_id}/agents/{int(agent_id)}/{skill_id}__{version}.md"


def _redis_url() -> str:
    # Redis settings are not modeled in Settings in this repo; read from env.
    return (
        os.getenv("REDIS_URL")
        or os.getenv("REDIS_CONNECTION_STRING")
        or os.getenv("REDIS")
        or ""
    ).strip()


def _redis_client():
    if not _REDIS_AVAILABLE:
        return None
    url = _redis_url()
    if not url:
        return None
    try:
        return redis.Redis.from_url(url, decode_responses=True)
    except Exception:
        return None


def _redis_key_skill(*, workspace_id: str, agent_id: int, skill_id: str, version: str) -> str:
    return f"skills:{workspace_id}:{int(agent_id)}:{skill_id}:{version}"


def _redis_key_active(*, workspace_id: str, agent_id: int, skill_id: str) -> str:
    return f"skills-active:{workspace_id}:{int(agent_id)}:{skill_id}"


def _redis_key_manifest(*, workspace_id: str, agent_id: int) -> str:
    return f"skills-manifest:{workspace_id}:{int(agent_id)}"


async def download_skill_blob(
    *, workspace_id: str, agent_id: int, skill_id: str, version: Optional[str] = None
) -> Dict[str, Any]:
    """Download raw markdown from Blob for a skill.

    If version is omitted, resolves active version from Redis when available,
    otherwise uses the default version from local definitions.
    """

    v = (version or "").strip()
    r = _redis_client()
    if not v and r is not None:
        try:
            active_ver = r.get(
                _redis_key_active(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                )
            )
            if isinstance(active_ver, str) and active_ver.strip():
                v = active_ver.strip()
        except Exception:
            pass

    if not v:
        defs = _load_definition_files()
        for d in defs:
            if str(d.get("skill_id")) == str(skill_id):
                v = str(d.get("version") or "1.0.0")
                break
    v = v or "1.0.0"

    # Ensure a missing skill triggers seed behavior (doc-aligned).
    await get_active_skill(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
    )

    md = _get_blob_text(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(v),
    )
    if md is None:
        md = ""
    return {
        "workspace_id": str(workspace_id),
        "agent_id": int(agent_id),
        "skill_id": str(skill_id),
        "version": str(v),
        "skill_markdown": md,
    }


def _skill_markdown_from_doc(doc: Dict[str, Any]) -> str:
    payload = json.dumps(doc, ensure_ascii=False)
    return "<!-- forgex-skill:v1 -->\n" + payload


def _parse_skill_markdown(md: str) -> Dict[str, Any]:
    md = (md or "").strip()
    if not md:
        return {}
    if md.startswith("<!-- forgex-skill:v1 -->"):
        rest = md.split("\n", 1)[1] if "\n" in md else ""
        try:
            obj = json.loads(rest.strip())
            return obj if isinstance(obj, dict) else {}
        except Exception:
            return {}
    try:
        obj = json.loads(md)
        return obj if isinstance(obj, dict) else {}
    except Exception:
        return {}


def _generated_fallback_skill(*, workspace_id: str, agent_id: int, skill_id: str) -> Dict[str, Any]:
    now = datetime.utcnow().isoformat()
    return {
        "workspace_id": str(workspace_id),
        "agent_id": int(agent_id),
        "skill_id": str(skill_id),
        "name": str(skill_id),
        "version": "0.0.0",
        "status": "active",
        "description": "Generated fallback skill (no seeded defaults found).",
        "role": "Dev",
        "objective": "Execute the request with best-effort quality.",
        "instructions": [],
        "constraints": [],
        "allowed_tools": [],
        "output_format": "markdown",
        "variables": {},
        "metadata": {"category": "fallback"},
        "source": "generated_fallback",
        "created_at": now,
        "updated_at": now,
    }


def _skill_rules_text(doc: Dict[str, Any]) -> str:
    role = doc.get("role") or ""
    objective = doc.get("objective") or ""
    instructions = (
        doc.get("instructions") if isinstance(doc.get("instructions"), list) else []
    )
    constraints = (
        doc.get("constraints") if isinstance(doc.get("constraints"), list) else []
    )

    parts: List[str] = ["Skill (Runtime Rules):"]
    if role:
        parts.append(f"Role: {role}")
    if objective:
        parts.append(f"Objective: {objective}")
    if instructions:
        parts.append("Instructions:")
        parts.extend([f"- {str(x)}" for x in instructions if str(x).strip()])
    if constraints:
        parts.append("Constraints:")
        parts.extend([f"- {str(x)}" for x in constraints if str(x).strip()])
    return "\n".join(parts).strip()


def validate_skill_doc(doc: Dict[str, Any]) -> None:
    """Validate minimal skill schema.

    This is intentionally lightweight: it prevents obviously broken skill blobs
    from silently degrading prompt construction.
    """

    if not isinstance(doc, dict) or not doc:
        raise ValueError("Skill document is empty or invalid")

    required = ["skill_id", "version", "role", "objective", "output_format"]
    missing = [k for k in required if not str(doc.get(k, "")).strip()]
    if missing:
        raise ValueError(f"Skill document missing required fields: {', '.join(missing)}")

    for list_key in ["instructions", "constraints", "allowed_tools"]:
        v = doc.get(list_key)
        if v is not None and not isinstance(v, list):
            raise ValueError(f"Skill field '{list_key}' must be a list")


def build_system_prompt(*, base_prompt: str, skill_doc: Dict[str, Any], task_prompt: str) -> str:
    validate_skill_doc(skill_doc)
    blocks = [
        (base_prompt or "").strip(),
        _skill_rules_text(skill_doc),
        (task_prompt or "").strip(),
    ]
    return "\n\n".join([b for b in blocks if b]).strip()


def _get_blob_text(
    *, workspace_id: str, agent_id: int, skill_id: str, version: str
) -> Optional[str]:
    """Download one skill blob.

    Returns:
    - str when the blob exists and is readable
    - None when the blob is missing (404)

    Raises:
    - RuntimeError for infra/auth/network errors (must not silently fallback)
    """
    container = _skills_container_name()
    blob_name = _blob_path(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(version),
    )
    try:
        svc = _blob_service()
        bc = svc.get_blob_client(container=container, blob=blob_name)
        return bc.download_blob().readall().decode("utf-8")
    except ResourceNotFoundError:
        # Missing blob is a valid state (seed may apply).
        logger.debug(
            "Skill blob missing",
            container=container,
            blob=blob_name,
        )
        return None
    except (ClientAuthenticationError, ServiceRequestError) as exc:
        # Auth or network issue: this is an infra error and must fail.
        logger.error(
            "Skill blob read failed (infra)",
            container=container,
            blob=blob_name,
            error=str(exc),
        )
        raise RuntimeError("Skill blob storage unavailable") from exc
    except HttpResponseError as exc:
        # Any other non-404 HTTP error should be treated as infra.
        status = getattr(exc, "status_code", None)
        if status == 404:
            logger.debug(
                "Skill blob missing",
                container=container,
                blob=blob_name,
            )
            return None

        logger.error(
            "Skill blob read failed (infra)",
            container=container,
            blob=blob_name,
            status_code=status,
            error=str(exc),
        )
        raise RuntimeError("Skill blob storage unavailable") from exc
    except Exception as exc:
        # Unknown error: treat as infra to avoid silent fallbacks.
        logger.error(
            "Skill blob read failed (infra/unknown)",
            container=container,
            blob=blob_name,
            error=str(exc),
        )
        raise RuntimeError("Skill blob storage unavailable") from exc


def _put_blob_text(
    *, workspace_id: str, agent_id: int, skill_id: str, version: str, text: str
) -> None:
    container = _skills_container_name()
    blob_name = _blob_path(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(version),
    )
    svc = _blob_service()
    cc = svc.get_container_client(container)
    try:
        cc.create_container()
    except Exception:
        pass
    bc = cc.get_blob_client(blob_name)
    bc.upload_blob(text.encode("utf-8"), overwrite=True, content_type="text/markdown")


def _seed_defaults_to_blob(*, workspace_id: str, agent_id: int) -> None:
    """Seed defaults from definitions/*.json into blob on first lookup (best-effort)."""
    defs = _load_definition_files()
    if not defs:
        return
    for d in defs:
        sid = d.get("skill_id")
        if not isinstance(sid, str) or not sid.strip():
            continue
        version = str(d.get("version") or "1.0.0")
        doc = dict(d)
        # Ensure basic identifiers for blob-based scheme.
        doc["workspace_id"] = str(workspace_id)
        doc["agent_id"] = int(agent_id)
        doc["skill_id"] = sid.strip()
        md = _skill_markdown_from_doc(doc)
        existing = _get_blob_text(
            workspace_id=str(workspace_id),
            agent_id=int(agent_id),
            skill_id=sid.strip(),
            version=version,
        )
        if existing is None:
            try:
                _put_blob_text(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=sid.strip(),
                    version=version,
                    text=md,
                )
            except Exception:
                # Best-effort seeding.
                continue


async def get_active_skill(
    *, workspace_id: str, agent_id: int, skill_id: str
) -> Dict[str, Any]:
    """Fetch active skill with Redis-first lookup and Blob fallback.

    Source of truth is Blob + Redis.
    - If missing in Redis+Blob, seed Blob once from local defaults and re-fetch.
    - If Blob is unavailable (infra error), fail instead of silently falling back.
    """

    # Keep JSON definitions seeding best-effort. Infra errors must not break requests.
    try:
        _seed_defaults_to_blob(workspace_id=str(workspace_id), agent_id=int(agent_id))
    except Exception:
        # Seeding is convenience only; runtime source of truth remains Blob/Redis.
        logger.warning(
            "Skill defaults seeding failed (best-effort)",
            workspace_id=str(workspace_id),
            agent_id=int(agent_id),
        )

    r = _redis_client()
    if r is not None:
        try:
            active_ver = r.get(
                _redis_key_active(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                )
            )
            if isinstance(active_ver, str) and active_ver.strip():
                cached = r.get(
                    _redis_key_skill(
                        workspace_id=str(workspace_id),
                        agent_id=int(agent_id),
                        skill_id=str(skill_id),
                        version=active_ver.strip(),
                    )
                )
                if isinstance(cached, str) and cached.strip():
                    return json.loads(cached)
        except Exception:
            # Redis is best-effort.
            pass

    # Blob fallback. If no active version in redis, choose the default version from definitions.
    defs = _load_definition_files()
    version = None
    for d in defs:
        if str(d.get("skill_id")) == str(skill_id):
            version = str(d.get("version") or "1.0.0")
            break
    version = version or "1.0.0"

    # Blob fallback. Missing skills should be seeded from local markdown once.
    # _get_blob_text raises RuntimeError for infra/auth errors.
    md = _get_blob_text(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(version),
    )

    if not isinstance(md, str) or not md.strip():
        # One-time seed for markdown-defined orchestrators (e.g. BA dynamic workflow).
        seeded = False
        try:
            seeded = _seed_markdown_to_blob(
                workspace_id=str(workspace_id),
                agent_id=int(agent_id),
                skill_id=str(skill_id),
                version=str(version),
            )
        except Exception as exc:
            logger.error(
                "Skill seed upload failed (infra)",
                container=_skills_container_name(),
                blob=_blob_path(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                    version=str(version),
                ),
                error=str(exc),
            )
            raise

        if seeded:
            # Re-fetch from Blob as the runtime source of truth.
            md = _get_blob_text(
                workspace_id=str(workspace_id),
                agent_id=int(agent_id),
                skill_id=str(skill_id),
                version=str(version),
            )

    if isinstance(md, str) and md.strip():
        doc = _parse_skill_markdown(md)
        if not doc:
            doc = _generated_fallback_skill(
                workspace_id=str(workspace_id),
                agent_id=int(agent_id),
                skill_id=str(skill_id),
            )

        if r is not None:
            try:
                r.set(
                    _redis_key_active(
                        workspace_id=str(workspace_id),
                        agent_id=int(agent_id),
                        skill_id=str(skill_id),
                    ),
                    str(version),
                )
                r.set(
                    _redis_key_skill(
                        workspace_id=str(workspace_id),
                        agent_id=int(agent_id),
                        skill_id=str(skill_id),
                        version=str(version),
                    ),
                    json.dumps(doc, ensure_ascii=False),
                )
            except Exception:
                pass
        return doc

    # Skill is still missing after attempted seed. Keep existing generated fallback.
    return _generated_fallback_skill(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
    )


async def update_active_skill(
    *, workspace_id: str, agent_id: int, skill_id: str, updates: Dict[str, Any]
) -> Dict[str, Any]:
    """Persist a new skill version to Blob and mark it active in Redis."""
    # Load current version (or default).
    r = _redis_client()
    current_version = None
    if r is not None:
        try:
            current_version = r.get(
                _redis_key_active(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                )
            )
        except Exception:
            current_version = None
    if not current_version:
        defs = _load_definition_files()
        for d in defs:
            if str(d.get("skill_id")) == str(skill_id):
                current_version = str(d.get("version") or "1.0.0")
                break
    current_version = str(current_version or "1.0.0")

    md = _get_blob_text(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=current_version,
    )
    base_doc = _parse_skill_markdown(md or "") if md else {}
    if not base_doc:
        base_doc = _generated_fallback_skill(
            workspace_id=str(workspace_id),
            agent_id=int(agent_id),
            skill_id=str(skill_id),
        )

    # Apply updates. Version is required for new blob name.
    new_version = str((updates or {}).get("version") or base_doc.get("version") or "1.0.0")
    doc = dict(base_doc)
    doc.update({k: v for k, v in (updates or {}).items() if k != "skill_markdown"})
    doc["workspace_id"] = str(workspace_id)
    doc["agent_id"] = int(agent_id)
    doc["skill_id"] = str(skill_id)
    doc["version"] = new_version
    doc["updated_at"] = datetime.utcnow().isoformat()

    md_out = _skill_markdown_from_doc(doc)
    _put_blob_text(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(new_version),
        text=md_out,
    )

    if r is not None:
        try:
            r.set(
                _redis_key_active(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                ),
                str(new_version),
            )
            r.set(
                _redis_key_skill(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                    version=str(new_version),
                ),
                json.dumps(doc, ensure_ascii=False),
            )
        except Exception:
            pass

    return doc


async def upload_skill_blob(
    *, workspace_id: str, agent_id: int, skill_id: str, version: str, skill_markdown: str
) -> Dict[str, Any]:
    """Upload raw markdown to Blob and refresh Redis (used by /api/upload-skill-blob)."""
    _put_blob_text(
        workspace_id=str(workspace_id),
        agent_id=int(agent_id),
        skill_id=str(skill_id),
        version=str(version),
        text=str(skill_markdown or ""),
    )
    doc = _parse_skill_markdown(skill_markdown or "")
    if not doc:
        doc = {
            "workspace_id": str(workspace_id),
            "agent_id": int(agent_id),
            "skill_id": str(skill_id),
            "version": str(version),
            "status": "active",
        }

    r = _redis_client()
    if r is not None:
        try:
            r.set(
                _redis_key_active(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                ),
                str(version),
            )
            r.set(
                _redis_key_skill(
                    workspace_id=str(workspace_id),
                    agent_id=int(agent_id),
                    skill_id=str(skill_id),
                    version=str(version),
                ),
                json.dumps(doc, ensure_ascii=False),
            )
        except Exception:
            pass

    return doc



================================================
FILE: src/services/stream_events.py
================================================
"""Durable persistence for live-streaming events (SSE / WebSocket).

The in-memory ``run_events.RunRecord`` gives low-latency fan-out to connected
clients but is process-local and lost on restart. This module mirrors every
event into MongoDB (collection ``DEV_STREAM_EVENTS_COLLECTION``) so a
client can replay a job's full event history after a process restart or from a
different instance.

Document schema: ``{job_id, seq, event, data, created_at, terminal}``.
Unique index on ``(job_id, seq)`` (declared in ``core.database``).
"""
from datetime import datetime
from typing import Any, Dict, List, Optional

from core.config import settings
from core.database import db_manager
from core.logging import logger

from services.run_events import TERMINAL_EVENTS


def _collection():
    return db_manager.get_mongo_collection(
        settings.database.DEV_STREAM_EVENTS_COLLECTION
    )


async def persist_event(job_id: str, seq: int, event: str, data: Dict[str, Any]) -> None:
    """Best-effort durable write of a single stream event.

    Streaming must not fail if persistence hiccups, so all errors are swallowed
    (the in-memory queue remains the source of truth for live subscribers).
    """
    try:
        await db_manager.initialize()
        await _collection().update_one(
            {"job_id": job_id, "seq": seq},
            {
                "$set": {
                    "job_id": job_id,
                    "seq": seq,
                    "event": event,
                    "data": data or {},
                    "terminal": event in TERMINAL_EVENTS,
                    "created_at": datetime.utcnow(),
                }
            },
            upsert=True,
        )
    except Exception as e:  # noqa: BLE001 - streaming persistence is non-critical
        logger.error("Failed to persist stream event", error=e)


async def load_events(job_id: str) -> List[Dict[str, Any]]:
    """Return a job's persisted events in emission order (empty on miss/error)."""
    try:
        await db_manager.initialize()
        cursor = _collection().find({"job_id": job_id}, {"_id": 0}).sort("seq", 1)
        return [doc async for doc in cursor]
    except Exception as e:  # noqa: BLE001
        logger.error("Failed to load stream events", error=e)
        return []


async def is_done(job_id: str) -> Optional[bool]:
    """True if a terminal event was persisted for the job; None if no events."""
    events = await load_events(job_id)
    if not events:
        return None
    return any(e.get("terminal") for e in events)



================================================
FILE: src/services/workflow_context.py
================================================
"""WorkflowContextManager — project-context curation for the Dev agent.

Agent-specific: curates project artifacts (e.g. BRD) from the workflow database /
Azure Blob storage to feed TSD generation. Fully async on Motor via the shared
``core.database.db_manager`` (no sync pymongo client).
"""
from __future__ import annotations

import logging
from typing import Any, Optional

from core import settings
from core.database import db_manager


class WorkflowContextManager:
    def __init__(self):
        # Prefer artifact collection; fall back to legacy WORKFLOW_COLLECTION_NAME.
        artifact_coll = settings.workflow.WORKFLOW_ARTIFACT_COLLECTION_NAME
        legacy_coll = settings.workflow.WORKFLOW_COLLECTION_NAME
        collection_name = artifact_coll or legacy_coll
        # Track which collection type is in use to build correct queries.
        self.using_artifacts_collection = bool(artifact_coll)
        if not collection_name:
            raise ValueError(
                "Workflow collection name is not set. Set "
                "WORKFLOW_ARTIFACT_COLLECTION_NAME or WORKFLOW_COLLECTION_NAME."
            )
        self._collection_name = collection_name

        # Optional Azure Blob Storage integration.
        # Initialize from settings when configured; keep graceful fallback
        # to support environments where blob access is intentionally disabled.
        self.blob_service_client: Optional[Any] = None
        self.blob_container_name: Optional[str] = None

        conn_str = settings.azure.BLOB_STORAGE_CONNECTION_STRING
        container_name = settings.azure.BLOB_CONTAINER_NAME
        if conn_str and container_name:
            try:
                from azure.storage.blob import BlobServiceClient

                self.blob_service_client = BlobServiceClient.from_connection_string(
                    conn_str
                )
                self.blob_container_name = container_name
            except Exception as exc:
                logging.warning(
                    "Blob storage initialization failed; continuing without blob access: %s",
                    exc,
                )

    def _fetch_blob_content(self, blob_name: str) -> str:
        """Download and extract text from a DOCX blob in Azure Storage."""
        if self.blob_service_client is None:
            logging.warning(
                f"Cannot fetch blob '{blob_name}' - Blob storage client not initialized"
            )
            return ""

        if not self.blob_container_name:
            logging.warning(
                f"Cannot fetch blob '{blob_name}' - Blob container name not set"
            )
            return ""

        try:
            import io

            from docx import Document

            blob_client = self.blob_service_client.get_blob_client(
                container=self.blob_container_name, blob=blob_name
            )
            logging.info(f"[Dev] Fetching blob: {blob_name}")

            blob_data = blob_client.download_blob()
            blob_bytes = blob_data.readall()

            doc = Document(io.BytesIO(blob_bytes))
            text_content = []
            for paragraph in doc.paragraphs:
                if paragraph.text.strip():
                    text_content.append(paragraph.text.strip())
            for table in doc.tables:
                for row in table.rows:
                    for cell in row.cells:
                        if cell.text.strip():
                            text_content.append(cell.text.strip())

            full_text = "\n".join(text_content)
            logging.info(
                f"[Dev] Fetched {len(full_text)} characters from blob '{blob_name}'"
            )
            return full_text
        except Exception as e:
            logging.error(
                f"[Dev] Error fetching blob content from '{blob_name}': {e}"
            )
            import traceback

            logging.error(traceback.format_exc())
            return ""

    async def curate_context(
        self, workspace_id: str, job_id: str, agent_feed: list | None
    ):
        """Curate project context from workflow_jobs / workflow_artifacts.

        STANDALONE MODE: if job_id is None, returns "" (no project context).
        NPD FLOW MODE: if job_id is provided, fetches artifacts and raises if none found.
        """
        try:
            if job_id is None:
                logging.info(
                    "[STANDALONE MODE] job_id is None. Running dev agent without project context."
                )
                return ""

            jobs_coll = db_manager.get_workflow_collection(
                settings.workflow.WORKFLOW_COLLECTION_NAME
            )
            artifacts_coll = db_manager.get_workflow_collection(
                settings.workflow.WORKFLOW_ARTIFACT_COLLECTION_NAME
            )

            job_query = {"workspace_id": workspace_id, "job_id": job_id}
            job_doc = await jobs_coll.find_one(job_query)
            job_artifacts = job_doc.get("job_artifacts", {}) if job_doc else {}

            if not agent_feed or (
                isinstance(agent_feed, list) and len(agent_feed) == 0
            ):
                agent_feed = ["BRD", "brd"]

            context_parts = []
            not_found = []

            for feed_item in agent_feed:
                value = job_artifacts.get(feed_item)
                if isinstance(value, str) and value.strip():
                    context_parts.append(f"{feed_item}: {value.strip()}")
                elif isinstance(value, dict) and "blob_name" in value:
                    blob_name = value.get("blob_name")
                    blob_content = self._fetch_blob_content(blob_name)
                    if blob_content:
                        context_parts.append(f"{feed_item}: {blob_content}")
                    else:
                        logging.warning(
                            f"Blob fetch failed for {feed_item} in job_artifacts (blob_name={blob_name})"
                        )
                        not_found.append(feed_item)
                else:
                    not_found.append(feed_item)

            if not_found:
                art_query = {"job_id": job_id}
                art_doc = await artifacts_coll.find_one(art_query)
                art_dict = art_doc.get("artifacts", {}) if art_doc else {}
                for feed_item in not_found:
                    value = None
                    if feed_item in art_dict:
                        value = art_dict[feed_item]
                    else:
                        lower_map = {str(k).lower(): k for k in art_dict.keys()}
                        if feed_item.lower() in lower_map:
                            value = art_dict[lower_map[feed_item.lower()]]
                    if isinstance(value, dict) and "blob_name" in value:
                        blob_name = value.get("blob_name")
                        blob_content = self._fetch_blob_content(blob_name)
                        if blob_content:
                            context_parts.append(f"{feed_item}: {blob_content}")
                        else:
                            logging.warning(
                                f"Blob fetch failed for {feed_item} in workflow_artifacts (blob_name={blob_name})"
                            )
                    elif isinstance(value, str) and value.strip():
                        context_parts.append(f"{feed_item}: {value.strip()}")
                    else:
                        logging.warning(
                            f"Artifact '{feed_item}' not found in workflow_jobs or workflow_artifacts for job_id={job_id}"
                        )

            context_string = "\n".join(context_parts)
            if context_string:
                return context_string
            # No upstream artifacts in the workflow DB. In a dynamic flow, upstream
            # deliverables live in the SHARED blob workflow folder, not these Mongo
            # collections — so degrade gracefully instead of hard-failing the whole
            # flow. The agent reads upstream work via its blob tools (list_outputs /
            # read_output / read_input over <ws>/dynamic_workflow/<job>).
            logging.warning(
                "No workflow-DB artifacts for job_id=%s; falling back to the shared "
                "workflow folder (agent should call list_outputs/read_output/read_input "
                "for upstream deliverables).",
                job_id,
            )
            return ""
        except Exception as e:
            logging.error(f"Error in curate_context: {e}")
            raise



================================================
FILE: src/services/workspace_tools.py
================================================
"""Per-session workspace + workspace-aware agent tools.

Each session's files live under a folder structure that mirrors the deep-agent
blob layout (``global``, ``large_tool_results``, ``memory``, ``skills``) plus a
``workspace`` folder that holds the actual design artifacts::

    <session_id>/
        global/
        large_tool_results/
        memory/
        skills/
        workspace/
            intake/          <- uploaded input documents (read-only to agents)
            dev/       <- the Dev agent's generated outputs
            PO/              <- other agents' folders, created dynamically
            DEV/
            ...

The ``workspace/<agent>`` folders are created **dynamically per agent role**:
every agent writes its outputs only into its *own* folder, but may **read** the
intake documents and every other agent's generated documents. This "read-all,
write-own" model is enforced at the tool layer (see ``create_workspace_tools``).

All reads and writes go through the **active deep-agent backend** selected by
``CLOUD_STORAGE_PROVIDER``:

* ``filesystem`` -> physical files under ``WORKSPACE_ROOT/<session_id>/...``
* ``azure``      -> blobs under ``<container>/<session_id>/...``
* ``s3``/``aws`` -> objects under ``<bucket>/<session_id>/...``

Routing everything through ``_build_backend`` (the same factory the agent itself
uses, prefixed by session id) keeps the agent's own file tools and these helper
tools writing to the exact same location.
"""
from __future__ import annotations

import json
import logging
import re
from dataclasses import dataclass
from pathlib import Path
from typing import Any, Dict, List, Optional, Union

logger = logging.getLogger("dev.workspace_tools")

from langchain_core.tools import tool

from core.config import settings

# ---------------------------------------------------------------------------
# Session layout constants
# ---------------------------------------------------------------------------

# Sibling folders created inside every session (mirror the deep-agent blob
# layout). The agent artifacts themselves live under ``workspace/``.
SESSION_FOLDERS = ("global", "large_tool_results", "memory", "skills")

# Root of the design artifacts inside a session.
WORKSPACE_SUBDIR = "workspace"
# Uploaded input documents live here; agents may read but never write them.
INTAKE_SUBDIR = f"{WORKSPACE_SUBDIR}/intake"
# Session-level sibling folders that hold, respectively, the resolved skill the
# agent ran with and the persisted chat transcript. Writing a real file into each
# is what makes them materialise in blob (blob stores have no empty folders).
SKILLS_SUBDIR = "skills"
MEMORY_SUBDIR = "memory"
MEMORY_TRANSCRIPT_FILE = "chat_history.json"
# Default agent folder when no registry profile is available.
DEFAULT_AGENT_FOLDER = "dev"

# Run-scope roots (see build_run_location). A run is either part of a shared
# multi-agent *dynamic workflow* (keyed by workflow_job_id — shared intake +
# memory + read-all outputs) or an *individual* agent run (its own isolated
# workspace + memory, persistent per agent).
RUN_SCOPE_DYNAMIC = "dynamic_workflow"
RUN_SCOPE_INDIVIDUAL = "individual"


def _slug(value, fallback: str) -> str:
    """Filesystem/blob-safe token from an arbitrary value (fallback if empty)."""
    slug = re.sub(r"[^A-Za-z0-9_-]+", "_", str(value or "").strip()).strip("_")
    return slug or fallback


@dataclass(frozen=True)
class RunLocation:
    """Where a single agent run's files live and how its docs are registered.

    Decouples the physical location (``local_key`` / ``blob_prefix``) from the
    document-registry key (``db_session_id``):

    * ``local_key``   — path under ``WORKSPACE_ROOT`` for the local mirror.
    * ``blob_prefix`` — blob/object prefix (kept identical to ``local_key``).
    * ``db_session_id`` — key written to ``fe_agent_document`` and Azure config
      (``workflow_job_id`` for dynamic runs so the whole workflow groups
      together; the conversation id for individual runs so the UI keeps
      fetching per conversation).
    """

    local_key: str
    blob_prefix: str
    db_session_id: str
    scope: str
    agent_folder: str
    workspace_id: Optional[str] = None
    workflow_job_id: Optional[str] = None


def build_run_location(
    workspace_id,
    *,
    conversation_id,
    workflow_job_id=None,
    agent_folder: Optional[str] = None,
) -> RunLocation:
    """Resolve the :class:`RunLocation` for a run from its request identifiers.

    Presence of ``workflow_job_id`` selects the shared *dynamic workflow* root
    (``<workspace_id>/dynamic_workflow/<job>``); its absence selects the
    per-agent, per-conversation *individual* root
    (``<workspace_id>/individual/<agent>/<conversation_id>``) so two
    conversations run by the same agent never share a blob prefix / local
    workspace and each gets its own isolated document storage.
    """
    wsid = _slug(workspace_id, "workspace")
    agent = _slug(agent_folder or DEFAULT_AGENT_FOLDER, DEFAULT_AGENT_FOLDER)
    if workflow_job_id:
        run_key = f"{RUN_SCOPE_DYNAMIC}/{_slug(workflow_job_id, 'job')}"
        scope = RUN_SCOPE_DYNAMIC
        db_sid = str(workflow_job_id)
    else:
        run_key = f"{RUN_SCOPE_INDIVIDUAL}/{agent}/{_slug(conversation_id, 'conversation')}"
        scope = RUN_SCOPE_INDIVIDUAL
        db_sid = str(conversation_id)
    key = f"{wsid}/{run_key}"
    return RunLocation(
        local_key=key,
        blob_prefix=key,
        db_session_id=db_sid,
        scope=scope,
        agent_folder=agent,
        workspace_id=(str(workspace_id) if workspace_id is not None else None),
        workflow_job_id=(str(workflow_job_id) if workflow_job_id else None),
    )


def _legacy_location(session_id: str, workspace_id: Optional[str] = None) -> RunLocation:
    """RunLocation preserving the pre-restructure ``<workspace_id>/<session_id>`` layout.

    Used when a caller only has a session id (history reads, direct document
    fetches) so those paths keep resolving old blobs/files unchanged.
    """
    sid = str(session_id)
    blob_prefix = f"{_slug(workspace_id, 'workspace')}/{sid}" if workspace_id is not None else sid
    return RunLocation(
        local_key=sid,
        blob_prefix=blob_prefix,
        db_session_id=sid,
        scope=RUN_SCOPE_INDIVIDUAL,
        agent_folder=DEFAULT_AGENT_FOLDER,
        workspace_id=(str(workspace_id) if workspace_id is not None else None),
    )


def _as_location(
    loc_or_session,
    workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> RunLocation:
    """Coerce the (session_id, workspace_id, run_location) inputs to a RunLocation.

    ``run_location`` wins when supplied; otherwise a legacy location is built
    from the session id (+ optional workspace id).
    """
    if run_location is not None:
        return run_location
    if isinstance(loc_or_session, RunLocation):
        return loc_or_session
    return _legacy_location(loc_or_session, workspace_id)


def agent_folder_slug(agent_profile: Optional[dict]) -> str:
    """Derive the per-agent workspace folder name from a registry profile.

    Uses the agent's registered name (falling back to its category), slugified to
    a filesystem/blob-safe token. When no profile is available this falls back to
    ``dev`` so the Dev service keeps a stable folder.
    """
    if agent_profile:
        name = agent_profile.get("agent_name") or agent_profile.get("agent_category")
        if name:
            slug = re.sub(r"[^A-Za-z0-9_-]+", "_", str(name).strip()).strip("_")
            if slug:
                return slug
    return DEFAULT_AGENT_FOLDER


def agent_output_subdir(agent_folder: str) -> str:
    """Backend-relative folder an agent writes its outputs into."""
    folder = (agent_folder or DEFAULT_AGENT_FOLDER).strip("/") or DEFAULT_AGENT_FOLDER
    return f"{WORKSPACE_SUBDIR}/{folder}"


def get_workspace_root() -> Path:
    root = Path(settings.deep_agent.WORKSPACE_ROOT).resolve()
    # DISABLED: cloud-only persistence must not create a local workspaces root.
    # root.mkdir(parents=True, exist_ok=True)
    return root


def _local_root(loc: RunLocation) -> Path:
    """Local-mirror directory for a run (base for filesystem/disk fallbacks)."""
    return get_workspace_root() / loc.local_key


def init_workspace(
    location: Union[str, RunLocation], agent_folder: Optional[str] = None
) -> Path:
    """Create (idempotently) the run's local workspace mirror and return its path.

    ``location`` is either a run-relative key (``<workspace_id>/individual/<agent>/<conversation_id>``
    or a legacy session id) or a :class:`RunLocation` (its ``local_key`` is used).
    Builds the full layout: the sibling folders (``global``,
    ``large_tool_results``, ``memory``, ``skills``), the ``workspace`` root, its
    ``intake`` folder, and — when ``agent_folder`` is given — that agent's own
    output folder. The local directories are always created so the filesystem
    backend and any UI code that inspects the disk have a stable root; cloud
    backends ignore these dirs and address blobs/objects by the prefix instead
    (folders there materialise on first write).
    """
    key = location.local_key if isinstance(location, RunLocation) else str(location)
    ws = get_workspace_root() / key
    # DISABLED: Azure/S3 backends materialise their folders in blob/object
    # storage on first write; no local workspace mirror is required.
    # for folder in SESSION_FOLDERS:
    #     (ws / folder).mkdir(parents=True, exist_ok=True)
    # (ws / INTAKE_SUBDIR).mkdir(parents=True, exist_ok=True)
    # (ws / "codebase").mkdir(parents=True, exist_ok=True)
    # folder = agent_folder_slug({"agent_name": agent_folder}) if agent_folder else DEFAULT_AGENT_FOLDER
    # (ws / agent_output_subdir(folder)).mkdir(parents=True, exist_ok=True)
    return ws


# ---------------------------------------------------------------------------
# Backend routing
# ---------------------------------------------------------------------------

def _backend_for_location(loc: RunLocation):
    """Return the active deep-agent backend for a resolved :class:`RunLocation`.

    Reuses ``agent_builder._build_backend`` (imported lazily to avoid a circular
    import) with an explicit ``blob_prefix`` and ``db_session_id`` so intake /
    output helpers write to the exact same blobs/objects/files the running agent
    does, and register documents under the same registry key.
    """
    from agent_builder import _build_backend  # lazy: avoids circular import

    ws = init_workspace(loc)
    return _build_backend(
        workspace_dir=str(ws),
        blob_prefix=loc.blob_prefix,
        db_session_id=loc.db_session_id,
        doc_meta={
            "workspace_id": loc.workspace_id,
            "workflow_job_id": loc.workflow_job_id,
            "scope": loc.scope,
        },
    )


def _session_backend(
    session_id, workspace_id: Optional[str] = None, run_location: Optional[RunLocation] = None
):
    """Backend for a run, resolved from an explicit RunLocation or legacy ids."""
    return _backend_for_location(_as_location(session_id, workspace_id, run_location))


def _backend_for_workspace(workspace_dir: str, loc: RunLocation):
    """Backend scoped to an existing per-run workspace directory + location."""
    from agent_builder import _build_backend  # lazy: avoids circular import

    return _build_backend(
        workspace_dir=str(workspace_dir),
        blob_prefix=loc.blob_prefix,
        db_session_id=loc.db_session_id,
        doc_meta={
            "workspace_id": loc.workspace_id,
            "workflow_job_id": loc.workflow_job_id,
            "scope": loc.scope,
        },
    )


def _ls_relative(backend, folder: str) -> List[str]:
    """List file paths under ``/<folder>`` relative to that folder.

    Works across every backend (they all implement the sync ``ls`` wrapper), but
    normalises the two shapes seen in the vendored backends: Azure returns an
    ``LsResult`` (``.entries``) while S3 returns a plain ``list[FileInfo]``.
    Directory entries are skipped; only files are returned. Errors degrade to an
    empty list so callers never crash on a missing/empty folder.
    """
    base = "/" + folder.strip("/")
    try:
        result = backend.ls(base)
    except Exception:  # noqa: BLE001 — backend/network hiccups -> treat as empty
        return []

    if isinstance(result, list):
        entries = result
    else:
        entries = getattr(result, "entries", None) or []
    prefix = base + "/"
    files: List[str] = []
    for entry in entries:
        # FileInfo is a dict-like {"path": "/outputs/HLD.md", "is_dir": False}
        path = entry.get("path") if isinstance(entry, dict) else getattr(entry, "path", None)
        is_dir = entry.get("is_dir") if isinstance(entry, dict) else getattr(entry, "is_dir", False)
        if not path or is_dir:
            continue
        rel = path[len(prefix):] if path.startswith(prefix) else path.lstrip("/")
        if rel:
            files.append(rel.replace("\\", "/"))
    return sorted(files)


def _ls_subdirs(backend, folder: str) -> List[str]:
    """Immediate subdirectory names under ``/<folder>`` (best-effort, empty on error).

    The vendored backends' ``ls`` is non-recursive and returns subfolders as
    directory entries; this extracts those names so callers can descend into the
    per-agent folders.
    """
    base = "/" + folder.strip("/")
    try:
        result = backend.ls(base)
    except Exception:  # noqa: BLE001
        return []
    entries = result if isinstance(result, list) else (getattr(result, "entries", None) or [])
    prefix = base + "/"
    names: List[str] = []
    for entry in entries:
        path = entry.get("path") if isinstance(entry, dict) else getattr(entry, "path", None)
        is_dir = entry.get("is_dir") if isinstance(entry, dict) else getattr(entry, "is_dir", False)
        if not path or not is_dir:
            continue
        rel = path[len(prefix):] if path.startswith(prefix) else path.lstrip("/")
        rel = rel.strip("/").replace("\\", "/")
        if rel and "/" not in rel:
            names.append(rel)
    return sorted(names)


def _disk_list(base_dir: Path, folder: str) -> List[str]:
    """List files under ``base_dir/folder`` directly from local disk.

    Used as a fallback for the filesystem provider (and any local mirror) so
    listing keeps working regardless of the FilesystemBackend ``ls`` contract.
    """
    d = Path(base_dir) / folder
    if not d.exists():
        return []
    return sorted(
        str(f.relative_to(d)).replace("\\", "/")
        for f in d.rglob("*")
        if f.is_file()
    )


def _read_via_backend(backend, file_path: str) -> str | None:
    """Read a file's text through the backend; None if missing/unreadable.

    Uses ``download_files`` (raw bytes) rather than ``read``: the S3 backend's
    ``read`` expects a JSON wrapper written by its own ``write``, whereas we
    persist via ``upload_files`` (raw bytes) for overwrite-safety. ``download_files``
    returns raw bytes on both Azure and S3, so it round-trips whatever we wrote.
    """
    try:
        responses = backend.download_files([file_path])
    except Exception:  # noqa: BLE001
        return None
    if not responses:
        return None
    resp = responses[0]
    content = getattr(resp, "content", None) if not isinstance(resp, dict) else resp.get("content")
    error = getattr(resp, "error", None) if not isinstance(resp, dict) else resp.get("error")
    if error or content is None:
        return None
    if isinstance(content, bytes):
        for enc in ("utf-8", "utf-8-sig", "cp1252", "latin-1"):
            try:
                return content.decode(enc)
            except UnicodeDecodeError:
                continue
        return content.decode("utf-8", errors="replace")
    return content


def _write_via_backend(backend, file_path: str, content: str) -> str | None:
    """Write text through the backend via ``upload_files`` (overwrites on both
    Azure and S3). Returns an error string on failure, else None."""
    try:
        responses = backend.upload_files([(file_path, content.encode("utf-8"))])
    except Exception as exc:  # noqa: BLE001
        return str(exc)
    if responses:
        resp = responses[0]
        error = getattr(resp, "error", None) if not isinstance(resp, dict) else resp.get("error")
        if error:
            return str(error)
    return None


# ---------------------------------------------------------------------------
# Intake documents (inputs/)
# ---------------------------------------------------------------------------

def get_inputs_dir(session_id: str) -> Path:
    """Per-session intake-documents directory (created if missing)."""
    d = get_workspace_root() / session_id / INTAKE_SUBDIR
    # DISABLED: intake files are persisted through the active cloud backend.
    # d.mkdir(parents=True, exist_ok=True)
    return d


def save_input_file(
    session_id, filename: str, data: bytes, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> str:
    """Save an uploaded intake document to the run's ``workspace/intake`` folder.

    Writes through the active backend so the file lands in the configured store
    (local disk / Azure blob / S3). Returns the relative path stored. The
    filename is flattened to its basename to prevent path traversal.
    """
    safe_name = Path(filename).name
    rel_path = f"{INTAKE_SUBDIR}/{safe_name}"
    backend = _session_backend(session_id, workspace_id, run_location)
    # upload_files carries raw bytes (handles binary intake docs); backends that
    # only store text will still round-trip UTF-8 content correctly.
    backend.upload_files([(rel_path, data)])
    return rel_path


def save_intake_document(
    session_id, filename: str, data: bytes, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> str:
    """Parse an uploaded intake file to Markdown and save it to ``workspace/intake``.

    This is the canonical intake writer: it converts every upload (``.docx`` /
    ``.pdf`` / plain text) into readable Markdown via
    :func:`intake_parsing.extract_intake_text` and stores it under a ``.md`` name,
    so the agent's ``read_input`` never sees raw binary. If parsing fails or the
    type is unsupported, the original bytes are saved unchanged as a last resort.
    Returns the relative path stored.
    """
    from services import intake_parsing  # local import: keep module import light

    try:
        out_name, text = intake_parsing.extract_intake_text(filename, data)
        return save_input_file(
            session_id, out_name, text.encode("utf-8"), workspace_id, run_location
        )
    except Exception:  # noqa: BLE001 — unsupported/corrupt: fall back to raw bytes
        return save_input_file(session_id, filename, data, workspace_id, run_location)


def list_input_files(
    session_id, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> List[str]:
    """Relative paths of all uploaded intake documents for a run."""
    loc = _as_location(session_id, workspace_id, run_location)
    files = _ls_relative(_backend_for_location(loc), INTAKE_SUBDIR)
    if not files:  # filesystem provider / local mirror fallback
        files = _disk_list(_local_root(loc), INTAKE_SUBDIR)
    return files


def intake_context_note(
    session_id, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> str:
    """A short note listing available intake documents, or ``""`` if none.

    Prepended to the user message at run start so the agent reliably knows intake
    documents exist (uploaded this turn OR a prior turn) instead of depending on
    it remembering to call ``list_inputs`` itself. Bare filenames only — the agent
    reads content via ``read_input(<name>)``.
    """
    files = list_input_files(session_id, workspace_id, run_location)
    names = [Path(f).name for f in files if f]
    if not names:
        return ""
    joined = ", ".join(names)
    return (
        f"[Intake documents already available in the intake folder: {joined}. "
        f"Read each with read_input(<name>) and ground your design in their "
        f"contents before generating any document.]"
    )


_UPLOAD_MARKER_RE = re.compile(r"\[Uploaded Document:\s*(?P<name>[^\]]+)\]\s*\n")


def save_uploaded_documents_from_message(
    session_id, message: str, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> List[str]:
    """Extract inlined intake documents from a chat message and save them to inputs/.

    The UI sends uploaded documents inside the message body as
    ``[Uploaded Document: NAME]\\n\\n<content>`` (it does not POST files). This
    parses each such marker and writes the extracted text into the session
    ``inputs/`` folder (via the active backend) so it shows up in the workspace
    and is readable via the ``list_inputs`` / ``read_input`` agent tools.
    Returns the relative paths saved.
    """
    if not message or "[Uploaded Document:" not in message:
        return []

    markers = list(_UPLOAD_MARKER_RE.finditer(message))
    saved: List[str] = []
    for i, m in enumerate(markers):
        name = m.group("name").strip() or f"document_{i + 1}"
        # The inlined content is already text; store it as readable Markdown so
        # the agent's read_input decodes clean text (never a binary .docx name).
        name = f"{Path(name).stem}.md"
        start = m.end()
        end = markers[i + 1].start() if i + 1 < len(markers) else len(message)
        content = message[start:end].strip()
        if content:
            saved.append(
                save_input_file(
                    session_id, name, content.encode("utf-8"), workspace_id, run_location
                )
            )
    return saved


# ---------------------------------------------------------------------------
# Generated documents (outputs/)
# ---------------------------------------------------------------------------

def _list_generated(backend, base_dir: Path) -> List[str]:
    """List every generated document across all ``workspace/<agent>`` folders.

    Returns paths relative to ``workspace/`` (e.g. ``dev/HLD.md``) so the
    caller can see which agent produced each document. The ``intake`` folder is
    excluded — those are inputs, not generated outputs.

    The vendored backends' ``ls`` is non-recursive, so this descends one level:
    it enumerates the per-agent subfolders under ``workspace`` and lists the files
    inside each. Falls back to a recursive local-disk walk for the filesystem
    provider / local mirror.
    """
    files: List[str] = []
    for sub in _ls_subdirs(backend, WORKSPACE_SUBDIR):
        if sub == "intake":
            continue
        for name in _ls_relative(backend, f"{WORKSPACE_SUBDIR}/{sub}"):
            files.append(f"{sub}/{name}")

    if not files:  # filesystem provider / local mirror fallback
        files = [
            f for f in _disk_list(base_dir, WORKSPACE_SUBDIR)
            if not f.startswith("intake/")
        ]
    return sorted(files)


def list_output_files(
    session_id, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> List[str]:
    """Relative paths (``<agent>/<file>``) of all generated documents for a run."""
    loc = _as_location(session_id, workspace_id, run_location)
    return _list_generated(_backend_for_location(loc), _local_root(loc))


def resolve_pushable_document(
    session_id, filename: str = "", workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> tuple[str | None, str | None, str | None]:
    """Resolve which generated document an external push (Jira/GitHub) should use.

    Returns ``(basename, content, error)`` — exactly one of ``content``/``error``
    is set. Used instead of a hardcoded filename so pushes follow whatever the
    agent actually saved this session:

    - ``filename`` given: read that document (any agent folder, by basename).
    - ``filename`` omitted: auto-pick when exactly one non-coverage-report
      document exists; otherwise return an error listing every generated
      document so the caller can ask the user which one to push.
    """
    candidates = [
        f for f in list_output_files(session_id, workspace_id, run_location)
        if Path(f).name != "coverage-report.md"
    ]
    if filename.strip():
        name = Path(filename.strip()).name
        text = read_output_file(session_id, name, workspace_id, run_location)
        if text is None:
            return None, None, (
                f"Document not found: {name}. Generated documents in this session: "
                + (", ".join(candidates) if candidates else "none yet")
            )
        return name, text, None

    if not candidates:
        return None, None, "No generated documents found in this session yet."
    if len(candidates) > 1:
        return None, None, (
            "Multiple generated documents exist — tell me which one to push by "
            "passing its filename: " + ", ".join(candidates)
        )
    name = Path(candidates[0]).name
    text = read_output_file(session_id, name, workspace_id, run_location)
    if text is None:
        return None, None, f"Document not found: {name}"
    return name, text, None


def _disk_read(base_dir: Path, folder: str, filename: str) -> str | None:
    """Read a file directly from the local run workspace (fallback)."""
    target = Path(base_dir) / folder / Path(filename).name
    if not target.exists() or not target.is_file():
        return None
    try:
        return target.read_text(encoding="utf-8", errors="replace")
    except Exception:  # noqa: BLE001
        return None


def _read_generated(backend, base_dir: Path, filename: str) -> str | None:
    """Read a generated document by ``<agent>/<file>`` path or bare filename.

    When ``filename`` names an agent folder (``dev/HLD.md``) it is read
    directly. A bare filename (``HLD.md``) is resolved by scanning every agent
    folder for a matching basename — so any agent can read any other agent's
    output without knowing which folder it lives in.
    """
    def _read_rel(rel_path: str) -> str | None:
        text = _read_via_backend(backend, f"{WORKSPACE_SUBDIR}/{rel_path}")
        if text is not None:
            return text
        # Local-disk fallback (filesystem provider / local mirror).
        disk = Path(base_dir) / WORKSPACE_SUBDIR / rel_path
        if disk.exists() and disk.is_file():
            try:
                return disk.read_text(encoding="utf-8", errors="replace")
            except Exception:  # noqa: BLE001
                return None
        return None

    rel = filename.strip("/").replace("\\", "/")
    if "/" in rel:
        text = _read_rel(rel)
        if text is not None:
            return text
    target = Path(filename).name
    for path in _list_generated(backend, base_dir):
        if Path(path).name == target:
            text = _read_rel(path)
            if text is not None:
                return text
    return None


def read_output_file(
    session_id, filename: str, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> str | None:
    """Read a generated document's text (from any agent folder) via the backend."""
    loc = _as_location(session_id, workspace_id, run_location)
    backend = _backend_for_location(loc)
    base = _local_root(loc)
    text = _read_generated(backend, base, filename)
    if text is None:  # local mirror fallback (default agent folder)
        text = _disk_read(base, agent_output_subdir(DEFAULT_AGENT_FOLDER), filename)
    return text


def read_input_file(
    session_id, filename: str, workspace_id: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
) -> str | None:
    """Read an uploaded intake document's text through the active backend."""
    loc = _as_location(session_id, workspace_id, run_location)
    rel = f"{INTAKE_SUBDIR}/{Path(filename).name}"
    text = _read_via_backend(_backend_for_location(loc), rel)
    if text is None:  # filesystem provider / local mirror fallback
        text = _disk_read(_local_root(loc), INTAKE_SUBDIR, filename)
    return text


_DOC_TYPES = [
    # PLAN.md is Dev's only canonical design artifact. HLD/LLD/TSD are inputs
    # or upstream documents and must never be selected for fallback output.
    ("PLAN.md", r"build plan|engineering plan|project scaffolding|technical scaffolding|scaffold|technical specification|\bTSD\b|\bPLAN\b"),
]


def parse_requested_docs(user_message: str) -> List[str]:
    """Infer which document filenames the user asked for from their message.

    Returns ``["PLAN.md"]`` only for a scaffold/planning request. Upstream
    HLD/LLD/TSD files are inputs, not Dev output targets.

    Note: this is intentionally conservative. If the user didn't explicitly ask
    for an artifact, we return an empty list so the end-of-run fallback does not
    dump conversational text into a design document.
    """
    text = user_message or ""
    requested = [name for name, pattern in _DOC_TYPES if re.search(pattern, text, re.IGNORECASE)]
    return requested


def persist_final_output(
    session_id,
    content: str,
    agent_folder: Optional[str] = None,
    workspace_id: Optional[str] = None,
    saved_docs: Optional[set] = None,
    requested_docs: Optional[List[str]] = None,
    run_location: Optional[RunLocation] = None,
) -> List[str]:
    """Persist the agent's final markdown to its own ``workspace/<agent>`` folder.

    Prefer artifacts explicitly saved via ``save_output`` / ``append_output``;
    use the narrow requested-document fallback only when no such artifact exists.

    This function used to be a safety net that wrote the model's final assistant
    text into a requested filename (defaulting to ``TSD.md``). That caused
    conversational responses to be persisted into ``TSD.md``. We now avoid any
    auto-write: chat history belongs in Mongo + ``memory/chat_history.json``.

    Type-aware behaviour:
    - ``saved_docs`` is the set of filenames the agent already wrote via
      ``save_output`` during the run. Those are authoritative — we do NOT
      overwrite them or emit a misleading generic file for them.
    - ``requested_docs`` is the list of doc filenames the user asked for
      (see :func:`parse_requested_docs`). When the agent saved nothing and the
      final content is substantive, it is written under the requested filename
      (e.g. ``HLD.md``) rather than a generic ``architecture-design.md``.
    - The markdown heading-split remains a safety net for the case where the
      model inlines a full document instead of calling ``save_output``.

    Returns the list of ``<agent>/<file>`` paths written.
    """
    content = (content or "").strip()
    if not content or content == "(no textual output)":
        return []
    # A saved artifact is authoritative. Never copy the assistant's trailing
    # chat response over it (the old fallback caused PLAN.md contamination).
    if {Path(name).name for name in (saved_docs or set())}:
        return []

    # Greetings, advisory answers, and integration-only turns are not artifacts.
    requested = {Path(name).name for name in (requested_docs or [])}
    if not requested:
        return []

    backend = _session_backend(session_id, workspace_id, run_location)
    folder = agent_folder_slug({"agent_name": agent_folder}) if agent_folder else DEFAULT_AGENT_FOLDER
    out_dir = agent_output_subdir(folder)
    written: List[str] = []

    def _write(name: str, text: str) -> None:
        _write_via_backend(backend, f"{out_dir}/{name}", text.strip() + "\n")
        written.append(f"{folder}/{name}")

    # Best-effort split by top-level markdown headings that name a PLAN document.
    section_map = [
        (filename, pattern)
        for filename, pattern in _DOC_TYPES
        if filename in requested
    ]
    heading_re = re.compile(r"^(#{1,3})\s+(.*)$", re.MULTILINE)
    headings = list(heading_re.finditer(content))
    split = False
    for filename, pattern in section_map:
        pat = re.compile(pattern, re.IGNORECASE)
        for i, m in enumerate(headings):
            if pat.search(m.group(2)):
                start = m.start()
                end = headings[i + 1].start() if i + 1 < len(headings) else len(content)
                # Extend to sibling/deeper headings until a same-or-higher level heading.
                level = len(m.group(1))
                for j in range(i + 1, len(headings)):
                    if len(headings[j].group(1)) <= level:
                        end = headings[j].start()
                        break
                    end = len(content)
                _write(filename, content[start:end])
                split = True
                break

    # If the model returned a requested document without a matching heading,
    # save it under the first requested design filename. Never create a generic
    # combined fallback file containing chat text.
    if not split:
        _write(next(iter(filename for filename, _ in _DOC_TYPES if filename in requested), "PLAN.md"), content)

    return written


# ---------------------------------------------------------------------------
# Session-level folder materialisation (skills/ and memory/)
# ---------------------------------------------------------------------------

def _skill_dict_to_markdown(skill: Dict[str, Any]) -> str:
    """Render a resolved skill definition dict into readable markdown."""
    lines: List[str] = []
    name = skill.get("name") or skill.get("skill_id") or "Skill"
    lines.append(f"# {name}")
    if skill.get("version"):
        lines.append(f"_Version: {skill.get('version')}_")
    lines.append("")
    if skill.get("role"):
        lines.append("## Role")
        lines.append(str(skill["role"]).strip())
        lines.append("")
    if skill.get("objective"):
        lines.append("## Objective")
        lines.append(str(skill["objective"]).strip())
        lines.append("")
    instructions = skill.get("instructions") or []
    if instructions:
        lines.append("## Instructions")
        for inst in instructions:
            lines.append(f"- {inst}")
        lines.append("")
    return "\n".join(lines).rstrip() + "\n"


async def persist_active_skill(
    run_location: RunLocation,
    workspace_id: str,
    agent_id: int = 1,
    skill_id: str = "dev_scaffolding_generation",
) -> Optional[str]:
    """Write the run's active resolved skill into the ``skills/`` folder.

    Best-effort: fetches the currently active skill (Redis+Blob source of truth)
    and writes it as ``skills/<skill_id>.md`` through the run backend so the folder
    materialises in blob with the exact skill the agent ran with. Returns the
    relative path written, or ``None`` on any failure (never raises).
    """
    try:
        from services.skills import get_active_skill

        skill = await get_active_skill(
            workspace_id=str(workspace_id), agent_id=int(agent_id), skill_id=str(skill_id)
        )
        if not isinstance(skill, dict) or not skill:
            return None
        rel_path = f"{SKILLS_SUBDIR}/{_slug(skill_id, 'skill')}.md"
        backend = _backend_for_location(run_location)
        err = _write_via_backend(backend, rel_path, _skill_dict_to_markdown(skill))
        if err:
            logger.warning("persist_active_skill write failed: %s", err)
            return None
        return rel_path
    except Exception as exc:  # noqa: BLE001 — non-critical, never break a run
        logger.warning("persist_active_skill failed: %s", exc)
        return None


async def persist_chat_transcript(
    run_location: RunLocation,
    workspace_id: str,
    user_id: str,
    conversation_id: str,
    limit: int = 200,
) -> Optional[str]:
    """Write the conversation transcript into the ``memory/`` folder.

    Best-effort: loads the turn-by-turn history (Mongo, via SessionHistoryManager)
    and writes it as ``memory/chat_history.json`` through the run backend so the
    folder materialises in blob. The chat window on refresh still loads history
    from Mongo; this is a durable per-run copy. Returns the relative path written,
    or ``None`` on any failure (never raises).
    """
    try:
        from services.session_managers import SessionHistoryManager

        history = await SessionHistoryManager().load_history(
            workspace_id=str(workspace_id),
            user_id=str(user_id),
            session_id=str(conversation_id),
            limit=limit,
        )
        if not history:
            return None
        payload = {
            "workspace_id": str(workspace_id),
            "conversation_id": str(conversation_id),
            "message_count": len(history),
            "messages": history,
        }
        rel_path = f"{MEMORY_SUBDIR}/{MEMORY_TRANSCRIPT_FILE}"
        backend = _backend_for_location(run_location)
        err = _write_via_backend(
            backend, rel_path, json.dumps(payload, indent=2, default=str)
        )
        if err:
            logger.warning("persist_chat_transcript write failed: %s", err)
            return None
        return rel_path
    except Exception as exc:  # noqa: BLE001 — non-critical, never break a run
        logger.warning("persist_chat_transcript failed: %s", exc)
        return None


# ---------------------------------------------------------------------------
# Agent-facing tools
# ---------------------------------------------------------------------------

def create_workspace_tools(
    workspace_dir: str,
    workspace_id: Optional[str] = None,
    agent_folder: Optional[str] = None,
    run_location: Optional[RunLocation] = None,
):
    """Build workspace-scoped save/read/list tools routed through the active backend.

    ``workspace_dir`` is the per-session workspace path; its basename is the
    session id. When ``workspace_id`` is provided the backend blob prefix is
    scoped under the workspace (``<workspace_id>/<session_id>``), so these tools
    read/write the same blobs/objects/files the agent's native tools do.

    ``agent_folder`` is this agent's role folder (default ``dev``). The
    "read-all, write-own" model is enforced here: ``save_output`` only ever
    writes into ``workspace/<agent_folder>``, while the read/list tools span the
    ``intake`` folder and *every* agent's output folder.
    """
    loc = run_location or _legacy_location(Path(workspace_dir).name, workspace_id)
    backend = _backend_for_workspace(workspace_dir, loc)
    ws_path = Path(workspace_dir)
    folder = agent_folder_slug({"agent_name": agent_folder}) if agent_folder else DEFAULT_AGENT_FOLDER
    out_dir = agent_output_subdir(folder)

    @tool
    def save_output(filename: str, content: str = "") -> str:
        """Save (create/overwrite) a generated document (e.g. HLD.md, LLD.md,
        TSD.md) to YOUR OWN agent folder so the user can view and download it.
        You can only write to your own folder.

        For a large document (e.g. a full multi-section TSD) do NOT pass the whole
        document in one call — that can be truncated. Instead write the FIRST
        section with save_output(filename, <section 1>), then add each following
        section with append_output(filename, <next section>)."""
        # ``content`` defaults to "" so a tool call whose arguments were truncated
        # mid-stream (the classic max_tokens cut-off on a giant document) no longer
        # raises a pydantic "content: Field required" error and cannot trigger the
        # regenerate-the-whole-document retry loop. Instead we return actionable
        # guidance to write the document in smaller sections.
        if not (content or "").strip():
            return (
                f"No content received for {folder}/{Path(filename).name} — the "
                "document was likely too large and got truncated. Write it in "
                "smaller pieces: call save_output with the FIRST section only, then "
                "append_output(filename, <next section>) for each remaining section."
            )
        safe = Path(filename).name
        error = _write_via_backend(backend, f"{out_dir}/{safe}", content)
        if error:
            return f"Failed to save {folder}/{safe}: {error}"
        return f"Saved successfully: {folder}/{safe}"

    @tool
    def append_output(filename: str, content: str = "") -> str:
        """Append a section to a document in YOUR OWN agent folder, creating it if
        it does not exist yet. Use this to build a large document (e.g. a full TSD)
        section-by-section: save_output(filename, <first section>) once, then
        append_output(filename, <next section>) for every subsequent section. This
        keeps each write small so nothing is truncated."""
        if not (content or "").strip():
            return (
                f"No content received to append to {folder}/{Path(filename).name}. "
                "Provide the next section's markdown as `content`."
            )
        safe = Path(filename).name
        rel = f"{out_dir}/{safe}"
        existing = _read_via_backend(backend, rel) or ""
        # Ensure sections are separated by a blank line when concatenating.
        joined = content if not existing else f"{existing.rstrip()}\n\n{content.lstrip()}"
        error = _write_via_backend(backend, rel, joined)
        if error:
            return f"Failed to append to {folder}/{safe}: {error}"
        return f"Appended successfully: {folder}/{safe}"

    @tool
    def read_output(filename: str) -> str:
        """Read a previously generated document produced by ANY agent in this
        session. Pass either a bare filename (e.g. HLD.md) or an agent-qualified
        path (e.g. dev/HLD.md). Use list_outputs() to see what exists."""
        text = _read_generated(backend, ws_path, filename)
        if text is None:
            return f"Output file not found: {filename}. It may not have been generated yet."
        return text

    @tool
    def list_inputs() -> str:
        """List intake documents the user uploaded for this session (BRD,
        requirements, existing specs, etc.). Call this first to discover what
        source material is available before generating designs."""
        files = _ls_relative(backend, INTAKE_SUBDIR) or _disk_list(ws_path, INTAKE_SUBDIR)
        if not files:
            return "No intake documents uploaded."
        return "Uploaded intake documents:\n" + "\n".join(files)

    @tool
    def read_input(filename: str) -> str:
        """Read the content of an uploaded intake document from the session
        intake folder (use list_inputs() to see available filenames)."""
        text = _read_via_backend(backend, f"{INTAKE_SUBDIR}/{Path(filename).name}")
        if text is None:
            return f"Intake document not found: {filename}. Use list_inputs() to see available files."
        return text

    @tool
    def list_outputs() -> str:
        """List all documents generated by ANY agent in this session, as
        ``<agent>/<file>`` paths so you can see which agent produced each one.
        You may read all of them but can only write into your own folder."""
        files = _list_generated(backend, ws_path)
        if not files:
            return "No documents generated yet."
        return "Generated documents:\n" + "\n".join(files)

    return [save_output, append_output, read_output, list_outputs, list_inputs, read_input]



================================================
FILE: src/skills/DEV_SCAFFOLDING_SKILL.md
================================================
# Dev Code Scaffolding Skill

## Metadata
- **Skill ID**: dev_scaffolding_generation
- **Version**: 1.0.0
- **Status**: active
- **Agent**: Dev-Dynamic-Agent
- **Created**: 2026-08-28
- **Updated**: 2026-08-28

## Role
Senior Staff Software Engineer with deep expertise in system implementation, clean/layered
architecture, project bootstrapping, and translating a Technical Specification Document (TSD)
into a production-ready code scaffold.

## Objective
Read a TSD and generate (1) a build **PLAN.md** and (2) a real, importable **code scaffold** — a
folder tree of stub files with docstrings, signatures, TODOs, config, entrypoints, a test skeleton,
packaging, and a README. The scaffold is the default deliverable: a development team should be able
to clone it and start implementing immediately. Do NOT write full business logic; scaffold structure,
contracts, and build order.

## Instructions

### 1. Greeting Detection
- If input is a greeting (hi, hello, thanks), respond briefly and politely; do NOT scaffold.
- Example: "Hi! I'm your Dev agent. Point me at a TSD and I'll plan and scaffold the codebase for you."

### 2. Context Gathering
- Call `list_inputs()` and `curate_workflow_context()` to locate intake documents.
- `read_input()` the **TSD** (e.g. `TSD.md`) and any schema/requirements files — read each ONCE.
- Call `load_conversation_history()`; call `list_outputs()` first on update/resume
  requests, then `get_previous_design()` when needed. Continue missing work and do
  not regenerate completed files.
- If no TSD is available, ask for one instead of guessing.
- Retrieve contextual knowledge via `kb_search(query, user_prompt)` when external context is useful.

### 3. Intent Determination
- **New scaffold (default)**: user provides a TSD/requirements → produce PLAN.md then the code tree.
- **Update request**: load the previous scaffold/plan via `get_previous_design()` and extend it,
  preserving existing structure unless a change requires restructuring.

### 4. Planning (do this FIRST — PLAN.md)
Derive an engineering plan from the TSD and save it as `PLAN.md`:
1. **Stack & rationale** — language, framework, datastore, messaging, packaging. Honor what the TSD
   specifies; only default when it is silent (see Engineering Defaults).
2. **Module/service breakdown** — map each component in the TSD to a module/package/service.
3. **Target folder structure** — the full tree as a fenced block, one-line purpose per folder.
   This tree MUST match the files you actually save in step 5.
4. **Build order & milestones** — the sequence a team should implement in, plus key risks/open questions.
5. **Diagram** — a Mermaid component/dependency diagram. Render it via `render_mermaid_diagram()`
   BEFORE saving the chunk that references it, so the saved markdown contains the `![Diagram](url)` link.

#### Incremental Saving (REQUIRED)
Never emit a whole large file in one tool call (it truncates), and never make one tiny call per
function (it exhausts the recursion limit). Build each long file in a FEW LARGE CHUNKS:
1. `save_output('<path>', <chunk 1>)` for the first chunk.
2. `append_output('<path>', <next chunk>)` for each subsequent chunk (~2-5 total for long files).
3. Appends to the SAME file must be SEQUENTIAL — `append_output` reads-appends-rewrites, so parallel
   appends race and lose data. Different files may be created independently. Reads are safe to parallelize.

### 5. Scaffolding (the code tree)
Persist EVERY file via `save_output`/`append_output` — chat text is not a deliverable. Save each file
at its intended relative path with forward slashes (folders are implied by the path), e.g.
`save_output('src/api/routes/claims.py', <stub>)`.

Each stub file must contain:
- A module docstring describing its responsibility.
- Imports and public class/function **signatures with type hints**.
- Concise `# TODO:` markers (or `raise NotImplementedError`) where logic belongs — NOT full logic.
- Obvious wiring only (router registration, dependency injection, config access).

A complete scaffold typically includes:
- **Entrypoint(s)**: `main.py` / app factory (or the ecosystem equivalent).
- **Config**: a settings/config module reading environment variables; `.env.example`.
- **Domain**: models / schemas / entities named from the TSD's real domain.
- **Layers**: `service/` (use-cases) and `repository/` (data access) with interfaces.
- **API**: routes/controllers wired to the entrypoint.
- **Cross-cutting**: logging, error handling, and (if in the TSD) messaging/eventing stubs.
- **Tests**: a mirrored `tests/` tree with one test stub per module and a `conftest`/fixtures file.
- **Packaging/Ops**: `requirements.txt`/`pyproject.toml` (or `package.json`/`pom.xml`), `Dockerfile`, CI stub.
- **README.md**: purpose, prerequisites, setup, run, and test instructions.

Finish and save one file before starting the next.

### 6. Finish and summarize
- Verify the generated files with `list_outputs()`.
- Return a SHORT summary: chosen stack, the top-level tree, and the number of files scaffolded.
- Do NOT paste full file bodies in the final message.

The Dev agent has no design-snapshot tool. Do not call or reference
`save_design_snapshot`; `PLAN.md` is the canonical persisted planning artifact.

## Engineering Defaults (override when the TSD specifies otherwise)
- **Language/Framework**: Python 3.11 + FastAPI (async) for services; adapt to Java/Spring Boot,
  Node/Express/NestJS, etc. when the TSD calls for them.
- **Architecture**: clean layered design (api → service → repository → model), config isolated,
  DI-friendly, 12-factor environment config.
- **Data**: PostgreSQL / Cosmos DB / Redis per the TSD; include model definitions and migration
  placeholders (e.g. an empty `migrations/` with a README).
- **Testing**: pytest (or the ecosystem-standard runner) with a mirrored `tests/` tree and fixtures.
- **Packaging/Ops**: `Dockerfile`, `.env.example`, dependency manifest, a minimal CI workflow stub.

## Output Contract
- `PLAN.md` and `README.md` are Markdown; code files are valid, importable source in the target language.
- Do NOT wrap entire files in code fences inside `save_output` — save the raw file content.
- The folder tree documented in `PLAN.md` must exactly match the files saved.
- Every stub is syntactically valid; no half-written lines, no placeholder junk like `foo/bar`.

## Example (Python/FastAPI target)
PLAN.md tree excerpt:
```
project/
  src/
    main.py            # FastAPI app factory + startup wiring
    core/config.py     # Settings from environment (pydantic BaseSettings)
    api/routes/        # HTTP controllers, one module per resource
    services/          # Use-case/business logic (stubs)
    repositories/      # Data-access interfaces + implementations (stubs)
    models/            # Pydantic schemas / ORM models
  tests/               # Mirrors src/, one test stub per module
  requirements.txt
  Dockerfile
  .env.example
  README.md
```
Stub example (`src/api/routes/claims.py`):
```python
\"\"\"Claims HTTP routes. Wires claim use-cases to FastAPI.\"\"\"
from fastapi import APIRouter, Depends

router = APIRouter(prefix="/claims", tags=["claims"])


@router.get("/{claim_id}")
async def get_claim(claim_id: str):
    \"\"\"Return a single claim by id.\"\"\"
    # TODO: call ClaimService.get(claim_id)
    raise NotImplementedError
```



================================================
FILE: tests/conftest.py
================================================
import os
import sys

import pytest

# Add src to path so we can import modules
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))


@pytest.fixture
def sample_request_data():
    """Sample request data for testing"""
    return {"name": "Test User", "data": {"key": "value"}}


@pytest.fixture
def mock_azure_function_request():
    """Mock Azure Function request object"""

    class MockRequest:
        def __init__(self, params=None, body=None):
            self.params = params or {}
            self._body = body

        def get_json(self):
            return self._body

        def get_body(self):
            return self._body

    return MockRequest



================================================
FILE: tests/test_cancel_job_api.py
================================================
import asyncio
import importlib
import json
import os
import sys

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

from services import dev_jobs

cancel_api = importlib.import_module("functions.api.cancel_job.__init__")


class _Req:
    def __init__(self, params=None, body=None):
        self.params = params or {}
        self._body = body

    def get_json(self):
        if self._body is None:
            raise ValueError("no body")
        return self._body


async def _call(req):
    return await cancel_api._handle_cancel_request(req)


def _response_json(resp):
    return json.loads(resp.get_body().decode("utf-8"))


def test_cancel_job_by_job_id_signals_sync_cancel_conversation(monkeypatch):
    async def _fake_get(job_id):
        assert job_id == "j1"
        return {
            "job_id": "j1",
            "status": "queued",
            "request": {"conversation_id": "c1"},
        }

    async def _fake_cancel(job_id):
        assert job_id == "j1"
        return True

    seen = {}

    def _fake_signal(*, conversation_id=None, job_id=None):
        seen["conversation_id"] = conversation_id
        return {"ok": True}

    monkeypatch.setattr(dev_jobs, "get_job", _fake_get)
    monkeypatch.setattr(dev_jobs, "cancel_job", _fake_cancel)
    monkeypatch.setattr(cancel_api, "cancel_conversation", _fake_signal)

    resp = asyncio.run(_call(_Req(params={"job_id": "j1"})))
    body = _response_json(resp)

    assert resp.status_code == 200
    assert body["success"] is True
    assert body["data"]["conversation_id"] == "c1"
    assert seen["conversation_id"] == "c1"


def test_cancel_job_prefers_explicit_conversation_id(monkeypatch):
    async def _fake_get(_job_id):
        return {
            "job_id": "j1",
            "status": "queued",
            "request": {"conversation_id": "from-job"},
        }

    async def _fake_cancel(_job_id):
        return True

    seen = {}

    def _fake_signal(*, conversation_id=None, job_id=None):
        seen["conversation_id"] = conversation_id

    monkeypatch.setattr(dev_jobs, "get_job", _fake_get)
    monkeypatch.setattr(dev_jobs, "cancel_job", _fake_cancel)
    monkeypatch.setattr(cancel_api, "cancel_conversation", _fake_signal)

    req = _Req(params={"job_id": "j1", "conversation_id": "explicit"})
    resp = asyncio.run(_call(req))
    body = _response_json(resp)

    assert resp.status_code == 200
    assert body["data"]["conversation_id"] == "explicit"
    assert seen["conversation_id"] == "explicit"


def test_cancel_conversation_only_with_async_signal(monkeypatch):
    async def _fake_signal(*, conversation_id=None, job_id=None):
        assert conversation_id == "c-only"
        return {"ok": True}

    monkeypatch.setattr(cancel_api, "cancel_conversation", _fake_signal)

    resp = asyncio.run(_call(_Req(params={"conversation_id": "c-only"})))
    body = _response_json(resp)

    assert resp.status_code == 200
    assert body["success"] is True
    assert body["data"]["conversation_id"] == "c-only"



================================================
FILE: tests/test_dev_jobs.py
================================================
"""Tests for the Mongo-backed send_message job store (faked collection)."""
import asyncio
import os
import sys

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

from services import dev_jobs  # noqa: E402


class _FakeCollection:
    """Minimal async stand-in for a Motor collection."""

    def __init__(self):
        self.docs = {}

    async def insert_one(self, doc):
        self.docs[doc["job_id"]] = dict(doc)

    async def update_one(self, flt, update):
        jid = flt["job_id"]
        self.docs.setdefault(jid, {"job_id": jid}).update(update["$set"])

    async def find_one(self, flt, projection=None):
        doc = self.docs.get(flt["job_id"])
        if doc is None:
            return None
        doc = dict(doc)
        doc.pop("_id", None)
        return doc


def _patch(monkeypatch):
    coll = _FakeCollection()

    async def _init():
        return None

    monkeypatch.setattr(dev_jobs.db_manager, "initialize", _init)
    monkeypatch.setattr(dev_jobs, "_collection", lambda: coll)
    return coll


def test_job_lifecycle(monkeypatch):
    _patch(monkeypatch)

    async def scenario():
        await dev_jobs.create_job("j1", {"arguments": {"x": 1}})
        await dev_jobs.mark_running("j1")
        await dev_jobs.complete_job("j1", {"status": "success", "content": "ok"})
        return await dev_jobs.get_job("j1")

    job = asyncio.run(scenario())
    assert job["status"] == dev_jobs.STATUS_COMPLETED
    assert job["result"] == {"status": "success", "content": "ok"}


def test_fail_job_keeps_result(monkeypatch):
    _patch(monkeypatch)

    async def scenario():
        await dev_jobs.create_job("j2", {})
        await dev_jobs.fail_job("j2", "boom", result={"status": "error"})
        return await dev_jobs.get_job("j2")

    job = asyncio.run(scenario())
    assert job["status"] == dev_jobs.STATUS_FAILED
    assert job["error"] == "boom"
    assert job["result"] == {"status": "error"}


def test_get_missing_job(monkeypatch):
    _patch(monkeypatch)
    assert asyncio.run(dev_jobs.get_job("nope")) is None


def test_cancel_job(monkeypatch):
    _patch(monkeypatch)

    async def scenario():
        await dev_jobs.create_job("j3", {})
        changed = await dev_jobs.cancel_job("j3")
        job = await dev_jobs.get_job("j3")
        return changed, job

    changed, job = asyncio.run(scenario())
    assert changed is True
    assert job["status"] == dev_jobs.STATUS_CANCELLED
    assert job["error"] == "cancelled"


def test_cancel_job_terminal_is_noop(monkeypatch):
    _patch(monkeypatch)

    async def scenario():
        await dev_jobs.create_job("j4", {})
        await dev_jobs.complete_job("j4", {"status": "success"})
        changed = await dev_jobs.cancel_job("j4")
        return changed, await dev_jobs.get_job("j4")

    changed, job = asyncio.run(scenario())
    assert changed is False
    assert job["status"] == dev_jobs.STATUS_COMPLETED



================================================
FILE: tests/test_http_example.py
================================================
import asyncio
import os
import sys
from unittest.mock import Mock

import azure.functions as func

# Add src to Python path
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

from functions.http_example import main


def _make_req(**overrides):
    """Build a mock HttpRequest with the attributes the middleware reads."""
    req = Mock(spec=func.HttpRequest)
    req.params = {}
    req.method = "GET"
    req.url = "http://localhost/api/http_example"
    req.headers = {}
    req.get_json.return_value = None
    for key, value in overrides.items():
        setattr(req, key, value)
    return req


def _make_context():
    context = Mock(spec=func.Context)
    context.function_name = "http_example"
    context.invocation_id = "test-invocation"
    return context


def _call(req):
    """Invoke the async Azure handler synchronously for testing."""
    return asyncio.run(main(req, _make_context()))


def test_http_example_with_name():
    """Test HTTP function with name parameter"""
    req = _make_req(params={"name": "Azure"})

    response = _call(req)

    assert response.status_code == 200
    assert "Hello, Azure!" in response.get_body().decode()


def test_http_example_with_json_body():
    """Test HTTP function with JSON body"""
    req = _make_req()
    req.get_json.return_value = {"name": "World"}

    response = _call(req)

    assert response.status_code == 200
    assert "Hello, World!" in response.get_body().decode()


def test_http_example_no_name():
    """Test HTTP function without name parameter"""
    req = _make_req()
    req.get_json.side_effect = Exception("No JSON body")

    response = _call(req)

    assert response.status_code == 400
    assert "Please provide a name parameter" in response.get_body().decode()



================================================
FILE: tests/test_mcp_client.py
================================================
"""Tests for the REST-side MCP client connection manager.

These exercise the connection lifecycle, per-(session_id, tab_id) caching, idle
eviction, and reconnect-on-failure without a live MCP server by faking the
``streamablehttp_client`` transport and ``ClientSession``.

Written with ``asyncio.run`` rather than pytest-asyncio so they run under a
plain pytest install.
"""
import asyncio
import os
import sys
from contextlib import asynccontextmanager
from types import SimpleNamespace

import pytest

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

from agent import mcp_client  # noqa: E402
from core.config import settings  # noqa: E402
from core.exceptions import MCPException  # noqa: E402


class FakeSession:
    """Stand-in for mcp.ClientSession."""

    def __init__(self, *, fail_calls: int = 0):
        self.initialized = False
        self.call_count = 0
        self._fail_calls = fail_calls

    async def __aenter__(self):
        return self

    async def __aexit__(self, *exc):
        return False

    async def initialize(self):
        self.initialized = True

    async def call_tool(self, name, arguments):
        self.call_count += 1
        if self.call_count <= self._fail_calls:
            raise RuntimeError("transport dropped")
        return SimpleNamespace(
            isError=False,
            structuredContent={"status": "success", "echo": arguments},
            content=[SimpleNamespace(text="ok")],
        )


def _install_fakes(monkeypatch_target, sessions):
    """Patch transport + ClientSession to hand out the given fake sessions."""
    created = {"transport": 0, "session": 0}

    @asynccontextmanager
    async def fake_transport(url, headers):
        created["transport"] += 1
        yield ("read", "write", None)

    def fake_client_session(read, write, **kwargs):
        idx = created["session"]
        created["session"] += 1
        return sessions[idx]

    monkeypatch_target.setattr(mcp_client, "streamablehttp_client", fake_transport)
    monkeypatch_target.setattr(mcp_client, "ClientSession", fake_client_session)
    return created


@pytest.fixture(autouse=True)
def _configure_mcp(monkeypatch):
    monkeypatch.setattr(settings.mcp, "MCP_SERVER_URL", "https://mcp.test")
    monkeypatch.setattr(settings.mcp, "MCP_ENDPOINT_PATH", "/mcp")
    monkeypatch.setattr(settings.mcp, "MCP_CONNECT_TIMEOUT", 5)
    monkeypatch.setattr(settings.mcp, "MCP_TOOL_TIMEOUT", 5)
    monkeypatch.setattr(settings.mcp, "MCP_IDLE_TTL", 300)
    monkeypatch.setattr(settings.mcp, "MCP_MAX_CONNECTIONS", 10)


def test_call_tool_connects_and_returns_structured_data(monkeypatch):
    sessions = [FakeSession()]
    created = _install_fakes(monkeypatch, sessions)
    mgr = mcp_client.MCPConnectionManager()

    async def scenario():
        result = await mgr.call_tool(
            "sess1", "tabA", "send_message", {"x": 1}, {"Authorization": "Bearer t"}
        )
        await mgr.close_all()
        return result

    result = asyncio.run(scenario())
    assert result.is_error is False
    assert result.data == {"status": "success", "echo": {"x": 1}}
    assert created["transport"] == 1
    assert sessions[0].initialized is True


def test_same_key_reuses_connection(monkeypatch):
    sessions = [FakeSession()]
    created = _install_fakes(monkeypatch, sessions)
    mgr = mcp_client.MCPConnectionManager()

    async def scenario():
        await mgr.call_tool("s", "t", "send_message", {}, {})
        await mgr.call_tool("s", "t", "send_message", {}, {})
        await mgr.close_all()

    asyncio.run(scenario())
    # One transport/session for two calls on the same (session, tab).
    assert created["transport"] == 1
    assert sessions[0].call_count == 2


def test_distinct_keys_get_distinct_connections(monkeypatch):
    sessions = [FakeSession(), FakeSession()]
    created = _install_fakes(monkeypatch, sessions)
    mgr = mcp_client.MCPConnectionManager()

    async def scenario():
        await mgr.call_tool("s", "tabA", "send_message", {}, {})
        await mgr.call_tool("s", "tabB", "send_message", {}, {})
        await mgr.close_all()

    asyncio.run(scenario())
    assert created["transport"] == 2


def test_idle_connection_is_evicted(monkeypatch):
    sessions = [FakeSession(), FakeSession()]
    created = _install_fakes(monkeypatch, sessions)
    monkeypatch.setattr(settings.mcp, "MCP_IDLE_TTL", 0)
    mgr = mcp_client.MCPConnectionManager()

    async def scenario():
        await mgr.call_tool("s", "t", "send_message", {}, {})
        # TTL=0 means the first connection is stale on the next acquire.
        await mgr.call_tool("s", "t", "send_message", {}, {})
        await mgr.close_all()

    asyncio.run(scenario())
    assert created["transport"] == 2


def test_retries_once_on_transport_failure(monkeypatch):
    # First session fails its only call; manager reconnects and succeeds.
    sessions = [FakeSession(fail_calls=1), FakeSession()]
    created = _install_fakes(monkeypatch, sessions)
    mgr = mcp_client.MCPConnectionManager()

    async def scenario():
        result = await mgr.call_tool("s", "t", "send_message", {}, {})
        await mgr.close_all()
        return result

    result = asyncio.run(scenario())
    assert result.is_error is False
    assert created["transport"] == 2


def test_missing_server_url_raises(monkeypatch):
    monkeypatch.setattr(settings.mcp, "MCP_SERVER_URL", "")
    mgr = mcp_client.MCPConnectionManager()

    async def scenario():
        await mgr.call_tool("s", "t", "send_message", {}, {})

    with pytest.raises(MCPException):
        asyncio.run(scenario())



================================================
FILE: tests/test_send_message_status_api.py
================================================
import importlib

send_message_status_api = importlib.import_module(
    "functions.api.send_message_status.__init__"
)


def test_failed_job_response_has_single_canonical_error():
    job = {
        "job_id": "j1",
        "status": "failed",
        "error": "Root failure",
        "result": {
            "status": "error",
            "content": "Nested failure",
            "error": "Another nested error",
        },
    }

    success, status, data, error = send_message_status_api._prepare_status_payload(
        "j1", job
    )

    assert success is False
    assert status == "failed"
    assert error == "Root failure"
    assert data["status"] == "failed"
    assert "result" not in data
    assert "error" not in data


def test_failed_job_response_falls_back_to_result_content_when_error_missing():
    job = {
        "job_id": "j2",
        "status": "failed",
        "error": None,
        "result": {
            "status": "error",
            "content": "Only nested error",
        },
    }

    success, status, data, error = send_message_status_api._prepare_status_payload(
        "j2", job
    )

    assert success is False
    assert status == "failed"
    assert error == "Only nested error"
    assert data["status"] == "failed"
    assert "result" not in data


def test_non_failed_job_keeps_result_payload():
    job = {
        "job_id": "j3",
        "status": "completed",
        "error": None,
        "result": {
            "status": "success",
            "content": "ok",
        },
    }

    success, status, data, error = send_message_status_api._prepare_status_payload(
        "j3", job
    )

    assert success is True
    assert status == "completed"
    assert error is None
    assert data["status"] == "completed"
    assert data["result"] == {"status": "success", "content": "ok"}



================================================
FILE: tests/test_session_managers.py
================================================
import asyncio
import os
import sys

sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

from core.config import settings  # noqa: E402
from services.session_managers import SessionHistoryManager  # noqa: E402


class _InsertResult:
    def __init__(self, inserted_id):
        self.inserted_id = inserted_id


class _FakeChatCollection:
    def __init__(self):
        self.inserted = []

    async def insert_one(self, doc):
        self.inserted.append(doc)
        return _InsertResult("msg-1")


class _FakeContextCollection:
    def __init__(self, existing_context):
        self.existing_context = existing_context
        self.updates = []

    async def find_one(self, *_args, **_kwargs):
        return self.existing_context

    async def update_one(self, *_args, **_kwargs):
        self.updates.append({"args": _args, "kwargs": _kwargs})


def _set_required_collection_settings(monkeypatch):
    monkeypatch.setattr(
        settings.database,
        "DEV_CHAT_COLLECTION",
        "dev_chat_collection_test",
        raising=False,
    )
    monkeypatch.setattr(
        settings.database,
        "DEV_SESSION_CONTEXT",
        "dev_session_context_test",
        raising=False,
    )


def test_append_message_updates_title_to_latest_user_message_when_not_custom(
    monkeypatch,
):
    _set_required_collection_settings(monkeypatch)
    manager = SessionHistoryManager()
    fake_chat = _FakeChatCollection()
    fake_context = _FakeContextCollection(
        {
            "title": "Old title",
            "is_custom_title": False,
        }
    )

    monkeypatch.setattr(
        SessionHistoryManager,
        "chat_collection",
        property(lambda _self: fake_chat),
    )
    monkeypatch.setattr(
        SessionHistoryManager,
        "context_collection",
        property(lambda _self: fake_context),
    )

    from services import session_managers as sm  # noqa: WPS433

    async def _noop_init():
        return None

    monkeypatch.setattr(sm.db_manager, "initialize", _noop_init)

    asyncio.run(
        manager.append_message(
            workspace_id="ws1",
            user_id="u1",
            session_id="s1",
            role="user",
            content="My latest user message",
        )
    )

    set_update = fake_context.updates[0]["args"][1]["$set"]
    assert set_update["title"] == "My latest user message"
    assert set_update["is_custom_title"] is False


def test_append_message_does_not_override_custom_title(monkeypatch):
    _set_required_collection_settings(monkeypatch)
    manager = SessionHistoryManager()
    fake_chat = _FakeChatCollection()
    fake_context = _FakeContextCollection(
        {
            "title": "User custom title",
            "is_custom_title": True,
        }
    )

    monkeypatch.setattr(
        SessionHistoryManager,
        "chat_collection",
        property(lambda _self: fake_chat),
    )
    monkeypatch.setattr(
        SessionHistoryManager,
        "context_collection",
        property(lambda _self: fake_context),
    )

    from services import session_managers as sm  # noqa: WPS433

    async def _noop_init():
        return None

    monkeypatch.setattr(sm.db_manager, "initialize", _noop_init)

    asyncio.run(
        manager.append_message(
            workspace_id="ws1",
            user_id="u1",
            session_id="s1",
            role="user",
            content="Newest message should not replace custom title",
        )
    )

    set_update = fake_context.updates[0]["args"][1]["$set"]
    assert "title" not in set_update
    assert "is_custom_title" not in set_update



================================================
FILE: tests/test_worker_poison_handlers.py
================================================
import asyncio
import json

import function_app


class _MockQueueMessage:
    def __init__(self, body: str):
        self._body = body.encode("utf-8")

    def get_body(self):
        return self._body


def test_send_message_worker_poison_marks_job_failed(monkeypatch):
    called = {}

    async def _fake_fail_job(job_id, error, result=None):
        called["job_id"] = job_id
        called["error"] = error
        called["result"] = result

    monkeypatch.setattr("services.dev_jobs.fail_job", _fake_fail_job)

    msg = _MockQueueMessage(json.dumps({"job_id": "job-send-1"}))
    asyncio.run(function_app.send_message_worker_poison(msg))

    assert called["job_id"] == "job-send-1"
    assert (
        called["error"] == "Message moved to dead-letter queue after retry exhaustion"
    )
    assert called["result"] is None


def test_update_design_hld_worker_poison_marks_job_failed(monkeypatch):
    called = {}

    async def _fake_fail_job(job_id, error, result=None):
        called["job_id"] = job_id
        called["error"] = error
        called["result"] = result

    monkeypatch.setattr("services.dev_jobs.fail_job", _fake_fail_job)

    msg = _MockQueueMessage(json.dumps({"job_id": "job-hld-1"}))
    asyncio.run(function_app.update_design_hld_worker_poison(msg))

    assert called["job_id"] == "job-hld-1"
    assert (
        called["error"] == "Message moved to dead-letter queue after retry exhaustion"
    )
    assert called["result"] is None


def test_poison_handlers_ignore_invalid_json(monkeypatch):
    called = {"count": 0}

    async def _fake_fail_job(job_id, error, result=None):
        called["count"] += 1

    monkeypatch.setattr("services.dev_jobs.fail_job", _fake_fail_job)

    bad = _MockQueueMessage("not-json")
    asyncio.run(function_app.send_message_worker_poison(bad))
    asyncio.run(function_app.update_design_hld_worker_poison(bad))

    assert called["count"] == 0



================================================
FILE: .github/instructions/01-personal-to-feature.md
================================================
# Phase 1: Personal Branch → Feature Branch

## Purpose

This phase is used when an individual developer is contributing code into the shared feature branch.

Flow:

feature/<developer>-<task>
    →
feature

## Rules

- Developers work only in their own branch.
- Direct commits are allowed only in personal branches.
- Rebase before merging.
- Never overwrite teammate changes.
- Keep commits logically grouped.
- Squash commits before merge if possible.

## Required Workflow

Start from latest feature:

```bash
git checkout feature
git pull origin feature

git checkout -b feature/<developer>-<task>
```

Work normally:

```bash
git add .
git commit -m "[TICKET] Description"
git push origin feature/<developer>-<task>
```

Before merging:

```bash
git fetch origin
git rebase origin/feature
```

If rebase was performed:

```bash
git push --force-with-lease
```

Merge through PR or approved merge process.

## AI Agent Rules

ALWAYS:

- Sync with latest feature branch first.
- Prefer rebase over merge.
- Preserve teammate changes.
- Use force-with-lease only on personal branches.

NEVER:

- Force push feature branch.
- Rewrite feature branch history.
- Commit directly to feature branch.


================================================
FILE: .github/instructions/02-feature-to-dev.md
================================================
# Phase 2: Feature Branch → Dev Branch

## Purpose

Promote integrated development work into the development environment.

Flow:

feature
    →
dev

## Strategy

FULL MERGE

Everything in feature should move to dev.

No filtering occurs here.

## Required Workflow

```bash
git checkout dev
git pull origin dev

git merge origin/feature
git push origin dev
```

## Commit Requirements

Before reaching dev:

- One logical feature = one logical commit.
- Squash noisy commits.
- Remove temporary commits.
- Use ticket IDs.

Example:

[TICKET-123] Add Login Validation

## AI Agent Rules

ALWAYS:

- Use full merge.
- Preserve complete feature history.
- Ensure dev receives all feature changes.

NEVER:

- Cherry-pick from feature to dev.
- Selectively move files.
- Rewrite dev history.
- Force push dev.


================================================
FILE: .github/instructions/03-dev-to-stage.md
================================================
# Phase 3: Dev Branch → Stage Branch

## Purpose

Promote only approved functionality to UAT / Stage.

Flow:

dev
    →
stage

## Strategy

CHERRY-PICK ONLY

Stage must contain only approved commits.

## Required Workflow

Find approved commit:

```bash
git log origin/dev --oneline
```

Move to stage:

```bash
git checkout stage
git pull origin stage

git cherry-pick -x <commit-hash>
git push origin stage
```

## Approval Requirement

Only approved tickets may move from dev to stage.

Unapproved work remains in dev.

## Conflict Resolution

If conflict occurs:

```bash
git add .
git cherry-pick --continue
```

Abort if required:

```bash
git cherry-pick --abort
```

## AI Agent Rules

ALWAYS:

- Use cherry-pick -x.
- Verify approval before promotion.
- Keep traceability.

NEVER:

- Merge dev into stage.
- Force push stage.
- Reset stage history.
- Cherry-pick without -x.


================================================
FILE: .github/instructions/04-stage-to-prod.md
================================================
# Phase 4: Stage Branch → Production Branch

## Purpose

Promote tested and approved releases into production.

Flow:

stage
    →
production

## Strategy

CHERRY-PICK ONLY

Production receives approved and tested commits only.

## Required Workflow

```bash
git checkout production
git pull origin production

git cherry-pick -x <stage-commit>
git push origin production
```

## Release Controls

- Production approval required.
- Only tested stage commits may be promoted.
- Tag every release.

Example:

```bash
git tag prod-YYYY-MM-DD
git push --tags
```

## Rollback

Use:

```bash
git revert <commit>
```

Never use:

```bash
git reset --hard
```

## Hotfixes

If a production hotfix is created:

- Release to production.
- Back-propagate to stage.
- Back-propagate to dev.

## AI Agent Rules

ALWAYS:

- Use cherry-pick -x.
- Preserve production history.
- Tag releases.
- Use revert for rollback.

NEVER:

- Merge stage into production.
- Force push production.
- Rewrite production history.
- Use reset --hard.


================================================
FILE: .github/instructions/git_routing.md
================================================
# Git Workflow Routing

If working on a personal branch and moving code to feature:
Read -> 01-personal-to-feature.md

If promoting feature to dev:
Read -> 02-feature-to-dev.md

If promoting dev to stage:
Read -> 03-dev-to-stage.md

If promoting stage to production:
Read -> 04-stage-to-prod.md


================================================
FILE: .github/workflows/function-deployment.yaml
================================================
name: Deploy Dev Rest Function (DEV)

on:
  workflow_dispatch:
  push:
    branches:
      - V2_dev
    paths:
      - 'src/**'
      - 'function_app.py'
      - 'host.json'
      - 'requirements.txt'
      - '.github/workflows/function-deployment.yaml'

env:
  FUNCTION_APP_NAME: arch-rest-dev
  FUNCTION_PATH: .
  rg_name: ${{ secrets.DEV_RG_NAME }}

jobs:
  deploy-function:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Build Python dependencies for Azure Functions
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt --target=".python_packages/lib/site-packages"

    - name: Deploy Azure Function
      uses: Azure/functions-action@v1
      with:
        app-name: ${{ env.FUNCTION_APP_NAME }}
        package: ${{ env.FUNCTION_PATH }}
        scm-do-build-during-deployment: false
        enable-oryx-build: false
        respect-funcignore: true
        # remote-build: true

    - name: Restart Function App
      run: |
        az functionapp restart \
          --name ${{ env.FUNCTION_APP_NAME }} \
          --resource-group ${{ env.rg_name }}



================================================
FILE: .github/workflows/python-da-dev-codedev-agents.yaml
================================================
name: Deploy-BA deep agent deploy to poly app

on:
  workflow_dispatch: 
  push:
    branches:
      - Dev-DeepAgent
    # paths:    
    #   - 'Dockerfile'
    #   - '.github/workflows/python-da-dev-codedev-agents.yaml'
   
env:
  AZURE_WEBAPP_NAME: forgex-dev-da-codedev                     
  REGISTRY: ${{ secrets.registry }}
  DOCKER_IMAGE_NAME: dev-da-codedev
  rg_name: ${{ secrets.DEV_RG_NAME }}
  subscription_id: ${{ secrets.subscription_id }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Log in to Azure Container Registry
      run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ${{ env.REGISTRY }} -u ${{ secrets.REGISTRY_USERNAME }} --password-stdin
   
    - name: Enable System Assigned Identity
      run: |
        az webapp identity assign \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }}

    - name: Assign AcrPull Role
      run: |
          PRINCIPAL_ID=$(az webapp identity show \
            --name ${{ env.AZURE_WEBAPP_NAME }} \
            --resource-group ${{ env.rg_name }} \
            --query principalId -o tsv)

          ACR_ID=$(az acr show \
            --name forgeXContainer \
            --resource-group forgeXdev \
            --query id -o tsv)

          az role assignment create \
            --assignee $PRINCIPAL_ID \
            --role AcrPull \
            --scope $ACR_ID || true
         
    # - name: Build and Push Docker Image
    #   run: |
    #    docker build --no-cache -t ${{ env.REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }} .
    #    docker push ${{ env.REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }}  
    - name: Build and Push Docker Image
      run: |
        docker build \
          --no-cache \
          --build-arg GH_PAT_READ=${{ secrets.GH_PAT_READ }} \
          -t ${{ env.REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }} .
        docker push ${{ env.REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }}

        
    - name: Save Previous Image Tag
      id: previous
      run: |
        az webapp config show \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }} \
          --query linuxFxVersion \
          --output tsv > previous_image.txt

    - name: Set LinuxFxVersion to New Image
      run: |
        az webapp config set \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }} \
          --linux-fx-version "DOCKER|${{ env.REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }}"

    - name: Enable ACR Access via Managed Identity
      run: |
        az resource update \
          --ids /subscriptions/${{ env.subscription_id }}/resourceGroups/${{ env.rg_name }}/providers/Microsoft.Web/sites/${{ env.AZURE_WEBAPP_NAME }}/config/web \
          --set properties.acrUseManagedIdentityCreds=true

    - name: Set Health Check Path
      run: |
        az webapp update \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }} \
          --set siteConfig.healthCheckPath="/health"

    - name: Restart App Service
      run: |
        az webapp restart \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }}


    - name: Rollback if Deployment Fails
      if: failure()
      run: |
        PREV_IMAGE=$(cat previous_image.txt)
        az webapp config set \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }} \
          --linux-fx-version "$PREV_IMAGE"
        az webapp restart \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }}

    - name: Verify deployed image
      run: |
        DEPLOYED_IMAGE=$(az webapp config show \
          --name ${{ env.AZURE_WEBAPP_NAME }} \
          --resource-group ${{ env.rg_name }} \
          --query linuxFxVersion \
          --output tsv)

        echo "Deployed image: $DEPLOYED_IMAGE"

        EXPECTED_IMAGE="DOCKER|${{ env.REGISTRY }}/${{ env.DOCKER_IMAGE_NAME }}:${{ github.sha }}"

        if [ "$DEPLOYED_IMAGE" != "$EXPECTED_IMAGE" ]; then
          echo "❌ Deployment failed or image mismatch!"
          exit 1
        else
          echo "✅ Image deployed successfully."
        fi
    
    - name: Cleanup Old ACR Images
      run: |
        ACR_NAME="forgeXContainer"
        REPOSITORY="${{ env.DOCKER_IMAGE_NAME }}"

        echo "Repository: $REPOSITORY"

        TAGS=$(az acr repository show-tags \
          --name $ACR_NAME \
          --repository $REPOSITORY \
          --orderby time_desc \
          -o tsv)

        COUNT=0

        for TAG in $TAGS
        do
          COUNT=$((COUNT + 1))

          if [ $COUNT -le 2 ]; then
            echo "Keeping: $TAG"
          else
            echo "Deleting: $TAG"

            az acr repository delete \
              --name $ACR_NAME \
              --image ${REPOSITORY}:$TAG \
              --yes
          fi
        done


