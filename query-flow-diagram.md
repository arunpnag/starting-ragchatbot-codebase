# RAG System Query Flow Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend (script.js)
    participant API as FastAPI (app.py)
    participant RAG as RAG System
    participant SM as Session Manager
    participant AI as AI Generator
    participant Claude as Claude API
    participant TM as Tool Manager
    participant ST as Search Tool
    participant VS as Vector Store
    participant DB as ChromaDB

    Note over U,DB: User Query Processing Flow

    %% 1. User Input
    U->>F: Types query & presses Enter
    F->>F: Disable input, show loading
    F->>F: Add user message to chat

    %% 2. API Request
    F->>+API: POST /api/query {query, session_id}
    
    %% 3. Session Management
    API->>+SM: Check/create session
    SM-->>-API: session_id
    
    %% 4. RAG Processing
    API->>+RAG: query(query, session_id)
    RAG->>+SM: get_conversation_history(session_id)
    SM-->>-RAG: conversation_history
    
    %% 5. AI Generation
    RAG->>+AI: generate_response(query, history, tools)
    AI->>AI: Build system prompt + context
    
    %% 6. Claude API Call
    AI->>+Claude: messages.create() with tools
    Claude-->>-AI: Response (with tool_use)
    
    %% 7. Tool Execution (if needed)
    alt Claude wants to use tools
        AI->>+TM: execute_tool(name, params)
        TM->>+ST: execute(query, course_name, lesson_number)
        
        %% 8. Vector Search
        ST->>+VS: search(query, filters)
        VS->>VS: Apply semantic search
        VS->>+DB: Query with embeddings
        DB-->>-VS: Search results
        VS-->>-ST: SearchResults (docs, metadata, distances)
        
        %% 9. Format Results
        ST->>ST: Format results with context
        ST->>ST: Store sources in last_sources
        ST-->>-TM: Formatted search results
        TM-->>-AI: Tool execution results
        
        %% 10. Final AI Response
        AI->>+Claude: Follow-up call with tool results
        Claude-->>-AI: Final synthesized response
    end
    
    AI-->>-RAG: Generated response text
    
    %% 11. Source Collection & Session Update
    RAG->>+TM: get_last_sources()
    TM-->>-RAG: sources[]
    RAG->>+TM: reset_sources()
    TM-->>-RAG: ✓
    RAG->>+SM: add_exchange(session_id, query, response)
    SM-->>-RAG: ✓
    
    RAG-->>-API: (response, sources)
    
    %% 12. API Response
    API-->>-F: QueryResponse {answer, sources, session_id}
    
    %% 13. UI Update
    F->>F: Remove loading animation
    F->>F: Update session_id if new
    F->>F: addMessage(answer, 'assistant', sources)
    F->>F: Convert markdown to HTML
    F->>F: Display sources collapsible
    F->>F: Re-enable input & focus
    F->>U: Show response in chat

    Note over U,DB: Complete query processed with context and sources
```

## Architecture Overview

```mermaid
graph TB
    subgraph "Frontend Layer"
        UI[Web Interface<br/>index.html]
        JS[JavaScript Client<br/>script.js]
        CSS[Styling<br/>style.css]
    end
    
    subgraph "API Layer"
        FastAPI[FastAPI Server<br/>app.py]
        CORS[CORS Middleware]
        Static[Static File Server]
    end
    
    subgraph "RAG Core"
        RAGSys[RAG System<br/>rag_system.py]
        DocProc[Document Processor<br/>document_processor.py]
    end
    
    subgraph "AI & Tools"
        AIGen[AI Generator<br/>ai_generator.py]
        Claude[Claude API<br/>Anthropic]
        ToolMgr[Tool Manager<br/>search_tools.py]
        SearchTool[Course Search Tool]
    end
    
    subgraph "Data Layer"
        VectorStore[Vector Store<br/>vector_store.py]
        ChromaDB[(ChromaDB<br/>Persistent Storage)]
        SessionMgr[Session Manager<br/>session_manager.py]
        Embeddings[Sentence Transformers<br/>Embeddings]
    end
    
    subgraph "Content"
        Docs[Course Documents<br/>docs/*.txt]
        Models[Data Models<br/>models.py]
        Config[Configuration<br/>config.py]
    end

    %% Connections
    UI --> JS
    JS --> FastAPI
    FastAPI --> RAGSys
    RAGSys --> AIGen
    RAGSys --> SessionMgr
    AIGen --> Claude
    AIGen --> ToolMgr
    ToolMgr --> SearchTool
    SearchTool --> VectorStore
    VectorStore --> ChromaDB
    VectorStore --> Embeddings
    DocProc --> VectorStore
    DocProc --> Models
    Docs --> DocProc
    Config --> RAGSys

    %% Styling
    classDef frontend fill:#e1f5fe
    classDef api fill:#f3e5f5
    classDef core fill:#e8f5e8
    classDef ai fill:#fff3e0
    classDef data fill:#fce4ec
    classDef content fill:#f1f8e9
    
    class UI,JS,CSS frontend
    class FastAPI,CORS,Static api
    class RAGSys,DocProc core
    class AIGen,Claude,ToolMgr,SearchTool ai
    class VectorStore,ChromaDB,SessionMgr,Embeddings data
    class Docs,Models,Config content
```

## Key Data Structures

```mermaid
graph LR
    subgraph "Request/Response Models"
        QReq[QueryRequest<br/>- query: str<br/>- session_id: Optional[str]]
        QResp[QueryResponse<br/>- answer: str<br/>- sources: List[str]<br/>- session_id: str]
    end
    
    subgraph "Core Data Models"
        Course[Course<br/>- id: str<br/>- title: str<br/>- description: str<br/>- lessons: List[Lesson]]
        Lesson[Lesson<br/>- number: int<br/>- title: str<br/>- content: str]
        Chunk[CourseChunk<br/>- course_id: str<br/>- lesson_number: int<br/>- content: str<br/>- metadata: Dict]
    end
    
    subgraph "Search Results"
        SResults[SearchResults<br/>- documents: List[str]<br/>- metadata: List[Dict]<br/>- distances: List[float]<br/>- error: Optional[str]]
    end
    
    Course --> Lesson
    Lesson --> Chunk
    Chunk --> SResults
    QReq --> QResp
```