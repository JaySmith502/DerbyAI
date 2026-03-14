# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MiroFish is a multi-agent swarm intelligence prediction engine. Users upload seed documents (reports, news, stories), describe a prediction requirement in natural language, and MiroFish builds a digital world where AI agents with independent personalities simulate social interactions on Twitter/Reddit platforms. The simulation results are analyzed into prediction reports.

## Development Commands

```bash
# Install all dependencies (root + frontend + backend)
npm run setup:all

# Start both frontend and backend concurrently
npm run dev

# Start individually
npm run backend    # Flask backend on port 5001
npm run frontend   # Vite dev server on port 3000

# Frontend build
npm run build

# Backend only (manual)
cd backend && uv run python run.py

# Install backend Python deps (creates venv via uv)
npm run setup:backend

# Run tests
cd backend && uv run pytest
```

## Required Environment Variables

Copy `.env.example` to `.env` in the project root. Two keys are mandatory:
- `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL_NAME` - Any OpenAI-SDK-compatible LLM
- `ZEP_API_KEY` - Zep Cloud for knowledge graph memory

Optional `LLM_BOOST_*` variants provide a separate accelerated LLM endpoint.

## Architecture

### 5-Step Workflow Pipeline

The entire application follows a sequential 5-step pipeline, reflected in both the frontend components (`Step1`-`Step5`) and backend API blueprints:

1. **Graph Building** (`/api/graph/*`) - Upload documents, generate ontology via LLM, build Zep knowledge graph
2. **Environment Setup** (`/api/simulation/create`, `/api/simulation/prepare`) - Read entities from Zep graph, generate OASIS agent profiles via LLM
3. **Simulation** (`/api/simulation/start`, `/api/simulation/*/round`) - Run dual-platform (Twitter + Reddit) parallel agent simulations via OASIS framework
4. **Report Generation** (`/api/report/generate`) - ReportAgent with tool-calling analyzes simulation results
5. **Deep Interaction** (`/api/report/chat`, `/api/simulation/*/interview`) - Chat with ReportAgent or interview individual simulation agents

### Backend (Python/Flask)

- **Entry point**: `backend/run.py` -> `app/__init__.py` (`create_app` factory)
- **API layer** (`app/api/`): Three Flask Blueprints
  - `graph_bp` at `/api/graph` - project CRUD, ontology generation, graph building
  - `simulation_bp` at `/api/simulation` - entity reading, simulation lifecycle
  - `report_bp` at `/api/report` - report generation, chat interaction
- **Services** (`app/services/`): Core business logic
  - `OntologyGenerator` - LLM-driven ontology extraction from documents
  - `GraphBuilderService` - Zep Cloud graph creation and data ingestion
  - `ZepEntityReader` - Reads and filters entities from Zep graphs
  - `OasisProfileGenerator` - LLM generates agent personas from entities
  - `SimulationConfigGenerator` - LLM generates simulation parameters (Pydantic models)
  - `SimulationManager` - Manages simulation state, persisted as JSON on disk
  - `SimulationRunner` - Runs OASIS simulations in subprocess, manages lifecycle
  - `SimulationIPCClient/Server` - IPC between Flask and simulation subprocess
  - `ZepGraphMemoryUpdater` - Updates Zep graph memory during simulation
  - `ReportAgent` - Tool-calling LLM agent that analyzes simulation data
  - `ZepToolsService` - Tools available to ReportAgent (search, panorama, interviews)
  - `TextProcessor` - Text chunking and preprocessing
- **Models** (`app/models/`): Data classes with JSON file persistence (no database)
  - `Project` / `ProjectManager` - Project state persisted under `backend/uploads/projects/`
  - `Task` / `TaskManager` - In-memory singleton with thread-safe task tracking
- **Utils** (`app/utils/`):
  - `LLMClient` - OpenAI SDK wrapper; strips `<think>` tags; supports JSON mode
  - `FileParser` - Extracts text from PDF (PyMuPDF), MD, TXT
  - `retry` - Retry decorator for LLM calls
  - `zep_paging` - Pagination helper for Zep API

### Frontend (Vue 3 + Vite)

- **Framework**: Vue 3 with vue-router, no state management library (uses local component state + `pendingUpload` store)
- **Views** map to workflow steps: `Home` -> `MainView` (orchestrates steps) -> `SimulationView` -> `SimulationRunView` -> `ReportView` -> `InteractionView`
- **Components**: `Step1GraphBuild`, `Step2EnvSetup`, `Step3Simulation`, `Step4Report`, `Step5Interaction`, `GraphPanel` (D3.js visualization), `HistoryDatabase`
- **API layer** (`src/api/`): Axios with retry wrapper; 5-minute timeout; proxy `/api` to backend via Vite config
- **Graph visualization**: D3.js for rendering knowledge graph

### Key Patterns

- **Async tasks**: Long-running operations (graph building, simulation prep, report generation) use background `threading.Thread` with `TaskManager` for progress tracking. Frontend polls task status.
- **No database**: All persistence is JSON files on disk under `backend/uploads/`. Projects, simulations, and reports each have their own directory structure.
- **LLM integration**: All LLM calls go through `LLMClient` using the OpenAI SDK format, making it compatible with any OpenAI-API-compatible provider.
- **OASIS framework**: Simulations run via the `camel-oasis` library in a subprocess. Communication uses IPC (stdin/stdout JSON protocol).
- **Dual-platform simulation**: Agents simultaneously interact on simulated Twitter and Reddit platforms with platform-specific action sets.

### Tech Stack

- **Backend**: Python 3.11+, Flask, uv (package manager), OpenAI SDK, Zep Cloud SDK, camel-oasis/camel-ai, PyMuPDF, Pydantic
- **Frontend**: Vue 3, Vite 7, vue-router 4, Axios, D3.js
- **Infrastructure**: Docker support, GitHub Actions CI for Docker image builds

## Important Notes

- The `.env` file lives at the **project root** (not in `backend/`). The backend config resolves it via relative path from `backend/app/config.py`.
- Windows: `run.py` sets UTF-8 encoding on stdout/stderr to handle Chinese characters in console output.
- The frontend Vite dev server proxies `/api` requests to `http://localhost:5001` (the backend).
- Simulation subprocesses are managed with cleanup handlers registered at app startup (`SimulationRunner.register_cleanup()`).
- The codebase uses Chinese for log messages, comments, and API error messages.
