# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Always use `uv` to run Python in this project — never raw `python` or `pip`.**

```bash
# Install dependencies
uv sync

# Run the app (from project root)
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# Run any Python script
uv run python <script.py>

# Add a dependency
uv add <package>

# App runs at http://localhost:8000 (web UI) and http://localhost:8000/docs (API docs)
```

There are no tests or linting configured in this project.

## Architecture

This is a **Course Materials RAG (Retrieval-Augmented Generation) system** — a full-stack app where users ask questions about course content via a chat interface, and an AI answers using retrieved course material as context.

### Backend (`backend/`)

Python/FastAPI app. All backend imports are relative within the `backend/` directory (no package structure).

**Request flow for queries:**
`app.py` (FastAPI route) → `rag_system.py` (orchestrator) → `ai_generator.py` (Claude API call with tool definitions) → Claude decides whether to call `search_course_content` tool → `search_tools.py` (ToolManager dispatches) → `vector_store.py` (ChromaDB semantic search) → results fed back to Claude → final answer returned.

**Key design decisions:**
- **Tool-based RAG**: Instead of always retrieving context, Claude is given a `search_course_content` tool and decides when/how to search. The tool loop is in `ai_generator.py._handle_tool_execution()`.
- **Two ChromaDB collections**: `course_catalog` (course-level metadata for name resolution) and `course_content` (chunked text for semantic search). Course name resolution uses vector similarity on the catalog before filtering content.
- **Document format**: Course files in `docs/` follow a specific format — first 3 lines are metadata (`Course Title:`, `Course Link:`, `Course Instructor:`), then `Lesson N: title` markers separate content. Parsing logic is in `document_processor.py.process_course_document()`.
- **Session management**: In-memory only (no persistence). Sessions store last N conversation exchanges, formatted as text and injected into the system prompt.
- **Startup loading**: `app.py` on_event("startup") loads all docs from `../docs/`, skipping courses already in ChromaDB by title.

### Frontend (`frontend/`)

Static HTML/JS/CSS served by FastAPI's StaticFiles mount at `/`. No build step. Uses `marked.js` from CDN for markdown rendering. API calls go to `/api/query` and `/api/courses`.

### Models (`backend/models.py`)

Pydantic models: `Course` → `Lesson` (one-to-many), `CourseChunk` (text chunk with course/lesson reference). Course title is used as the unique identifier throughout (ChromaDB IDs, dedup on load).

### Config (`backend/config.py`)

Single `Config` dataclass. Key tunables: `CHUNK_SIZE=800`, `CHUNK_OVERLAP=100`, `MAX_RESULTS=5`, `MAX_HISTORY=2`. Uses `all-MiniLM-L6-v2` for embeddings, `claude-sonnet-4` for generation.
