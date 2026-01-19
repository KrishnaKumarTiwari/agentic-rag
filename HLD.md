# High-Level Design (HLD): Enterprise Agentic RAG with MCP

> **Status**: DRAFT
> **Version**: 1.0
> **Date**: 2026-01-19

## 1. Executive Summary
This document outlines the High-Level Design for a production-grade **Agentic Retrieval-Augmented Generation (RAG)** system suitable for enterprise environments. The architecture moves beyond simple "Chain" based RAG to an **Agentic** approach, utilizing the **Model Context Protocol (MCP)** to standardize connections between the AI Reasoning Engine (The Agent) and enterprise data/tools.

Key differentiating features:
- **Agentic Core**: Dynamic planning and tool use instead of static pipelines.
- **MCP Integration**: Standardized protocol for connecting to internal Data & API silos.
- **Hybrid Search**: Semantic (Vector) + Keyword (BM25) + Re-ranking.
- **Enterprise Guardrails**: PII redaction, topic adherence, and hallucination detection.

## 2. System Context Diagram
The system sits between Corporate Users and the Enterprise Data Ecosystem.

```mermaid
graph TD
    User[Corporate User] -->|HTTPS/WSS| Frontend[Web/Mobile UI]
    Frontend -->|REST/GraphQL| Gateway[API Gateway / Load Balancer]
    
    subgraph AgenticPlatform [Agentic RAG Platform]
        Gateway --> Orchestrator[Agent Orchestrator Service]
        Orchestrator <-->|Memory| StateStore[Redis/Postgres]
        Orchestrator <-->|Inference| LLM_Gateway[LLM Gateway / Model Service]
        
        subgraph Guardrails [Guardrails Layer]
            InputGuard[Input Guardrail]
            OutputGuard[Output Guardrail]
        end
        
        Orchestrator -.-> InputGuard
        Orchestrator -.-> OutputGuard
    end

    subgraph MCPEcosystem [MCP Ecosystem - Tools and Data]
        Orchestrator -- "MCP Protocol (JSON-RPC)" --> Host[MCP Host/Client]
        
        Host -->|Connects| KMCP[Knowledge MCP Server]
        Host -->|Connects| SQLMCP[SQL/Data MCP Server]
        Host -->|Connects| ActionMCP[API Action MCP Server]
        
        KMCP <-->|Search| VectorDB[(Vector Database)]
        KMCP <-->|Search| Elastic[(Search Engine)]
        
        SQLMCP <-->|Query| DataWarehouse[(Snowflake/BigQuery)]
        ActionMCP <-->|API| InternalApp[Internal ERP/CRM]
    end

    subgraph DataPipeline [Data Pipeline - Async]
        Docs[Documents/Confluence/SharePoint] --> Ingestion[Ingestion Service]
        Ingestion -->|Split/Clean| Parsing[Advanced Parsing]
        Parsing -->|Embed| EmbedModel[Embedding Service]
        EmbedModel --> VectorDB
    end

    LLM_Gateway --> OpenAI[External LLM (GPT-4o)]
    LLM_Gateway --> SelfHosted[Internal LLM (vLLM/Llama)]
```

## 3. Core Component Architecture

### 3.1. Frontend & Gateway Layer
- **UI**: Supports streaming responses, citation rendering, and "thought process" visibility (optional).
- **Gateway**: Handles AuthN (SSO/SAML), Rate Limiting, and Request Validation.

### 3.2. Agent Orchestrator (The "Brain")
- **Role**: Replaces the static RAG chain. It receives the user query, maintains conversation history, and decides *which* tools to call.
- **Framework**: LangGraph / AutoGen / Custom State Machine.
- **Logic**:
    1.  **Plan**: Deconstruct query (e.g., "Compare sales Q1 vs Q2").
    2.  **Tool Selection**: Decide to call `Knowledge_MCP` (for policy) or `SQL_MCP` (for sales data).
    3.  **Synthesize**: Combine results into a cohesive answer.

### 3.3. Model Context Protocol (MCP) Layer
Standardizes how the Agent accesses data. Instead of hardcoding SDKs for every DB, we use MCP Servers.
- **MCP Host**: The runtime within the Agent Platform that manages connections to MCP Servers.
- **MCP Servers**: Lightweight sidecars or services that expose specific resources (Prompts, Resources, Tools).
    - *Example*: A Salesforce MCP Server exposes `get_customer(id)` as a tool.

### 3.4. Retrieval & Knowledge Base (The "Library")
- **Hybrid Search**: Combines Dense Vector Search (KNN) for semantic meaning and Sparse Keywood Search (BM25) for exact matches (product codes, names).
- **Re-ranking**: A Cross-Encoder model (e.g., Cohere Rerank, BGE-Reranker) rescores the top-N results to improve precision before sending to the LLM.

### 3.5. Data Ingestion Pipeline
- **Unstructured ETL**: Handles PDFs, PPTX, HTML.
- **Steps**:
    1.  **Extraction**: OCR for images, layout preservation for tables (Unstructured.io / Adobe API).
    2.  **Chunking**: Semantic chunking (not just fixed tokens) based on headers or paragraphs.
    3.  **Enrichment**: Adding metadata (Author, Date, Category) to chunks for filtering.
    4.  **Embedding**: Asynchronous batch processing.

### 3.6. Safety & Governance (Guardrails)
- **Input Rails**: detailed prompt injection detection (e.g., Lakera, NeMo Guardrails).
- **Output Rails**: PII redaction (Microsoft Presidio) and Hallucination checks (Self-Check GPT or citation verification).

## 4. Key Workflows

### 4.1. "Smart" Retrieval Flow
1.  **User Query**: "What is the reimbursement policy for travel?"
2.  **Agent**: Analyzes query. Calls `mcp_knowledge_server.search(query="travel reimbursement policy", filters={type: "policy"})`.
3.  **MCP Server**: Executes Hybrid Search on VectorDB + Re-ranks results. Returns top 5 chunks.
4.  **Agent**: Reads chunks. Determines answer is sufficient.
5.  **Agent**: Generates response with citations.

### 4.2. Complex Multi-Step Flow
1.  **User Query**: "Summarize the ACME project status and email it to the PM."
2.  **Agent**: 
    - Step 1: Call `mcp_jira_server.get_tickets(project="ACME")`.
    - Step 2: Call `mcp_confluence_server.get_page(title="ACME Roadmap")`.
    - Step 3: Synthesize summary.
    - Step 4: Ask User for confirmation to send email? (Human-in-the-loop).
    - Step 5: Call `mcp_outlook_server.send_email(...)`.

## 5. Technology Stack Recommendations
- **LLM**: GPT-4o / Claude 3.5 Sonnet (Capabilities), Llama 3 70B (Private).
- **Orchestration**: LangGraph (Python) or customized LangChain.
- **Vector DB**: Qdrant / Weaviate / Milvus (Production scale).
- **Ingestion**: Unstructured.io / LlamaParse.
- **Observability**: LangSmith / Arize Phoenix.
