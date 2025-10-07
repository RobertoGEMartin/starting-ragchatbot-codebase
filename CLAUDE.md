# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a RAG (Retrieval-Augmented Generation) system for course materials that uses ChromaDB for vector storage, Anthropic's Claude for AI generation, and FastAPI for the backend. The system enables users to query course materials and receive context-aware responses with conversation history.

## Development Commands

### Running the Application
```bash
# Quick start (from root)
./run.sh

# Manual start
cd backend
uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

### Dependency Management
```bash
# Install dependencies (uses uv package manager)
uv sync
```

### Environment Setup
Create a `.env` file in the root directory:
```
ANTHROPIC_API_KEY=your_api_key_here
```

## Commit Guidelines

When creating commits, use simple commit messages without attribution footers or emoji.
Do not add "Generated with Claude Code" or "Co-Authored-By" text.

## Architecture

### Core RAG Pipeline Flow

The system follows a tool-based RAG architecture where Claude has access to search tools instead of receiving pre-retrieved context:

1. **User Query** → `RAGSystem.query()` (rag_system.py:102)
2. **AI Generation with Tools** → Claude decides whether to use search tools (ai_generator.py:43)
3. **Tool Execution** → If Claude calls search, `CourseSearchTool.execute()` runs (search_tools.py:52)
4. **Vector Search** → `VectorStore.search()` performs semantic search with filters (vector_store.py:61)
5. **Response Generation** → Claude synthesizes answer from search results
6. **Session Management** → Conversation history stored via `SessionManager` (session_manager.py:37)

### Key Components

**RAGSystem** (rag_system.py)
- Main orchestrator coordinating all components
- Manages document ingestion via `add_course_document()` and `add_course_folder()`
- Routes queries through AI generator with tool access

**VectorStore** (vector_store.py)
- Uses ChromaDB with two collections:
  - `course_catalog`: Course metadata for semantic course name matching
  - `course_content`: Chunked course material with lesson-level granularity
- Implements unified search interface with course name resolution and filtering
- Course names are fuzzy-matched via semantic search before content filtering

**DocumentProcessor** (document_processor.py)
- Parses structured course documents with format:
  ```
  Course Title: [title]
  Course Link: [url]
  Course Instructor: [instructor]
  Lesson 1: [title]
  Lesson Link: [url]
  [content]
  ```
- Implements sentence-based chunking with configurable overlap (config.py:19-20)
- Adds course/lesson context to chunks for better retrieval

**AIGenerator** (ai_generator.py)
- Handles Claude API interactions with tool calling
- Implements agentic tool execution loop (ai_generator.py:89)
- Uses static system prompt optimized for one-search-per-query pattern

**ToolManager & CourseSearchTool** (search_tools.py)
- Abstract tool interface for extensibility
- CourseSearchTool provides semantic search with course/lesson filtering
- Tracks sources from searches for UI display

**SessionManager** (session_manager.py)
- Maintains conversation history per session
- Configurable history limit (MAX_HISTORY in config.py:22)

### Configuration

All configuration is centralized in `backend/config.py`:
- `CHUNK_SIZE`: 800 characters (affects retrieval granularity)
- `CHUNK_OVERLAP`: 100 characters (maintains context between chunks)
- `MAX_RESULTS`: 5 (search results returned per query)
- `MAX_HISTORY`: 2 (conversation turns remembered)
- `EMBEDDING_MODEL`: "all-MiniLM-L6-v2" (sentence-transformers)
- `ANTHROPIC_MODEL`: "claude-sonnet-4-20250514"

### Document Loading

On startup, the application:
1. Loads documents from `../docs` directory (app.py:91-98)
2. Checks for existing courses to avoid duplication (rag_system.py:76)
3. Processes new courses only (prevents re-ingestion)

Supported formats: PDF, DOCX, TXT (document_processor.py:81)

### API Endpoints

**POST /api/query** (app.py:56)
- Processes user queries with optional session_id
- Returns answer with sources and session_id

**GET /api/courses** (app.py:76)
- Returns course statistics (total courses, course titles)

### Frontend

Simple HTML/JS/CSS interface in `frontend/`:
- Communicates with FastAPI backend via fetch API
- Displays answers with source attribution
- Maintains session state for conversation continuity

## Key Implementation Details

### Two-Collection Strategy
The system uses separate collections to enable semantic course name matching before content filtering. This allows queries like "search MCP course for tools" to work even if the actual course title is "Introduction to Model Context Protocol".

### Tool-Based RAG vs. Traditional RAG
Unlike traditional RAG where context is pre-retrieved and injected into prompts, this system gives Claude access to search tools. Claude decides when and how to search, enabling better handling of general knowledge vs. course-specific questions.

### Chunk Context Enhancement
First chunks of lessons include lesson context prefix (document_processor.py:186), and final lesson chunks include both course and lesson context (document_processor.py:234) to improve retrieval quality.

## Python Version

Requires Python 3.13+ (specified in pyproject.toml and .python-version)
