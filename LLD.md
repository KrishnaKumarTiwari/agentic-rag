# Low-Level Design (LLD): Enterprise Agentic RAG with MCP

> **Status**: DRAFT
> **Version**: 1.0
> **Date**: 2026-01-19

## 1. Introduction
This Low-Level Design (LLD) document details the implementation specifics for the Agentic RAG system defined in the HLD. It covers API specifications, data schemas, component class designs, and configuration parameters required for development.

## 2. API Specifications

### 2.1. Agent Interaction API
The primary entry point for the frontend or external systems. Recommended to follow OpenAI-compatible limits for easier integration.

**Endpoint**: `POST /v1/agent/chat`

**Request Body**:
```json
{
  "session_id": "uuid-v4",
  "user_id": "corp-user-123",
  "messages": [
    { "role": "user", "content": "Fetch the Q3 sales report for EMEA." }
  ],
  "config": {
    "temperature": 0.1,
    "model": "gpt-4o"
  }
}
```

**Response Stream (Server-Sent Events)**:
- `event: tool_start` -> Payload: `{ tool: "mcp_sales_db", input: "..." }`
- `event: tool_end` -> Payload: `{ output: "..." }`
- `event: thought` -> Payload: `{ content: "Found data, now summarizing." }`
- `event: token` -> Payload: `{ text: "The" }` ...
- `event: complete` -> Payload: `{ citation_ids: ["doc-123"] }`

### 2.2. MCP Server Interface (JSON-RPC 2.0)
Each MCP Server (Knowledge, SQL, etc.) must implement the Model Context Protocol.

**Method**: `tools/list`
**Response**:
```json
{
  "tools": [
    {
      "name": "search_knowledge_base",
      "description": "Semantic search over corporate documents",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": { "type": "string" },
          "filters": { "type": "object" }
        }
      }
    }
  ]
}
```

**Method**: `tools/call`
**Params**: `{ "name": "search_knowledge_base", "arguments": { "query": "travel policy" } }`

## 3. Data Schema Design

### 3.1. Vector Database Schema (Qdrant JSON)
**Collection**: `corporate_knowledge`
**Configuration**: `Cosine Distance`, `HNSW Index`

```json
{
  "id": "uuid-v4",
  "vector": [0.012, -0.045, ...], // 1536-dim (OpenAI text-embedding-3-small)
  "payload": {
    "content": "Employees must submit travel expenses within 30 days...",
    "source_url": "https://sharepoint.corp/policies/travel.pdf",
    "chunk_index": 5,
    "metadata": {
      "title": "Global Travel Policy",
      "author": "HR_dept",
      "date_published": "2025-01-15",
      "compliance_level": "confidential"
    },
    "parent_doc_id": "doc-uuid-888"
  }
}
```

### 3.2. Relational Database (PostgreSQL)
Used for conversation history (Memory) and User Feedback.

**Table: `sessions`**
- `session_id` (PK, UUID)
- `user_id` (Index)
- `created_at` (Timestamp)
- `metadata` (JSONB) - store user preferences or context

**Table: `messages`**
- `id` (PK)
- `session_id` (FK)
- `role` (enum: user, assistant, system, function)
- `content` (Text)
- `token_count` (Int)

**Table: `feedback`**
- `id` (PK)
- `message_id` (FK) - The AI response being rated
- `score` (Int) - +1 or -1
- `comment` (Text)

## 4. Component Design & Logic

### 4.1. Ingestion Worker (Python / Celery)
**Modules**:
1.  **`DocumentLoader`**: Factory pattern to load `PdfLoader`, `HtmlLoader`.
2.  **`TextSplitter`**:
    - *Logic*: RecursiveCharacterTextSplitter with `chunk_size=512`, `chunk_overlap=50`.
    - *Semantic Extension*: If header is detected, force new chunk.
3.  **`MetadataExtractor`**: Extract Title, Date using Regex or LLM (expensive but accurate).
4.  **`EmbeddingClient`**: Automatic retry with exponential backoff for API rate limits.

### 4.2. Retrieval Engine (The "RAG" Core)
**Class**: `HybridRetriever`
- **Method `retrieve(query, top_k=20)`**:
    - Parallel execution:
        - `self.vector_db.search(query_vector)`
        - `self.keyword_db.search(query_text)` (Elasticsearch/BM25)
    - **Fusion**: Reciprocal Rank Fusion (RRF) to combine lists.
- **Method `rerank(results, top_k=5)`**:
    - Input: 20 fused results.
    - Action: Call Cross-Encoder (Cohere/BGE).
    - Output: Top 5 most relevant chunks.

### 4.3. Agent Orchestrator (LangGraph)
**Graph Nodes**:
- **`__start__`**: Analyze input.
- **`router`**: Logic: `if query needs_data -> goto tools; else -> goto direct_response`.
- **`tools_execution`**: Iterate tool calls and execute MCP clients.
- **`reflect`** (Optional): Criticize own output (for complex tasks).
- **`generate`**: Final LLM synthesis with citations.

## 5. Sequence Diagram: Query Execution

```mermaid
sequenceDiagram
    participant User
    participant Orch as Orchestrator
    participant Guard as Guardrails
    participant MCP as MCP Client
    participant VDB as VectorDB
    participant LLM

    User->>Orch: "What are the Q3 risks?"
    Orch->>Guard: Validate Input
    Guard-->>Orch: OK
    
    Orch->>LLM: "Plan: How to answer 'Q3 risks'?"
    LLM-->>Orch: "Call KnowledgeSearch(query='Q3 risks')"
    
    Orch->>MCP: call_tool("KnowledgeSearch", ...)
    MCP->>VDB: Hybrid Search + Rerank
    VDB-->>MCP: [Chunk A, Chunk B]
    MCP-->>Orch: Tool Output
    
    Orch->>LLM: "Context: [Chunk A, Chunk B]. Answer user."
    LLM-->>Orch: "Based on the reports, Q3 risks are..."
    
    Orch->>Guard: Validate Output (Hallucination Check)
    Guard-->>Orch: OK
    Orch->>User: Final Response
```

## 6. Configuration & Environment Variables

| Variable | Default | Description |
| :--- | :--- | :--- |
| `OPENAI_API_KEY` | - | Core LLM Key |
| `VECTOR_DB_URL` | `localhost:6333` | Qdrant/Milvus Endpoint |
| `MCP_KNOWLEDGE_URL` | `http://mcp-knowledge:8080` | Internal MCP Server |
| `CHUNK_SIZE` | `512` | Token count per chunk |
| `RERANK_ENABLED` | `true` | Enable cross-encoder reranking |
| `LOG_LEVEL` | `INFO` | Application logging level |

## 7. Observability Specifications

### 7.1. OpenTelemetry Spans
Structure for tracing an Agentic Request.

**Parent Span**: `agent_execution`
- `attributes.session_id`: "sess-123"
- `attributes.user_id`: "u-456"

**Child Spans**:
1.  **`tool_call`**
    - `name`: "mcp_knowledge_search"
    - `attributes.tool.name`: "knowledge"
    - `attributes.tool.args`: "{\"query\": \"...\"}"
2.  **`llm_generation`**
    - `name`: "gpt4o_completion"
    - `attributes.llm.model`: "gpt-4o"
    - `attributes.llm.token_count`: 154

### 7.2. Metrics (Prometheus)
- **`rag_latency_seconds`** (Histogram): Buckets [0.5, 1, 2.5, 5, 10, 30]
- **`rag_requests_total`** (Counter): Labels: `status` (success/error), `user_tier`.
- **`rag_tokens_consumed`** (Counter): Labels: `model_name`, `type` (prompt/completion).
- **`rag_feedback_score`** (Gauge): Rolling average of user feedback.

## 8. Evaluation Configuration (Ragas)

Configuration for the Offline Evaluation Pipeline.

```yaml
# ragas_config.yaml
metrics:
  - context_precision
  - context_recall
  - faithfulness
  - answer_relevancy

evaluation_set:
  path: "datasets/golden_qa_v1.json"
  size: 200

thresholds:
  faithfulness: 0.9
  answer_relevancy: 0.85
```

