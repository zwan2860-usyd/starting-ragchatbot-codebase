# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Retrieval-Augmented Generation (RAG) system for answering questions about course materials. The system uses ChromaDB for vector storage, sentence-transformers for embeddings, and Anthropic's Claude for AI-powered responses with tool use.

## Running the Application

### Quick Start
```bash
./run.sh
```

### Manual Start
```bash
cd backend
uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

### Environment Setup
```bash
# Install uv package manager
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Set up environment variables
# Create .env file with: ANTHROPIC_API_KEY=your_key_here
```

## Architecture

### System Flow
1. User query → FastAPI endpoint (`/api/query`)
2. RAGSystem receives query and delegates to AIGenerator
3. AIGenerator calls Claude with tool definitions
4. Claude decides whether to use CourseSearchTool
5. If tool use: ToolManager executes search via VectorStore
6. VectorStore queries ChromaDB collections
7. Results returned to Claude for final response synthesis
8. Response and sources sent back to frontend

### Core Components

**RAGSystem** (`backend/rag_system.py`)
- Main orchestrator that coordinates all subsystems
- Handles document ingestion from files/folders
- Manages query processing with conversation context
- Initializes and connects all components

**VectorStore** (`backend/vector_store.py`)
- ChromaDB wrapper with two collections:
  - `course_catalog`: Course metadata (titles, instructors, lessons) for semantic course name resolution
  - `course_content`: Chunked course material for content search
- Implements unified search interface with automatic course name resolution
- Supports filtering by course and lesson number

**AIGenerator** (`backend/ai_generator.py`)
- Handles Claude API interactions
- Implements tool use pattern with automatic tool execution
- Manages conversation context and system prompts
- Uses `claude-sonnet-4-20250514` model with temperature=0

**ToolManager & CourseSearchTool** (`backend/search_tools.py`)
- Tool-based architecture following Anthropic's tool use pattern
- CourseSearchTool provides semantic search with course/lesson filtering
- Tool results include source tracking for UI display
- Extensible design allows registering additional tools

**DocumentProcessor** (`backend/document_processor.py`)
- Parses structured course documents with expected format:
  - Line 1: `Course Title: [title]`
  - Line 2: `Course Link: [url]`
  - Line 3: `Course Instructor: [instructor]`
  - Following: `Lesson N: [title]` markers with content
- Chunks text using sentence-based splitting with configurable overlap
- Creates CourseChunk objects with course and lesson context

**SessionManager** (`backend/session_manager.py`)
- Manages conversation sessions with message history
- Maintains configurable history limit (MAX_HISTORY=2 exchanges)
- Formats history for Claude's context window

**Models** (`backend/models.py`)
- `Course`: Title (used as unique ID), instructor, lessons, links
- `Lesson`: Number, title, link
- `CourseChunk`: Content, course_title, lesson_number, chunk_index

### Key Design Patterns

**Dual Collection Strategy**
The VectorStore uses two ChromaDB collections:
- Course catalog for fuzzy course name matching (user says "MCP" → resolves to "Introduction to MCP")
- Course content for actual semantic search of material

**Tool Use Pattern**
Rather than injecting retrieved context directly, the system provides Claude with a search tool. This allows Claude to:
- Decide whether to search or answer from general knowledge
- Make intelligent queries based on user intent
- Request specific course or lesson filtering

**Chunk Context Enhancement**
First chunk of each lesson gets prefix: `"Lesson N content: ..."`
Helps with lesson-specific queries and maintains context during retrieval

## Configuration

All settings in `backend/config.py`:
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2"
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: "./chroma_db" (relative to backend/)

## Adding Course Documents

Documents must follow the structured format with course metadata and lesson markers. Place files in the `docs/` directory and they will be auto-loaded on server startup.

Supported formats: `.pdf`, `.docx`, `.txt`

The system automatically skips re-processing existing courses based on title matching.
