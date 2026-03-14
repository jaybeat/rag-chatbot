# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Course Materials RAG (Retrieval-Augmented Generation) System - a full-stack application that answers questions about course materials using semantic search and AI-powered responses.

## Common Development Commands

### Setup
```bash
# Install dependencies (requires uv package manager)
uv sync

# Set up environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
```

### Running the Application
```bash
# Quick start (recommended)
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

### Backend Structure (`backend/`)

The backend follows a modular RAG architecture with clear separation of concerns:

**Core Components (coordinated by `RAGSystem`)**

1. **`rag_system.py`** - Main orchestrator that initializes and coordinates all components. Key methods:
   - `add_course_document()` - Process and index a single course file
   - `add_course_folder()` - Batch process all documents in `../docs/`
   - `query()` - Main entry point for user queries with tool-based search

2. **`vector_store.py`** - ChromaDB wrapper managing two collections:
   - `course_catalog` - Course metadata (title, instructor, lessons) for semantic course matching
   - `course_content` - Actual course material chunks with course_title and lesson_number metadata
   - Uses `all-MiniLM-L6-v2` embeddings via sentence-transformers

3. **`document_processor.py`** - Parses course documents with expected format:
   ```
   Course Title: [title]
   Course Link: [url]
   Course Instructor: [name]
   Lesson 1: [Lesson Title]
   Lesson Link: [url]
   [content...]
   Lesson 2: ...
   ```
   - Extracts metadata from first 3 lines
   - Splits lessons by "Lesson N:" pattern
   - Creates overlapping chunks with sentence-aware splitting

4. **`ai_generator.py`** - Anthropic Claude API client with:
   - Static system prompt with search tool usage rules
   - Tool execution support via `_handle_tool_execution()`
   - Conversation history integration

5. **`search_tools.py`** - Tool system for AI-driven search:
   - `CourseSearchTool` - Searches course content with optional course/lesson filters
   - `ToolManager` - Registers tools and tracks sources for UI display

6. **`session_manager.py`** - In-memory conversation history with configurable `MAX_HISTORY`

7. **`models.py`** - Pydantic models: `Course`, `Lesson`, `CourseChunk`

8. **`config.py`** - Configuration loaded from environment variables and hardcoded defaults

9. **`app.py`** - FastAPI application with:
   - CORS and trusted host middleware
   - API endpoints: `POST /api/query`, `GET /api/courses`
   - Static file serving from `../frontend/`
   - Startup event loads documents from `../docs/`

### Frontend Structure (`frontend/`)

Simple static files served by the backend:
- `index.html` - Chat interface with sidebar for course stats
- `script.js` - Handles API communication and UI updates
- `style.css` - Styling

### Document Processing Flow

1. Documents are loaded from `docs/` folder on startup
2. `DocumentProcessor` parses each file into `Course` + `CourseChunk` objects
3. `VectorStore.add_course_metadata()` adds course to catalog collection
4. `VectorStore.add_course_content()` adds chunks to content collection
5. Chunks include context prefix: "Course {title} Lesson {N} content: {chunk}"

### Query Flow

1. User query received at `POST /api/query`
2. `RAGSystem.query()` creates prompt and gets conversation history
3. `AIGenerator.generate_response()` calls Claude with `search_course_content` tool
4. If tool use requested, `ToolManager.execute_tool()` runs `CourseSearchTool.execute()`
5. Search results returned to Claude for final response generation
6. Sources extracted from tool and returned to UI
7. Exchange added to session history

## Configuration

Key settings in `backend/config.py`:
- `ANTHROPIC_MODEL` - Claude model (default: claude-sonnet-4-20250514)
- `EMBEDDING_MODEL` - Sentence transformer model (default: all-MiniLM-L6-v2)
- `CHUNK_SIZE` / `CHUNK_OVERLAP` - Document chunking parameters (800/100)
- `MAX_RESULTS` - Search results limit (5)
- `MAX_HISTORY` - Conversation memory (2 exchanges)
- `CHROMA_PATH` - Vector database location (./chroma_db)

## Data Storage

- **Vector DB**: `backend/chroma_db/` (ChromaDB persistent storage)
- **Documents**: `docs/` folder containing .txt, .pdf, or .docx course files
- **Sessions**: In-memory only (lost on restart)

## Technology Stack

- Python 3.13+
- FastAPI + Uvicorn
- ChromaDB + Sentence-Transformers
- Anthropic Claude API
- uv (package manager)
