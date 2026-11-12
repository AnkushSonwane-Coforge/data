# CodeInsightAI — Reverse Engineering Backend

> Automatically generate Business Requirements Documents (BRD) and Technical Specification Documents (TSD) from source code using LLM-powered analysis.

CodeInsightAI analyses uploaded codebases across 20+ programming languages (COBOL, JCL, BMS, C, C++, C#, Java, JavaScript, Python, SQL, Fortran, PL/I, RPG, ColdFusion, JSP, JDE Event Rules, XML/DFDL, Assembly, and more) and produces structured documentation through four distinct generation pipelines — one based on Abstract Syntax Tree (AST) traversal, one based on iterative LLM expansion, one agentic pipeline built on DeepAgents that drives an orchestrator + specialised subagents, and CodeSense: a multi-language static analysis and knowledge graph pipeline.

---

## Table of Contents

- [Features](#features)
- [Architecture Overview](#architecture-overview)
- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Environment Variables](#environment-variables)
- [Running the Application](#running-the-application)
- [API Reference](#api-reference)
- [Document Generation Pipelines](#document-generation-pipelines)
- [Supported Providers](#supported-providers)
- [Configuration Reference](#configuration-reference)
- [Deployment](#deployment)

---

## Features

- **Four generation pipelines** — AST traversal for deep code analysis; iterative expansion for template-driven refinement; an agentic DeepAgents pipeline for conversational, multi-document generation and editing; and CodeSense for multi-language static analysis, cross-language call graphs, and graph-anchored documentation
- **CodeSense static analysis & knowledge graph** — parses 20+ legacy and modern languages, computes cyclomatic complexity and token metrics, extracts dependencies, builds cross-language call graphs, detects dead code, and generates interactive standalone HTML reports
- **Interactive graph exploration & AI assistant** — visualizes call and dependency graphs with BFS traversal, entrypoint detection, subsystem clustering, dependency matrices, and AI-powered node explanation / contextual Q&A
- **Multi-language BRD & TSD generation (CodeSense)** — generates structured section-by-section JSON documents with multi-retriever RAG, quality scoring, gap-filling validation, section-level human feedback review, and DOCX compilation
- **Graph-anchored BRD generation** — anchors requirements generation to specific execution entry points or plain-English functionality queries via knowledge graph traversal
- **Agentic pipeline (DeepAgents)** — an orchestrator coordinates code-analyzer, doc-writer, quality-checker, doc-formatter, and doc-editor subagents; supports chat-driven generation, targeted document edits, human-in-the-loop plan approval, document versioning, and real-time SSE streaming
- **Multi-language code parsing** — C#, VB.NET, COBOL, Python, Java, C/C++, JavaScript, JCL, BMS, PL/I, RPG, Fortran, SQL, XML/DFDL, Assembly, ColdFusion, JSP, JDE ER, Sabretalk
- **Multiple document types** — BRD, Application TSD, Program-level TSD, Application Insights, Standalone HTML Code Analysis Reports
- **Pluggable LLM providers** — Azure OpenAI, Google Gemini, AWS Bedrock (Claude), OpenAI
- **Pluggable vector stores** — Azure AI Search, AWS OpenSearch, Chroma
- **Pluggable databases** — SQL Server (Azure & AWS RDS), PostgreSQL, MySQL
- **Pluggable cloud storage** — Azure Blob Storage, AWS S3
- **Mermaid diagram generation & refinement** — automatic flowcharts embedded in documents
- **Background processing** — all document and report generation runs in async threads / background tasks
- **Template support** — user-supplied PDF and Markdown templates guide document structure
- **RAG-powered context retrieval** — embeddings stored in vector DB for relevant code retrieval during generation
- **Full observability with Langfuse** — distributed tracing, node execution trees, token telemetry, and performance reports

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                  FastAPI Application                                   │
│                                                                                        │
│  ┌───────────┐  ┌───────────┐  ┌────────────────┐  ┌────────┐  ┌─────────────────────┐ │
│  │  /common  │  │/embeddings│  │/doc_generation_│  │ /agent │  │     /codesense      │ │
│  │/iterations│  │           │  │* (traversal &  │  │ (Deep  │  │/codesense_app_brd/tsd│ │
│  │           │  │           │  │  iterative)    │  │ Agents)│  │ /html_report /graph │ │
│  └───────────┘  └───────────┘  └────────────────┘  └────────┘  └─────────────────────┘ │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │
                             ┌──────────────┼──────────────┐
                             │              │              │
                       ┌─────▼──────┐  ┌────▼────┐  ┌────▼──────┐
                       │   models/  │  │  llm/   │  │   rag/    │
                       │ (DB layer) │  │(LLM hub)│  │(retrieval)│
                       └─────┬──────┘  └────┬────┘  └─────┬─────┘
                             │              │             │
     ┌───────────────────────▼──────────────▼─────────────▼────────────────────────┐
     │                                 pipelines/                                  │
     │  ┌──────────────┐ ┌─────────────────┐ ┌───────────┐ ┌─────────────────────┐ │
     │  │  traversal/  │ │iterative_expan..│ │   agent/  │ │     code_sense/     │ │
     │  │  (AST-based) │ │ (template-based)│ │(DeepAgents│ │ (Static Analysis &  │ │
     │  └──────────────┘ └─────────────────┘ │ orch. +   │ │  Knowledge Graph &  │ │
     │         shared/ (convert2word, h2d, …)│ subagents)│ │  Multi-Lang DocGen) │ │
     │                                       └───────────┘ └─────────────────────┘ │
     └─────────────────────────────────────────────────────────────────────────────┘
                             │              │             │
                       ┌─────▼────┐  ┌──────▼──┐  ┌──────▼───┐
                       │ Database │  │  Cloud  │  │  Vector  │
                       │(SQL/PG/  │  │ Storage │  │   Store  │
                       │  MySQL)  │  │(Az/S3)  │  │(Az/OS/Ch)│
                       └──────────┘  └─────────┘  └──────────┘
```

---

## Project Structure

```
ReverseEngineering/
│
├── main.py                          # FastAPI app entry point, CORS, router registration
├── config.py                        # Centralised environment configuration
├── db.py                            # Database initialisation on startup
│
├── api/
│   └── routes/
│       ├── common.py                # ZIP upload, template management, file paths
│       ├── embeddings.py            # Embedding CRUD endpoints
│       ├── iterations.py            # Iteration management endpoints
│       ├── doc_generation_traversal.py   # BRD/TSD via traversal pipeline
│       ├── doc_generation_iterative.py   # BRD/TSD/Insights via iterative pipeline
│       ├── ml_analysis_reports.py   # CodeSense status, HTML reports, section viewer, BRD/TSD routes
│       ├── graph.py                 # Knowledge graph, D3 JSON, BFS traversal, AI explain/chat
│       ├── graph_anchored_brd.py    # Subgraph & entry-point anchored BRD generation
│       ├── doc_chat.py              # Conversational document query endpoints
│       └── analytics.py             # Telemetry & performance analytics
│
├── llm/                             # LLM integration (project-wide)
│   ├── __init__.py                  # Exports initialize_llm, handle_embedding_function
│   ├── llm_calls.py                 # LLM invocation, retry/continuation logic
│   ├── handler_embedding.py         # Chunk code and push to vector store
│   └── prompts_template/
│       └── prompts.py               # All system & user prompt templates
│
├── models/
│   ├── models.py                    # Primary DB models (SQL Server / PostgreSQL)
│   └── models_mysql.py              # MySQL-specific models
│
├── pipelines/
│   ├── code_sense/                  # CodeSense static analysis, graph & multi-lang doc-gen
│   │   ├── ingestion/               # Multi-language static code analyzers & workflows
│   │   │   ├── code_analysis.py     # Main LangGraph analysis workflow (parsing → graph → summary)
│   │   │   ├── call_stack_resolver.py # Resolves intra & cross-language call chains
│   │   │   ├── mainframe_reports.py # Specialized reporting for mainframe/legacy systems
│   │   │   ├── analyzers/           # 20+ specialized language analyzers (COBOL, JCL, Java, C#, etc.)
│   │   │   └── preprocessors/       # COBOL & RPG preprocessors (copybook expansion)
│   │   │
│   │   ├── graph/                   # Knowledge graph construction & interactive services
│   │   │   ├── graph_builder.py     # Builds cross-language call graphs and UI node/edge models
│   │   │   ├── graph_service.py     # BFS traversal, dependency matrix, filtering, node source
│   │   │   ├── graph_ai_summary.py  # AI node explanation, node chat Q&A, question suggestion
│   │   │   ├── entrypoint_detector.py # True execution entrypoint identification & pruning
│   │   │   ├── subsystem_detector.py  # Automated clustering into logical functional subsystems
│   │   │   ├── cross_lang_linker.py # Bridges C++ APIs, XML/DFDL schemas, SQL, and backend code
│   │   │   ├── kg_store.py          # Knowledge graph persistence
│   │   │   └── generic_kuzu_interface.py # Embedded Kùzu graph database interface
│   │   │
│   │   ├── doc_generation/          # Multi-language BRD & TSD generation
│   │   │   ├── app_brd_multi_language.py # Multi-language BRD generator (RAG + design summary)
│   │   │   ├── app_tsd_multi_language.py # Multi-language TSD generator (RAG + design summary)
│   │   │   ├── generate_brd_template.py # BRD template schema and structure generators
│   │   │   ├── generate_tsd_template.py # TSD template schema and structure generators
│   │   │   ├── llm_validation.py    # Multi-step section output validation & gap-fill
│   │   │   └── section_feedback.py  # Section-level feedback and targeted re-generation
│   │   │
│   │   ├── report_generators/       # Standalone interactive HTML report generator
│   │   │   ├── html_report_generator.py # Assembles self-contained HTML analysis reports
│   │   │   ├── content_processor.py # Processes section markdown, tables, and code blocks
│   │   │   ├── mermaid_processor.py # Renders embedded Mermaid architecture diagrams
│   │   │   ├── table_download_utils.py # Injects Excel/CSV export capabilities into HTML tables
│   │   │   └── templates/           # Domain templates (mainframe, web_api, database, etc.)
│   │   │
│   │   └── trace_analysis.py        # Langfuse trace telemetry processing for CodeSense
│   │
│   ├── traversal/                   # AST-based code analysis pipeline
│   │   ├── ingestion/
│   │   │   ├── csharp_parser/       # C# class, interface, dependency parsing
│   │   │   ├── vbnet_parser/        # VB.NET class, interface, dependency parsing
│   │   │   ├── cobol_parser/        # COBOL program parsing
│   │   │   ├── language_handler/    # ASP.NET multi-language routing
│   │   │   ├── handler.py           # Build structured JSON from parsed AST
│   │   │   ├── ingestion_handler.py # Orchestrates ZIP → JSON pipeline
│   │   │   └── zip_handler.py       # ZIP extraction & folder traversal
│   │   └── doc_generation/
│   │       ├── outline_gent.py      # Generate initial BRD/TSD outline
│   │       ├── final_doc_gent.py    # Expand outline into full document
│   │       ├── dependency_extraction.py
│   │       ├── asked_file_processing_asp_net.py
│   │       ├── asked_file_processing_cobol.py
│   │       ├── prompts_brd/         # Tech-stack-specific prompts
│   │       └── utils/
│   │
│   ├── iterative_expansion/         # Template-driven iterative pipeline
│   │   └── doc_generation/
│   │       ├── brd_iterative.py     # Business Requirements Document
│   │       ├── app_tsd_iterative.py # Application Technical Specification
│   │       ├── prog_tsd_iterative.py# Program-level Technical Specification
│   │       ├── app_insights.py      # Application Insights document
│   │       └── toc_generation.py    # Table of contents generation
│   │
│   ├── agent/                       # Agentic pipeline (DeepAgents + LangGraph)
│   │   ├── builder.py               # Builds the deep agent (orchestrator + subagents) & selects backend
│   │   ├── prompts.py               # Orchestrator + subagent system prompts
│   │   ├── tools.py                 # Custom tools (request_plan_approval HITL interrupt)
│   │   ├── routes.py                # /agent/* FastAPI routes (run, stream, versions, sessions, templates)
│   │   ├── streaming.py             # SSE event stream + plan-approval resume coordinator
│   │   ├── version_store.py         # Document/version/session/task persistence
│   │   ├── db_init.py               # Creates agent tables on startup; orphans running tasks
│   │   ├── memory_templates.py      # Canonical SKILL.md / AGENTS.md seed content
│   │   ├── azure_blob_backend/      # Azure Blob storage backend for agent workspace
│   │   └── aws_s3_backend/          # AWS S3 storage backend for agent workspace
│   │
│   └── shared/                      # Shared doc-gen utilities
│       ├── convert2word.py          # JSON/HTML → Word (.docx)
│       ├── h2d.py                   # HTML → python-docx parser
│       ├── json2markdown.py         # JSON → numbered Markdown
│       └── mermaid_refinement.py    # Find & refine Mermaid diagrams in JSON
│
├── rag.py                           # Vector store operations & retriever factory
│
├── utils/
│   ├── common_utils.py              # Text splitting, PDF parsing, HTML utilities
│   ├── cloud_storage_utils.py       # Azure Blob / AWS S3 upload & download
│   ├── database_factory.py          # Database connection factory
│   ├── embedding_factory.py         # Embedding provider factory
│   ├── vector_store_factory.py      # Vector DB factory
│   ├── aws_opensearch_utils.py      # OpenSearch indexing & deletion
│   ├── azure_search_utils.py        # Azure AI Search indexing & deletion
│   ├── doc_gen_iterative_utils.py   # Markdown/HTML helpers for iterative pipeline
│   ├── logging_config.py            # Centralised logging setup
│   └── parser_utils.py              # AST name extraction helpers
│
├── components/
│   ├── custom_output_parsers/       # LLM output fixing parser
│   └── custom_text_splitters/       # Language-aware code splitters
│
├── templates/                       # Static document templates
├── Dockerfile
├── requirements.txt
├── pyproject.toml
└── .env.local / .env.development / .env.production
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **API Framework** | FastAPI, Uvicorn |
| **LLM Providers** | Azure OpenAI, Google Gemini, AWS Bedrock (Claude), OpenAI |
| **LLM Orchestration** | LangChain (core, openai, google-genai, aws, community), LangGraph |
| **Agentic Framework** | DeepAgents, LangGraph (orchestrator + subagents, checkpointing, HITL interrupts) |
| **Static Code Analysis** | 20+ specialized language analyzers, AST parsing, regex tokenizers, preprocessors |
| **Graph & Visualization** | Kùzu Graph DB, NetworkX, D3.js (interactive UI graph), Graphviz |
| **Observability** | Langfuse (distributed tracing, node execution trees, telemetry reports) |
| **Vector Stores** | Azure AI Search, AWS OpenSearch, Chroma |
| **Embeddings** | Google Generative AI, Azure OpenAI, OpenAI, AWS Bedrock (Titan) |
| **Databases** | SQL Server, PostgreSQL, MySQL (Azure & AWS RDS) |
| **Cloud Storage** | Azure Blob Storage, AWS S3 |
| **Document Output** | python-docx, xhtml2pdf, reportlab, markdown2 |
| **HTML & Data Processing** | BeautifulSoup4, pandas, pydantic |
| **Infrastructure** | Docker, AWS Elastic Beanstalk, Azure App Service |
| **HTML Processing** | BeautifulSoup4 |
| **Data** | pandas, networkx |
| **Infrastructure** | Docker, AWS Elastic Beanstalk, Azure App Service |

---

## Getting Started

### Prerequisites

- Python **3.11+**
- An ODBC driver if using SQL Server (`ODBC Driver 18 for SQL Server`)
- Access to at least one LLM provider (Azure OpenAI / Gemini / AWS Bedrock)
- Access to at least one vector store (Azure AI Search / AWS OpenSearch / Chroma)
- Access to at least one database (SQL Server / PostgreSQL / MySQL)

### Installation

```bash
# Clone the repository
git clone <repository-url>
cd ReverseEngineering

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate        # Linux/macOS
venv\Scripts\activate           # Windows

# Install dependencies
pip install -r requirements.txt
```

### Environment Variables

Copy the appropriate environment file and fill in your values:

```bash
cp .env.local .env
```

#### Core Application

```env
ENV=local                          # local | development | production
APP_VERSION=2.0.4
APPLICATION_BACKEND_URL=http://localhost:8000
FRONTEND_URL_LOCAL=http://localhost:3000
```

#### LLM Provider (choose one)

```env
LLM_OPTION_NAME=gemini              # gemini | openai | openai-litellm | aws-bedrock-claude | aws-bedrock-claude-sso
AGENT_LLM_MODEL=gemini2             # gemini2 | openai-litellm_2 (agent doc synthesis)
TOOL_LLM_OPTION=openai              # openai | gemini3 | openai-litellm_3 (fast tool calls)

# Azure OpenAI
OPENAI_LLM_DEPLOYMENT_NAME=...
OPENAI_LLM_VERSION=2023-05-15
OPENAI_LLM_ENDPOINT=...
OPENAI_LLM_KEY=...

# OpenAI LiteLLM Proxy (supports openai-litellm, openai-litellm_2, openai-litellm_3)
OPENAI_LiteLLM_LLM_ENDPOINT=https://<litellm-proxy>/v1
OPENAI_LiteLLM_LLM_KEY=sk-...
OPENAI_LiteLLM_LLM_DEPLOYMENT_NAME=gpt-4o           # for openai-litellm (primary)
OPENAI_LiteLLM_LLM_DEPLOYMENT_NAME_2=gpt-4o         # for openai-litellm_2 (agent)
OPENAI_LiteLLM_LLM_DEPLOYMENT_NAME_3=gpt-4o-mini    # for openai-litellm_3 (tools)

# Google Gemini (supports gemini, gemini2, gemini3)
GEMINI_API_KEY=...
GEMINI_MODEL_NAME=gemini-2.0-flash                  # for gemini (primary)
GEMINI_MODEL_NAME_2=gemini-3.5-flash                # for gemini2 (agent)
GEMINI_MODEL_NAME_3=gemini-2.0-flash                # for gemini3 (tools)

# AWS Bedrock
AWS_BEDROCK_MODEL_ID=anthropic.claude-3-5-sonnet-20241022-v2:0
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1
```

#### Database (choose one)

```env
DATABASE_PROVIDER=mssql             # mssql | mysql | postgresql | mssql-aws-rds

# SQL Server
SQL_SERVER=...
SQL_DATABASE=...
SQL_USER=...
SQL_PASSWORD=...
SQL_ODBC_DRIVER=ODBC Driver 18 for SQL Server

# PostgreSQL
POSTGRES_HOST=...
POSTGRES_PORT=5432
POSTGRES_DATABASE=...
POSTGRES_USER=...
POSTGRES_PASSWORD=...

# MySQL (AWS RDS)
RDS_HOST=...
RDS_PORT=3306
RDS_USER=...
RDS_PASSWORD=...
RDS_DB_NAME=...
```

#### Cloud Storage (choose one)

```env
CLOUD_STORAGE_PROVIDER=azure        # azure | aws

# Azure Blob Storage
AZURE_STORAGE_CONNECTION_STRING=...
AZURE_STORAGE_ACCOUNT_NAME=...
AZURE_STORAGE_ACCOUNT_KEY=...
AZURE_CONTAINER_CODEBASE=codebase
AZURE_CONTAINER_OUTPUT=output
AZURE_CONTAINER_EMBEDDING=embedding
AZURE_CONTAINER_TEMPLATE=templates

# AWS S3
AWS_S3_BUCKET_NAME=...

```

#### Vector Store (choose one)

```env
VECTOR_STORE_PROVIDER=azure_search  # azure_search | chroma | aws_opensearch

# Azure AI Search
SERVICE_NAME=...
AZURE_SEARCH_API_KEY=...

# AWS OpenSearch
AWS_OPENSEARCH_ENDPOINT=...
AWS_OPENSEARCH_USERNAME=...
AWS_OPENSEARCH_PASSWORD=...

# Chroma (local — no additional config required)
```

#### Embedding Provider (choose one)

```env
EMBEDDING_PROVIDER=google           # google | azure | openai | aws_embedding | openai-litellm

GOOGLE_API_KEY=...
GOOGLE_EMBEDDING_MODEL=models/embedding-001

# LiteLLM Proxy
# OPENAI_LiteLLM_EMBEDDING_DEPLOYMENT_NAME=CodeinsightAI-dev-workspace-embedding
# OPENAI_LiteLLM_EMBEDDING_ENDPOINT=https://<litellm-host>/v1
# OPENAI_LiteLLM_EMBEDDING_KEY=sk-...
```

#### RAG Tuning

```env
SIZE_OF_CODE_CHUNK=20000            # Characters per code chunk
THRESHOLD_SIZE_CHUNK=2000           # Minimum chunk size before splitting
NO_OF_CHUNKS=5                      # Chunks to retrieve per query
SCORE_THRESHOLD=0.8                 # Minimum similarity score
RETRIEVER_K=5                       # Top-k documents returned
```

#### Agentic Pipeline (DeepAgents)

```env
AGENT_WORKSPACES_ROOT=./workspaces  # Local root for agent workspaces (used in local mode)
AGENT_LLM_MODEL=                    # Optional LLM override for the agent; falls back to LLM_OPTION_NAME

# The agent's workspace backend is selected by CLOUD_STORAGE_PROVIDER (see Cloud Storage above):
#   azure   → Azure Blob Storage (uses CODEBASE_CONTAINER as the agent file system)
#   s3/aws  → AWS S3
#   (unset) → local filesystem under AGENT_WORKSPACES_ROOT
```

> The agentic pipeline creates its own database tables on startup (`agent_document`,
> `agent_document_version`, `agent_session`, `agent_task`, `module_pipeline_type`, …).
> Any running tasks are marked **orphaned** on restart so the UI can prompt a re-run.

---

## Running the Application

```bash
# Development (with auto-reload)
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

Interactive API docs: `http://localhost:8000/docs`

---

## API Reference

### General

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/version` | Returns app version and environment |
| `POST` | `/upload_zip` | Upload codebase ZIP to cloud storage |
| `GET` | `/default_prompt_templates/{document_id}/{tech_stack_id}` | Get default prompt templates with signed URLs |
| `POST` | `/get_file_path` | Get signed URL for a generated document |

### Embeddings

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/create_embedding` | Create vector embeddings for a codebase |
| `GET` | `/get_embeddings` | List all embeddings |
| `GET` | `/embeddings/{embedding_id}` | Get embedding details |
| `DELETE` | `/embeddings/{embedding_id}` | Delete embedding from DB and vector store |

### Iterations

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/create_iteration` | Create a new iteration with prompts and templates |
| `GET` | `/get_iterations/{module_id}` | List iterations for a module |
| `GET` | `/get_iteration_by_id/{iteration_id}` | Get iteration details with template URLs |
| `PUT` | `/update_iteration` | Update iteration prompts/templates |

### Document Generation — Traversal Pipeline

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/generate_brd` | Generate BRD via AST traversal |
| `POST` | `/generate_tsd` | Generate TSD via AST traversal |

### Document Generation — Iterative Expansion Pipeline

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/generate_brd_iterative_expansion` | Generate BRD via iterative LLM expansion |
| `POST` | `/generate_tsd_iterative_expansion` | Generate Application TSD |
| `POST` | `/generate_program_tsd_iterative_expansion` | Generate Program-level TSD |
| `POST` | `/generate_application_insights` | Generate Application Insights document |

All document generation endpoints accept:

```
iteration_id      (str, form)  - Iteration identifier
llm_option        (str, form)  - LLM override (actual value taken from config)
document_id       (int, form)  - Document type identifier
tech_stack_id     (int, form)  - Technology stack identifier
module_id         (int, form)  - Module identifier
embedding_id      (int, form)  - Embedding to use for retrieval
user_id           (int, form)  - Requesting user
```

All generation endpoints return immediately; generation runs in a background thread and updates the iteration status (`Running` → `Completed` / `Failed`).

### Document Generation — Agentic Pipeline (DeepAgents)

**Pipeline run & streaming**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/agent/run` | Start the agent pipeline as a background task; returns a `task_id` |
| `GET` | `/agent/run/{task_id}/stream` | SSE stream of pipeline events (supports reconnect & replay across restarts) |
| `POST` | `/agent/run/{task_id}/stop` | Cancel a running pipeline task server-side |
| `GET` | `/agent/run/{task_id}/plan` | Read the `doc_plan.json` the orchestrator is awaiting approval on |
| `PUT` | `/agent/run/{task_id}/plan` | Persist the user-edited plan and resume the agent (HITL) |
| `GET` | `/agent/status/{document_id}` | Check whether a task is running / orphaned for a document |
| `GET` | `/agent/download/{project_id}/{path}` | Download a generated file from the workspace |

**Documents & versions**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/agent/documents?module_id=` | List documents for a module |
| `POST` | `/agent/documents` | Create a new document record |
| `DELETE` | `/agent/documents/{document_id}` | Delete a document and all its versions/sessions |
| `GET` | `/agent/documents/{document_id}/status` | Get a document's generation status |
| `GET` | `/agent/documents/{document_id}/versions` | List versions (without content) |
| `GET` | `/agent/documents/{document_id}/versions/{version_id}` | Get a version with full content |
| `POST` | `/agent/documents/{document_id}/versions/{version_id}/restore` | Restore a version (deletes newer ones) |
| `DELETE` | `/agent/documents/{document_id}/versions/{version_id}` | Delete a specific version |
| `GET` | `/agent/documents/{document_id}/versions/{version_id}/docx` | Convert a version to DOCX and download |

**Sessions (chat history)**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/agent/sessions` | Create a chat session |
| `GET` | `/agent/sessions?module_id=` | List sessions for a module |
| `GET` | `/agent/sessions/{session_id}` | Get a session with messages |
| `GET` | `/agent/sessions/by-document/{module_id}/{document_id}` | Get the session for a document |
| `PUT` | `/agent/sessions/{session_id}` | Update session messages |
| `DELETE` | `/agent/sessions/{session_id}` | Delete a session |

**Templates (PDF / Markdown)**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/agent/documents/{document_id}/template` | Upload a PDF or Markdown template (PDF text is extracted for the agent) |
| `GET` | `/agent/documents/{document_id}/template/view-url` | Get a short-lived URL to view the source template |
| `GET` | `/agent/documents/{document_id}/template/raw` | Stream the raw template file (local mode) |
| `DELETE` | `/agent/documents/{document_id}/template` | Remove the template source and agent-readable copy |

`POST /agent/run` accepts a JSON body:

```jsonc
{
  "project_id": "123",          // module_id, used as the workspace/scope key
  "message": "Generate a BRD",  // latest user instruction (drives intent)
  "history": [],                 // prior chat turns [{role, content}, ...]
  "document_id": "abc123",      // target document (optional)
  "module_id": 123,              // module id (optional; falls back to project_id)
  "user_id": 1                   // requesting user (optional)
}
```

Generation runs in a background asyncio task and streams progress over SSE. A
module-scoped guard prevents parallel `code_analysis.json` runs; once analysis
exists, multiple documents in the module can generate concurrently.

### CodeSense — Static Analysis & Reports

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/code_analysis_status?project_id=&module_id=` | Get analysis progress percentage, active phase, and completed steps |
| `GET` | `/code_inventory/{project_id}/{module_id}` | Get parsed code inventory and overall statistics JSON |
| `GET` | `/html_report/{project_id}/{module_id}` | Get the complete, self-contained standalone HTML code analysis report |

### CodeSense — Document Generation & Sections

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/codesense_application_brd` | Generate multi-language CodeSense BRD in background thread; returns `trace_id` |
| `POST` | `/codesense_application_tsd` | Generate multi-language CodeSense TSD in background thread; returns `trace_id` |
| `GET` | `/brd/sections/{project_id}/{module_id}/{iteration_id}` | Get generated BRD section JSON (HTML content + Mermaid blocks) for UI rendering |
| `GET` | `/tsd/sections/{project_id}/{module_id}/{iteration_id}` | Get generated TSD section JSON (HTML content + Mermaid blocks) for UI rendering |
| `POST` | `/brd/compile_docx/{project_id}/{module_id}/{iteration_id}` | Compile BRD JSON sections into Word `.docx` with embedded Mermaid images |
| `POST` | `/tsd/compile_docx/{project_id}/{module_id}/{iteration_id}` | Compile TSD JSON sections into Word `.docx` with embedded Mermaid images |

Both `/codesense_application_brd` and `/codesense_application_tsd` accept:

```
iteration_id      (int, form)  - Iteration identifier
llm_option        (str, form)  - LLM model option (overridden by AGENT_LLM_MODEL)
document_id       (int, form)  - Document type identifier
tech_stack_id     (int, form)  - Technology stack identifier
module_id         (int, form)  - Module identifier
embedding_id      (int, form)  - Embedding to use for retrieval
user_id           (int, form)  - Requesting user
```

### CodeSense — Section Feedback & Regeneration

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/iterations/feedback` | Save user feedback (approved/rejected + notes) for an individual section |
| `GET` | `/iterations/feedback/{iteration_id}` | Retrieve all section feedback recorded for an iteration |
| `GET` | `/iterations/feedback/status/{feedback_id}` | Poll regeneration status for a specific feedback submission |

### CodeSense — Knowledge Graph & Visualization

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/graph/generate_graph` | Generate or force-rebuild dependency & call graph for a project/module |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/data` | Fetch filtered graph data (nodes, edges, statistics) for D3.js UI rendering |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/node` | Get detailed node metadata (type, language, calls, dependencies) |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/node/source` | Get source code snippet for citation drawer |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/matrix` | Get cross-file dependency interaction matrix |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/stats` | Get general graph statistics (node counts, edge counts, density) |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/entrypoints` | Detect and list true execution entry points grouped by language |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/subsystems` | Get detected subsystems and functional clusters |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/traverse/bfs` | Breadth-First Search traversal up to specified depth (sub-slice graph) |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/traverse/bfs/full` | Full BFS traversal from root returning all reachable nodes and edges |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/code_structure` | Hierarchical folder and file structure tree |
| `GET` | `/api/v1/graph/{project_id}/{module_id}/node_questions` | Get AI summary questions and file-type-specific prompt suggestions |
| `POST` | `/api/v1/graph/{project_id}/{module_id}/explain_node` | AI explanation of a node based on surrounding graph neighborhood |
| `POST` | `/api/v1/graph/{project_id}/{module_id}/chat_node` | Interactive conversational Q&A on a specific code node |

### CodeSense — Graph-Anchored BRD

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/graph_entry_points/{project_id}` | List all selectable entry points in the knowledge graph |
| `POST` | `/resolve_entry_point` | Match plain-English functionality description to an entry-point node ID via LLM |
| `POST` | `/generate_graph_anchored_brd` | Generate a targeted BRD anchored to a selected graph node / entry point |

### CodeSense — Trace Observability

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/codesense/trace-analysis/{trace_id}` | Fetch Langfuse telemetry, pipeline execution tree, token usage, and latency analysis |

---

## Document Generation Pipelines

### Pipeline 1: Traversal (AST-based)

Best suited for deep structural analysis of compiled languages.

```
ZIP Upload
    │
    ▼
Unzip & Traverse Directory
    │
    ▼
Parse Source Code (C# / VB.NET / COBOL via tree-sitter)
    │  ├── Extract: classes, interfaces, methods
    │  └── Extract: dependencies, variable types
    │
    ▼
Build Structured JSON
    │  ├── Resolve interface → implementing class mappings
    │  └── Resolve cross-file method references
    │
    ▼
Create Embeddings → Vector Store
    │
    ▼
Generate Document Outline (LLM + RAG)
    │
    ▼
Expand Each Section (LLM + RAG)
    │
    ▼
Assemble Word/PDF → Upload to Cloud Storage
```

**Supported languages:** C#, VB.NET, COBOL, Python, ASP.NET

### Pipeline 2: Iterative Expansion (Template-driven)

Best suited for business-context-aware documents with user-controlled structure.

```
User provides:
  ├── Business context description
  ├── Static template PDF  (document structure)
  ├── Iterative template PDF  (section detail level)
  └── Custom system + user prompts
    │
    ▼
Download & Parse Templates (PDF)
    │
    ▼
Generate Document Skeleton (LLM + RAG)
    │  └── Static template + business context
    │
    ▼
Iteratively Expand Each Section (LLM + RAG)
    │  ├── Iterative template guides depth
    │  └── Mermaid diagrams generated & refined
    │
    ▼
JSON → Markdown → HTML → Word (.docx)
    │
    ▼
Generate Table of Contents
    │
    ▼
Upload to Cloud Storage + Update DB
```

**Supported documents:** BRD, Application TSD, Program TSD, Application Insights

### Pipeline 3: Agentic (DeepAgents)

A conversational, agent-driven pipeline built on [DeepAgents](https://github.com/langchain-ai/deepagents) and LangGraph. An **orchestrator** agent plans and coordinates work, delegating to specialised **subagents**, with all artifacts written to a per-module workspace (Azure Blob, S3, or local filesystem) that doubles as the agent's virtual file system.

**Subagents:**

| Subagent | Responsibility | Output |
|----------|---------------|--------|
| `code-analyzer` | Reads the uploaded codebase and produces a structured navigation summary | `/code_analysis.json` |
| `doc-writer` | Writes each document section as HTML, grounded in source code with citations | `<scope>/final_*.html` sections |
| `quality-checker` | Scores each section on completeness, accuracy, clarity, citations | `/quality_review.json` |
| `doc-formatter` | Removes duplicate headings/blocks, normalises markdown/HTML structure | cleaned final doc |
| `doc-editor` | Applies targeted edits (add/remove/rename/reorder/merge/split sections) to an existing doc | edited final doc |

```
User message (chat)
    │
    ▼
Orchestrator: recon (ls /, ls <scope>) → DECISION TREE
    │  ├── New doc        → analyze → plan → (HITL approval) → write → assemble → review → format
    │  └── Modification   → doc-editor (targeted edits, grounded in code)
    │
    ▼
code-analyzer → /code_analysis.json   (skipped if it already exists)
    │
    ▼
Orchestrator writes <scope>/doc_plan.json
    │
    ▼
request_plan_approval  ⏸  (Human-in-the-loop: user reviews/edits the plan,
    │                      pipeline resumes via PUT /agent/run/{task_id}/plan)
    ▼
doc-writer → appends each section (HTML) into <scope>/final_*.html
    │
    ▼
Assemble (remove %%APPEND%%, insert TOC)
    │
    ▼
quality-checker → /quality_review.json → orchestrator rewrites low-scoring sections
    │
    ▼
doc-formatter → cleanup → version saved to DB → streamed to UI over SSE
```

**Key capabilities:**

- **Any document type** — BRD, TSD, HLD, SLD, PRD, SAD, API docs, etc. (driven by the user's request and the optional uploaded template)
- **Conversational & iterative** — chat sessions with history; follow-up messages route to `doc-editor` for targeted changes instead of regenerating
- **Human-in-the-loop** — execution pauses on `doc_plan.json` for user review/edits before any sections are written
- **Versioning** — every generation is stored as a document version; versions can be listed, viewed, restored, deleted, and exported to DOCX
- **Real-time streaming** — progress (and the live event log) is pushed over Server-Sent Events, with reconnect/replay support across server restarts
- **Pluggable workspace backend** — the agent's file system maps to Azure Blob, AWS S3, or local disk based on `CLOUD_STORAGE_PROVIDER`
- **Skills & memory** — canonical `SKILL.md` files (code-controlled, refreshed each run) guide subagent workflows; `AGENTS.md` accumulates per-project memory/feedback
- **Langfuse tracing** — runs are traced with user/session context when Langfuse is enabled

### Pipeline 4: CodeSense (Static Analysis, Knowledge Graph & Multi-Language Generation)

A deep static-analysis and knowledge-graph-driven reverse engineering pipeline designed for complex, heterogeneous, multi-language codebases — spanning legacy mainframe stacks (COBOL, JCL, BMS, PL/I, RPG, Assembly), enterprise backend systems (Java, C#, C/C++, SQL, Python), enterprise ERPs (JD Edwards Event Rules), and web tiers (JavaScript, JSP, ColdFusion, XML/DFDL).

```
Codebase ZIP Upload
    │
    ▼
Multi-Language Static Analysis (20+ Analyzers & Preprocessors)
    │  ├── Extract classes, programs, procedures, copybooks, JCL steps, SQL
    │  ├── Compute cyclomatic complexity & token metrics
    │  └── Detect dead code & unused symbols
    │
    ▼
Cross-Language Dependency & Call Graph Generation
    │  ├── Resolve intra- and cross-language invocations
    │  ├── Detect execution entry points & cluster into subsystems
    │  └── Build interactive D3 JSON + persist in Knowledge Graph (Kùzu)
    │
    ▼
AI Domain Identification & Architecture Summary
    │  ├── Business domain & application feature extraction
    │  ├── Multi-language architectural design summary (design_summary.txt)
    │  └── Standalone Interactive HTML Analysis Report (html_report.html)
    │
    ▼
CodeSense Contextual Multi-Retriever RAG
    │  ├── Multi-query retrieval over primary + secondary tech stacks
    │  ├── Injects design summary + companion artifacts into section prompts
    │  └── Targeted discovery: business logic, calculations, errors, data structures
    │
    ▼
Multi-Language Document Generation (BRD & Application TSD)
    │  ├── Generate structured section JSON (HTML + Mermaid diagrams)
    │  ├── Multi-step LLM validation & automated gap-filling
    │  └── Section quality scoring against strict completeness thresholds
    │
    ▼
Section Viewer, Feedback Loop & DOCX Compilation
    │  ├── Render section JSON with live Mermaid charts
    │  ├── Section-level review (Approve/Reject + feedback notes) & targeted regen
    │  └── Compile final HTML/JSON into formatted Word (.docx) document
```

#### 1. Multi-Language Static Code Analysis

CodeSense features dedicated analyzers and preprocessors for over 20 programming languages and dialects:

| Language | Extracted Entities & Artifacts | Dependency & Relationship Types |
|----------|--------------------------------|---------------------------------|
| **COBOL** | Programs, paragraphs, sections, data divisions, copybooks | `CALLS`, `PERFORMS`, `COPIES`, `ACCESSES_DB` (EXEC SQL), `USES_MAP` (CICS BMS) |
| **JCL** | Jobs, job steps (`EXEC PGM`), cataloged & in-stream PROCs, DD datasets | `INVOKES_PROC`, `EXECUTES_UTILITY`, `LOADS_FROM`, `ACCESSES_DATASET` |
| **BMS** | Screen mapsets, maps, field definitions, screen navigation | `USES_MAP`, `COPIES`, `NAVIGATES_TO` |
| **PL/I (PL/1)** | Procedures, entry points, structures, include members | `CALLS_PROCEDURE`, `CALLS_ENTRY`, `INCLUDES`, `ACCESSES_DB` |
| **RPG (III/IV/ILE)** | Programs, subroutines, copy members, file specifications | `CALLS`, `PERFORMS`, `COPIES`, `QUERIES_TABLE` |
| **C / C++** | Functions, methods, classes, structs, namespaces, header files | `CALLS_FUNCTION`, `INCLUDES`, `IMPORTS`, `USES_STRUCTURE` |
| **C# (.NET)** | Namespaces, classes, interfaces, methods, LINQ queries, DTOs | `CALLS`, `IMPORTS` (usings), `ACCESSES_DB` (EF / ADO), `USES_FRAMEWORK` |
| **Java** | Packages, classes, interfaces, methods, annotations, Spring routes | `CALLS`, `IMPORTS`, `ACCESSES_DB` (JPA / JDBC), `USES_MIDDLEWARE` |
| **Python** | Modules, classes, functions, async coroutines, imports | `CALLS`, `IMPORTS`, `ACCESSES_DB`, `CALLS_EXTERNAL` |
| **JavaScript / TypeScript** | Modules, classes, functions, export/import statements, API fetch calls | `CALLS`, `IMPORTS`, `CALLS_EXTERNAL`, `NAVIGATES_TO` |
| **SQL** | Tables, views, stored procedures, triggers, foreign keys, indexes | `QUERIES_TABLE`, `QUERIES_VIEW`, `USES_TRIGGER`, `USES_INDEX` |
| **XML / XSD / DFDL** | Schema definitions, message types, complex types, DFDL annotations | `INCLUDES`, `USES_STRUCTURE`, `CALLS_FUNCTION` (API mapping) |
| **JDE Event Rules (ER)** | Named Event Rules (NER), forms, controls, UBE reports, table I/O | `CALLS_JDE_APP`, `CALLS_JDE_FORM`, `CALLS_JDE_UBE`, `ACCESSES_JDE_TABLE` |
| **ColdFusion (CFM)** | Components (CFC), CF functions, include tags, database queries | `CALLS_FUNCTION`, `INCLUDES`, `ACCESSES_DB` |
| **JSP** | Custom taglibs, Java beans, servlet mappings, scriptlets | `CALLS_PROCEDURE`, `INCLUDES`, `IMPORTS` |
| **Fortran (77/90)** | Programs, subroutines, functions, modules, common blocks | `CALLS_PROCEDURE`, `CALLS_FUNCTION`, `INCLUDES` |
| **Assembly (ASM)** | Procedures, macros, segments, register operations, sys calls | `CALLS_PROCEDURE`, `USES_MACRO`, `INCLUDES` |
| **Sabretalk** | Programs, segments, macros, TPF system interfaces | `CALLS`, `USES_MACRO`, `CALLS_SYSTEM` |

**Preprocessors:**
- `improved_cobol_preprocessor.py` — Resolves fixed/free-format COBOL, inline comments, continuation columns, and nested `COPY` statements.
- `rpg_preprocessor.py` — Handles fixed-format RPG column specifications (H, F, D, I, C, O specs) and expands copy members.

**Metrics & Dead Code Detection:**
- Calculates **Cyclomatic Complexity** for every function, method, and procedure.
- Estimates **token usage** per file and language.
- Identifies **dead code** (unreferenced routines, orphaned copybooks/includes, unreachable paragraphs) and tags them with line ranges and confidence levels.

#### 2. Cross-Language Knowledge Graph & Call Graph

CodeSense models the entire codebase as a directed semantic property graph:

- **Cross-Language Linker (`cross_lang_linker.py`)** — Stitches disparate layers together:
  - Maps external DFDL/XSD schema operations to C++ API controller entry points.
  - Links JCL job step execution to COBOL / PL/I program binaries and datasets.
  - Links CICS `EXEC CICS SEND MAP` calls to physical BMS mapsets.
  - Connects backend application queries to database tables, views, and stored procedures.
- **Entrypoint Detector (`entrypoint_detector.py`)** — Combines topology (in-degree zero / boundary roots) with code pattern indicators (main methods, CICS transaction IDs, REST endpoints, batch jobs) to identify true execution entry points.
- **Subsystem Detector (`subsystem_detector.py`)** — Uses graph partitioning and connected component analysis to group files and programs into functional modules / subsystems.
- **Interactive BFS Traversal** — API endpoints (`/traverse/bfs`, `/traverse/bfs/full`) allow querying directed call chains, upstream callers, or downstream dependencies up to custom depth.
- **AI Graph Assistant (`graph_ai_summary.py`)** — Provides real-time AI node explanation (`/explain_node`), extension-specific Q&A chat (`/chat_node`), and contextual prompt suggestions (`/node_questions`).

#### 3. Standalone Interactive HTML Analysis Report

The CodeSense pipeline generates a self-contained, portable HTML report (`html_report.html`) containing:
- **Executive Metrics Dashboard** — Total files, lines of code, detected languages, cyclomatic complexity distributions, and dead code ratios.
- **Dedicated Language Tabs** — In-depth breakdown for Mainframe (COBOL/JCL/BMS/PL1), Web/API (Java/C#/Python/JS), Database (SQL), and Legacy ERP components.
- **Interactive Component Explorer** — Searchable and filterable tables of all programs, classes, methods, and tables.
- **Embedded Architecture Diagrams** — Flowcharts and sequence diagrams generated via Mermaid.js.
- **Built-in Excel / CSV Export** — Injected client-side table download tools (`table_download_utils.py`) for exporting inventory and metrics directly from the browser.

#### 4. Contextual Multi-Retriever RAG

During document generation, CodeSense constructs an optimized multi-retriever:
- Pulls from primary and secondary tech stack embeddings simultaneously.
- Incorporates the global `design_summary.txt` (architectural context, business domains, data flows).
- Executes multi-query retrieval per section (up to 150+ chunks per section) targeting business logic rules, computational formulas, validation logic, exception handling tables, and data structure schemas.

#### 5. Multi-Language BRD & TSD Generation

CodeSense generates granular, production-ready documentation:
- **Business Requirements Document (BRD)** (`/codesense_application_brd`):
  - Executive summary and business context
  - High-level business capabilities and functional breakdown
  - Granular business rules and decision matrices
  - Business exception handling and error remediation policies
  - End-to-end business process flows with Mermaid diagrams
- **Application Technical Specification Document (TSD)** (`/codesense_application_tsd`):
  - System architecture and component inventory
  - Cross-module call stack diagrams and interaction patterns
  - Database schemas, table interactions, CRUD matrices, and SQL operations
  - API endpoint specifications, payloads, and protocol mappings
  - Batch job flows, scheduling dependencies, and error recovery procedures
- **Validation & Quality Scoring (`llm_validation.py`, `QUALITY_CONFIG`)**:
  - Validates section completeness against 90% quality thresholds.
  - Automatically executes gap-fill retrieval and regeneration if structural subsections or calculation tables are missing.

#### 6. Section Viewer, Feedback Loop & Word Export

- **JSON Section Viewer** — Returns individual section HTML blocks (`GET /brd/sections/...`, `GET /tsd/sections/...`) for granular review in the UI.
- **Human-in-the-Loop Feedback** (`POST /iterations/feedback`) — Allows users to mark sections as approved or rejected with specific feedback comments, triggering targeted section re-runs without regenerating the entire document.
- **DOCX Compilation** (`POST /brd/compile_docx/...`, `POST /tsd/compile_docx/...`) — Converts section JSON into a beautifully formatted Microsoft Word `.docx` file with automated cover pages, headers/footers, styled tables, and server-rendered Mermaid flowchart images.

#### 7. Graph-Anchored BRD Generation

In addition to whole-application BRDs, CodeSense supports **Graph-Anchored BRD generation** (`POST /generate_graph_anchored_brd`):
- Users select an entry-point node or search via plain-English functionality descriptions (`POST /resolve_entry_point`).
- The pipeline extracts the exact execution subgraph via BFS traversal and generates a focused BRD covering solely the selected business capability and its downstream call chain.

---

## Supported Providers

### LLM Providers

| Provider | `LLM_OPTION_NAME` value | Notes / Multi-Model Roles |
|----------|------------------------|---------------------------|
| Azure OpenAI | `openai` | Primary / Tool LLM |
| OpenAI LiteLLM (Proxy) | `openai-litellm`, `openai-litellm_2`, `openai-litellm_3` | `_1` (chat), `_2` (agent doc gen), `_3` (tools) |
| Google Gemini | `gemini`, `gemini2`, `gemini3` | Primary, Agent (`gemini2`), Tools (`gemini3`) |
| AWS Bedrock (Claude) | `aws-bedrock-claude` | Bedrock Claude Converse |
| AWS Bedrock (Claude via SSO) | `aws-bedrock-claude-sso` | SSO credentials |

### Vector Stores

| Provider | `VECTOR_STORE_PROVIDER` value | Notes |
|----------|------------------------------|-------|
| Azure AI Search | `azure_search` | Recommended for production |
| AWS OpenSearch | `aws_opensearch` | Vector + full-text search |
| Chroma | `chroma` | Local development |

### Embedding Providers

| Provider | `EMBEDDING_PROVIDER` value |
|----------|--------------------------|
| Google Generative AI | `google` |
| Azure OpenAI | `azure` |
| OpenAI | `openai` |
| OpenAI LiteLLM (Proxy) | `openai-litellm` |
| AWS Bedrock (Titan) | `aws_embedding` |

### Databases

| Provider | `DATABASE_PROVIDER` value |
|----------|--------------------------|
| SQL Server (Azure) | `mssql` |
| SQL Server (AWS RDS) | `mssql-aws-rds` |
| PostgreSQL | `postgresql` |
| MySQL (AWS RDS) | `mysql` |

### Cloud Storage

| Provider | `CLOUD_STORAGE_PROVIDER` value |
|----------|-------------------------------|
| Azure Blob Storage | `azure` |
| AWS S3 | `aws` |

---

## Configuration Reference

Key tuneable settings in `config.py`:

| Variable | Default | Description |
|----------|---------|-------------|
| `POOL_MAX_WORKERS` | `4` | Background thread pool size |
| `TSD_PER_TASK_DELAY` | `0.8` | Seconds between concurrent TSD section tasks |
| `SIZE_OF_CODE_CHUNK` | `20000` | Max characters per code chunk for embeddings |
| `THRESHOLD_SIZE_CHUNK` | `2000` | Min characters before further splitting |
| `NO_OF_CHUNKS` | `5` | Number of RAG chunks retrieved per query |
| `SCORE_THRESHOLD` | `0.8` | Minimum similarity score for retrieval |
| `RETRIEVER_K` | `5` | Top-k documents returned by retriever |
| `MAX_CONTINUATION_ATTEMPTS` | `3` | Max LLM continuation calls for truncated responses |
| `MERMAID_SERVICE_URL` | _(required)_ | Custom mermaid rendering service base URL (e.g. `http://localhost:3000`). Diagrams are sent as a POST to `{MERMAID_SERVICE_URL}/api/convert` |


### LLM Continuation Handling

When a model returns `finish_reason: max_tokens` (output truncated), the system automatically:

1. Detects the truncation from the finish reason
2. Builds a continuation prompt using the last 3 messages + partial response
3. Makes up to `MAX_CONTINUATION_ATTEMPTS` additional calls
4. Merges responses (string concatenation or dict merge depending on type)

This ensures complete documents are generated even for large codebases.

---

## Deployment

### Docker

```bash
docker build -t codeinsightai-backend .
docker run -p 8000:8000 --env-file .env.production codeinsightai-backend
```

### AWS Elastic Beanstalk

Deploy the application to Elastic Beanstalk and set all environment variables in the EB environment configuration panel.

### Azure App Service

Deploy as a Python web app. Set all environment variables in Application Settings.

### Health Check

```
GET /version
→ { "version": "2.0.4", "environment": "production" }
```

---

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `production` | Stable production releases |
| `development` | Active development, PRs target this branch |

To contribute: branch from `development`, make your changes, and open a pull request targeting `development`.
