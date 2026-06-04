# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Tiny Ragent AI is a RAG (Retrieval-Augmented Generation) intelligent agent system built on Spring Boot. It provides intelligent document processing, vector-based retrieval, knowledge base management, conversational memory, and deep thinking capabilities.

## Build Commands

```bash
# Build all modules
mvn clean install -DskipTests

# Build with tests
mvn clean install

# Run tests for a specific module
mvn test -pl bootstrap

# Run a single test class
mvn test -pl bootstrap -Dtest=ClassName

# Run a single test method
mvn test -pl bootstrap -Dtest=ClassName#methodName

# Format code (Spotless)
mvn spotless:apply

# Check code format
mvn spotless:check
```

## Running the Application

```bash
# Run from bootstrap module
cd bootstrap && mvn spring-boot:run

# Or run the JAR directly
java -jar bootstrap/target/bootstrap-0.0.1-SNAPSHOT.jar
```

The application starts on port 9090 with context path `/api/ragent`.

## Module Structure

- **bootstrap**: Main application module containing business logic
  - `rag/` - RAG core (retrieval, intent recognition, chat pipeline)
  - `knowledge/` - Knowledge base, documents, chunks management
  - `ingestion/` - Document ingestion pipeline engine
  - `admin/` - Dashboard and admin controllers
  - `user/` - User management

- **framework**: Common infrastructure (no business logic)
  - Exception handling, result wrappers
  - Distributed ID generation (snowflake)
  - Idempotent annotations
  - Trace context for RAG operations
  - Message queue producers

- **infra-ai**: AI model integration layer
  - LLM chat clients (SiliconFlow, BaiLian)
  - Embedding service with routing
  - Rerank service with routing
  - Model health tracking and failover

## Key Architecture Patterns

### Multi-Channel Retrieval Engine

The retrieval system uses parallel search channels with priority-based execution:

1. **IntentDirectedSearchChannel** (priority 1) - Searches specific knowledge bases based on intent recognition
2. **VectorGlobalSearchChannel** (priority 10) - Fallback global search across all KBs when intent confidence is low

Flow: `MultiChannelRetrievalEngine` → `SearchChannel` implementations → `SearchResultPostProcessor` chain

### Model Routing with Failover

`ModelRoutingExecutor` provides high-availability model calls:
- Routes to model candidates by priority
- Tracks failures per model (`ModelHealthStore`)
- Opens circuit breaker after `failure-threshold` failures
- Auto-recovers after `open-duration-ms`

Used by: `RoutingLLMService`, `RoutingEmbeddingService`, `RoutingRerankService`

### Ingestion Pipeline Engine

Chain-based document processing pipeline with nodes:

```
FetcherNode → ParserNode → ChunkerNode → EnhancerNode → EnricherNode → IndexerNode
```

Each node:
- Implements `IngestionNode` interface
- Can have conditions for conditional execution
- Outputs flow to next node via `nextNodeId`

Engine: `IngestionEngine.execute(PipelineDefinition, IngestionContext)`

### RAG Trace System

Annotation-based distributed tracing:
- `@RagTraceRoot` - Marks entry point, creates trace ID
- `@RagTraceNode` - Marks child operations, builds tree

Aspect: `RagTraceAspect` captures timing, status, and errors.

## Configuration

Main config: `bootstrap/src/main/resources/application.yaml`

Key config sections:
- `rag.vector.type` - Vector store: `pg` or `milvus`
- `rag.search.channels` - Retrieval channel thresholds
- `ai.providers` - Model provider URLs and API keys
- `ai.chat/embedding/rerank.candidates` - Model candidates with priorities

Environment variables for secrets:
- `BAILIAN_API_KEY`
- `SILICONFLOW_API_KEY`

## Dependencies

- Spring Boot 3.5.7, Java 17
- PostgreSQL + pgvector (or Milvus for vectors)
- Redis/Redisson for caching and distributed locks
- RocketMQ for async messaging
- Apache Tika for document parsing (PDF, DOCX, Markdown, etc.)
- Sa-Token for authentication
