# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Retrieval-Augmented Generation (RAG) Chatbot System** for educational course materials. Users ask questions about courses, and the system uses semantic search with ChromaDB to retrieve relevant content, then Claude generates contextual answers with source citations.

**Tech Stack:** FastAPI (backend), Vanilla JS (frontend), ChromaDB (vector DB), Claude API (AI), Sentence-Transformers (embeddings)

## Development Commands

### Setup
```bash
# Install dependencies (creates .venv automatically)
uv sync

# Configure API key in .env
ANTHROPIC_API_KEY=your-key-here
```

### Running the Application
```bash
# Quick start (from root)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

Access at: http://localhost:8000

### Development
```bash
# Run from different port
cd backend
uv run uvicorn app:app --reload --port 8080

# Clear ChromaDB and reload documents
# Delete ./backend/chroma_db/ folder, then restart server
```

## Architecture Overview

### Two-Turn AI Pattern (Tool Calling)
The system uses Claude's tool calling capability with a **two-API-call pattern**:

1. **First Call**: Claude receives user query + tool definitions, decides to use `search_course_content` tool
2. **Tool Execution**: System executes vector search in ChromaDB
3. **Second Call**: Claude receives search results, synthesizes final answer

This differs from traditional RAG where you inject retrieved context directly into the prompt.

### Dual Vector Store Strategy
ChromaDB maintains **two separate collections**:

- **`course_catalog`**: Course metadata (titles, instructors, lessons). Used for fuzzy course name resolution.
- **`course_content`**: Actual text chunks with metadata. Used for semantic content search.

When searching, the system first resolves the course name via semantic search on `course_catalog`, then searches `course_content` with that exact title as a filter.

### Document Processing Pipeline

**Format Expected** (`/docs` folder):
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
...
```

**Processing Flow**:
1. Parse metadata from first 3 lines (title, link, instructor)
2. Split into lessons using regex: `r'^Lesson\s+(\d+):\s*(.+)$'`
3. Chunk each lesson (800 chars, 100 char overlap, sentence-aware)
4. Prefix chunks with context: `"Course {title} Lesson {num} content: {text}"`
5. Store in ChromaDB with metadata: `{course_title, lesson_number, chunk_index}`

### Component Responsibilities

**Backend (`/backend`):**
- `app.py`: FastAPI endpoints (`POST /api/query`, `GET /api/courses`)
- `rag_system.py`: Master orchestrator, coordinates all components
- `ai_generator.py`: Claude API wrapper, handles tool calling flow
- `vector_store.py`: ChromaDB interface with dual-collection pattern
- `search_tools.py`: Tool definitions and execution (`CourseSearchTool`, `ToolManager`)
- `document_processor.py`: Parses course files, chunks text
- `session_manager.py`: In-memory conversation history (max 2 exchanges)
- `config.py`: Centralized configuration dataclass
- `models.py`: Pydantic schemas (`Course`, `Lesson`, `CourseChunk`)

**Frontend (`/frontend`):**
- `index.html`: Two-column layout (sidebar + chat)
- `script.js`: Handles API calls, renders markdown responses, manages UI state
- `style.css`: Dark theme styling

### Configuration Parameters (config.py)

Critical settings in `Config` dataclass:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (384-dimensional vectors)
- `CHUNK_SIZE`: 800 chars
- `CHUNK_OVERLAP`: 100 chars
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges kept in session

### Session Management

Sessions are **in-memory only** (not persistent):
- Each session tracks up to `MAX_HISTORY * 2` messages (2 exchanges = 4 messages)
- Session ID format: "session_1", "session_2", etc.
- History is appended to Claude's system prompt for context

### Source Attribution Flow

Sources are tracked separately from the search results:
1. `CourseSearchTool._format_results()` populates `self.last_sources`
2. After AI response, `ToolManager.get_last_sources()` retrieves them
3. Sources sent to frontend: `["Course Title - Lesson N", ...]`
4. Frontend renders as collapsible `<details>` section

## Key Development Considerations

### Adding New Tools
1. Create class inheriting from `Tool` (search_tools.py)
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` with tool logic
4. Register in `rag_system.py`: `tool_manager.register_tool(new_tool)`

### Modifying Document Format
If changing course document structure:
1. Update regex patterns in `document_processor.py:process_course_document()`
2. Adjust metadata extraction logic (lines 110-139)
3. Update `models.py` Pydantic schemas if needed

### Changing Chunking Strategy
Edit `document_processor.py:chunk_text()`:
- Sentence splitting regex at line 34
- Chunk size/overlap logic at lines 43-90
- Context prefix format at lines 185-234

### Vector Store Modifications
When adding new collections or changing schema:
1. Update `vector_store.py:__init__()` to create collections
2. Modify metadata structure in `add_course_content()`/`add_course_metadata()`
3. Update filter logic in `_build_filter()` for new metadata fields

### AI System Prompt
Located in `ai_generator.py:SYSTEM_PROMPT` (lines 8-30). Modifying this affects:
- When Claude decides to use tools
- Response tone and format
- Search behavior

## Data Flow Summary

```
User Query (frontend)
  → POST /api/query (app.py)
  → rag_system.query() (orchestrator)
  → ai_generator.generate_response() (first Claude call)
  → Claude decides to use tool
  → tool_manager.execute_tool()
  → vector_store.search() (two-stage: catalog → content)
  → ChromaDB vector similarity search
  → Format results + track sources
  → ai_generator._handle_tool_execution() (second Claude call)
  → Claude synthesizes answer
  → Return (answer, sources) to frontend
  → Render markdown + collapsible sources
```

## Prerequisites

- Python 3.13+
- uv package manager
- Anthropic API key
- Git Bash (Windows) for running shell scripts
