# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Running the Application
```bash
# Quick start (recommended)
chmod +x run.sh
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

### Dependency Management
```bash
# Install/update dependencies
uv sync

# Add new dependency
uv add package-name

# Python version requirement: 3.13+
```

### Environment Setup
Create `.env` file in project root:
```
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

## Architecture Overview

### System Design: Tool-Based RAG with Intelligent Search

The application implements a **tool-based RAG pattern** where Claude AI intelligently decides whether to search the knowledge base or use general knowledge. This is NOT a traditional RAG system that always retrieves context.

**Query Flow Architecture:**
1. **Frontend** → FastAPI endpoint (`/api/query`)
2. **RAG System** orchestrates the flow
3. **AI Generator** (Claude) analyzes query and decides search strategy
4. **Tool Manager** provides `search_course_content` tool to Claude
5. Claude either:
   - Calls search tool → Vector Store → ChromaDB (if course-specific)
   - Uses general knowledge (if not course-specific)
6. Response flows back with sources tracked

### Component Interactions

**RAG System (`rag_system.py`)**
- Central orchestrator that coordinates all components
- Manages the tool-based search approach via `ToolManager`
- Maintains conversation sessions through `SessionManager`
- Key method: `query()` - processes user queries with optional tool usage

**AI Generator (`ai_generator.py`)**
- Integrates with Anthropic Claude API using tool calling
- System prompt instructs Claude to use search tool selectively
- Handles tool execution flow: initial response → tool calls → final response
- Model: `claude-sonnet-4-20250514`

**Vector Store (`vector_store.py`)**
- Two ChromaDB collections:
  - `course_catalog`: Course metadata for semantic name matching
  - `course_content`: Chunked course content with embeddings
- Two-stage search process:
  1. Course resolution (semantic match on course name if provided)
  2. Content search (vector similarity on chunks)
- Embedding model: `all-MiniLM-L6-v2`

**Search Tools (`search_tools.py`)**
- Implements Anthropic tool interface pattern
- `CourseSearchTool` exposes search to AI with smart filtering
- Tracks sources for attribution in responses

### Data Processing Pipeline

**Document Processing (`document_processor.py`)**
1. Parses course files for metadata (title, instructor, link)
2. Identifies lesson boundaries (`Lesson X: Title`)
3. Chunks text with sentence-based splitting (800 chars, 100 overlap)
4. Adds contextual prefixes to chunks for better retrieval

**Session Management (`session_manager.py`)**
- Maintains conversation history (last 2 exchanges by default)
- Session-based context for follow-up questions

### Configuration

Key settings in `backend/config.py`:
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation turns
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2

### Frontend Architecture

Simple vanilla JavaScript SPA:
- `frontend/index.html`: Main UI structure
- `frontend/script.js`: Handles API calls, session management, markdown rendering
- `frontend/style.css`: Responsive chat interface styling
- Uses Marked.js for markdown rendering

## Development Notes

### API Endpoints
- `POST /api/query`: Main query endpoint with session support
- `GET /api/courses`: Returns course statistics
- Static files served from `frontend/` directory

### Database
- ChromaDB persists to `./chroma_db/` directory
- Automatically loads documents from `docs/` on startup
- Checks for existing courses to avoid re-processing

### Course Document Format
Expected structure for documents in `docs/`:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: Introduction
[content...]

Lesson 1: [title]
Lesson Link: [url]
[content...]
```

### No Testing Infrastructure
Currently no test suite exists. When adding tests:
- Use pytest for testing framework
- Test vector search functionality
- Test AI tool execution flow
- Mock Anthropic API calls

### No Linting/Formatting Tools
Consider adding when needed:
- `uv add --dev ruff` for linting
- `uv add --dev black` for formatting
- `uv add --dev mypy` for type checking