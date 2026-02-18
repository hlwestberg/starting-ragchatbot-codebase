# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

Always use `uv` to run the server — never use `pip` or `python` directly.

```bash
# Quick start
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

App runs at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

**First-time setup:**
```bash
uv sync                          # install dependencies
cp .env.example .env             # then add ANTHROPIC_API_KEY to .env
```

## Architecture

This is a full-stack RAG chatbot. The **backend** (`backend/`) is a FastAPI app served on port 8000, which also statically serves the **frontend** (`frontend/`) from the root path. There is no build step for the frontend — it's plain HTML/CSS/JS.

### Backend Component Responsibilities

- **`app.py`** — FastAPI entrypoint. Two API routes: `POST /api/query` and `GET /api/courses`. On startup, loads all `.txt` files from `../docs/` into the vector store (skips already-loaded courses).
- **`rag_system.py`** — Central orchestrator. Wires together all components and owns the `query()` method that drives the full RAG flow.
- **`ai_generator.py`** — Wraps the Anthropic API. Makes a first call to Claude with tools available; if Claude requests a tool (`stop_reason == "tool_use"`), executes it and makes a second call without tools to get the final answer.
- **`vector_store.py`** — ChromaDB wrapper with two collections: `course_catalog` (one doc per course, used for fuzzy course-name resolution) and `course_content` (chunked lesson text, used for semantic search). Course title is used as the document ID.
- **`document_processor.py`** — Parses `.txt` course files and chunks lesson content. Expected file format: first 3 lines are `Course Title:`, `Course Link:`, `Course Instructor:`, followed by `Lesson N: Title` / `Lesson Link:` / content blocks.
- **`search_tools.py`** — Defines the `Tool` ABC and `CourseSearchTool`, which is the only tool exposed to Claude. `ToolManager` registers tools and routes Claude's tool calls to the right implementation.
- **`session_manager.py`** — In-memory session store. Keeps last `MAX_HISTORY` (default: 2) exchanges per session. History is injected into the system prompt, not the messages array.
- **`config.py`** — Single `Config` dataclass, instantiated once as `config` and imported everywhere.
- **`models.py`** — Pydantic models: `Course`, `Lesson`, `CourseChunk`.

### Key Data Flow

1. Query arrives at `POST /api/query`
2. `RAGSystem.query()` fetches session history and calls `AIGenerator.generate_response()`
3. Claude decides whether to call `search_course_content` tool
4. If tool is called: `CourseSearchTool` → `VectorStore.search()` → ChromaDB → formatted chunk results returned to Claude
5. Claude synthesizes final answer in a second API call (no tools)
6. Sources collected from `ToolManager.get_last_sources()`, session history updated, response returned

### Adding New Course Documents

Drop `.txt` files into the `docs/` folder following the expected format. They are loaded automatically on server startup. To force a reload of already-indexed courses, call `add_course_folder(..., clear_existing=True)`.

### Extending with New Tools

Implement the `Tool` ABC in `search_tools.py`, then register with `ToolManager` in `RAGSystem.__init__()`. The tool definition dict must follow Anthropic's tool schema.
