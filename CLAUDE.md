# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Course Materials RAG (Retrieval-Augmented Generation) System - a full-stack web application that processes educational content into a searchable knowledge base and provides AI-powered responses to user queries about course materials.

## Development Commands

### Setup and Installation
```bash
# Install dependencies
uv sync

# Set up environment (copy .env.example to .env and add ANTHROPIC_API_KEY)
cp .env.example .env
```

### Running the Application
```bash
# Quick start (recommended)
chmod +x run.sh && ./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

### Development URLs
- Web Interface: http://localhost:8000
- API Documentation: http://localhost:8000/docs

## Architecture Overview

### Core RAG System Design
The application follows a modular RAG architecture with clear separation of concerns:

**RAG System (`rag_system.py`)** - Main orchestrator that coordinates all components:
- Manages document processing pipeline
- Orchestrates AI generation with tool-based search
- Handles session management and conversation context
- Coordinates vector search through tools

**Tool-Based Search Architecture** - Uses Anthropic's tool calling for structured searches:
- `ToolManager` registers and executes tools
- `CourseSearchTool` performs semantic search with course/lesson filtering
- Tools store sources for UI attribution and reset after each query
- AI decides when and how to use search tools based on query context

**Vector Storage Strategy** - ChromaDB with dual storage approach:
- Course metadata collection for high-level course information
- Course content collection for searchable text chunks
- Uses sentence-transformers for embedding generation
- Supports semantic search with course name and lesson number filtering

### Data Flow Architecture

**Document Processing Pipeline**:
1. `DocumentProcessor` reads course files and extracts structured data
2. Parses course title, lessons, and content using regex patterns
3. Chunks content with configurable size and overlap (800 chars, 100 overlap)
4. Creates `Course` and `CourseChunk` models for storage

**Query Processing Pipeline**:
1. FastAPI receives query and manages session state
2. RAG system builds context from conversation history
3. AI generator calls Claude API with available search tools
4. If Claude uses tools, `CourseSearchTool` performs vector search
5. Tool results are synthesized into final response
6. Sources are collected from tools and returned to frontend

### Key Configuration Parameters

Located in `backend/config.py`:
- `CHUNK_SIZE: 800` - Text chunk size for vector storage
- `CHUNK_OVERLAP: 100` - Overlap between chunks to preserve context
- `MAX_RESULTS: 5` - Maximum search results returned
- `MAX_HISTORY: 2` - Conversation messages to retain
- `ANTHROPIC_MODEL: "claude-sonnet-4-20250514"` - AI model used
- `EMBEDDING_MODEL: "all-MiniLM-L6-v2"` - Sentence transformer for embeddings

### Session Management Design
- Session-based conversation tracking with automatic session creation
- Conversation history limited by `MAX_HISTORY` setting
- Messages stored as `Message` objects with role and content
- History formatted and passed to AI for context awareness

### Frontend-Backend Integration
- Vanilla JavaScript frontend with fetch-based API communication
- FastAPI serves both API endpoints and static files
- Real-time loading states and source attribution in UI
- Session persistence across page refreshes
- Markdown rendering for AI responses

## Data Models

The system uses three core Pydantic models:

**Course Model**: Represents complete courses with title (unique identifier), optional instructor, and lessons list

**Lesson Model**: Contains lesson number, title, and optional lesson link

**CourseChunk Model**: Text chunks for vector storage with course context, lesson number, and chunk position

## Tool System Architecture

The tool system enables Claude to perform structured searches:
- Tools implement abstract `Tool` interface with `get_tool_definition()` and `execute()` methods
- `ToolManager` handles registration, execution, and source tracking
- `CourseSearchTool` provides semantic search with optional course/lesson filtering
- Tools store search sources in `last_sources` for UI display
- Sources are reset after each query to prevent accumulation

## Environment Requirements

- Python 3.13+
- Anthropic API key (required)
- uv package manager
- ChromaDB for vector storage (auto-initialized)

## Important Implementation Notes

- The application auto-loads documents from `docs/` folder on startup
- Vector embeddings are created once and persisted in `./chroma_db`
- Course titles serve as unique identifiers across the system
- Conversation context is maintained per session but limited by `MAX_HISTORY`
- The AI generator uses temperature=0 for consistent responses
- CORS is configured to allow all origins for development
- always use uv for package installation and not pip
- use uv to run the server and not use pip directly
- make sure to use uv to manage all dependencies
- use uv to run python files