# Implementation Plan: Witcher Character Persona Chat

**Branch**: `001-character-persona-chat` | **Date**: 2026-09-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-character-persona-chat/spec.md`

## Summary

A stateless Spring Boot backend exposing two independent per-persona chat endpoints — `/api/v1/geralt/chat` and `/api/v1/jaskier/chat`. Each persona is defined by its own system prompt *and* its own LLM generation parameters (notably temperature): Geralt runs at low temperature and stays strictly lore-grounded, while Jaskier runs at a higher temperature and is allowed to improvise original creative content in character. Both personas draw on a shared lore knowledge base — Witcher book text chunked, embedded locally, and stored in PostgreSQL/pgvector, retrieved (RAG) and injected into the prompt sent to a Groq-hosted Llama model. LangChain4j is the LLM/RAG framework used throughout. This service holds no session state itself: each request optionally carries a caller-supplied conversation-context summary; an external AI gateway (routing, caching, cross-persona session/history management) is a separate service entirely out of scope here.

## Technical Context

**Language/Version**: Java 21

**Primary Dependencies**: Spring Boot 4.1.1 (`spring-boot-starter-web`), LangChain4j (`langchain4j-core`; `langchain4j-open-ai` for the `OpenAiChatModel` pointed at Groq's OpenAI-compatible endpoint, instantiated once per persona with its own temperature; `langchain4j-pgvector` for `PgVectorEmbeddingStore`; `langchain4j-embeddings-all-minilm-l6-v2` for the local ONNX embedding model; LangChain4j's `DocumentSplitters`/`EmbeddingStoreIngestor`/`EmbeddingStoreContentRetriever` for the RAG pipeline), Lombok

**Storage**: PostgreSQL 16+ with the `pgvector` extension, holding embedded lore chunks (`LoreChunk` records via LangChain4j's `PgVectorEmbeddingStore`), shared by both personas. No conversation-state storage of any kind — the service is stateless per spec FR-009; any caller-supplied context arrives in the request and is discarded after the response is produced.

**Testing**: JUnit 5 + `spring-boot-starter-test` for unit/slice tests; Testcontainers (Postgres+pgvector) for integration tests of the retrieval/ingestion path; a manual `quickstart.md` script for end-to-end validation against the live Groq API.

**Target Platform**: Self-hosted JVM server (local/dev machine or a single container), Linux/macOS

**Project Type**: Single web-service project (existing Maven module `org.epam.witcherchatbot`) — REST API only, called by an external gateway rather than an end-user client directly

**Performance Goals**: First in-character reply within 10s of a request (SC-001); no other throughput targets (this service is a backend called by one gateway, not a public multi-tenant API)

**Constraints**: Groq free-tier rate limits must be respected — on 429/error, degrade gracefully with an in-character "unsure/busy" response (ties to FR-010/FR-011) rather than a raw error; zero paid infrastructure required (free Groq inference tier, locally-run embedding model, self-hosted Postgres); no authentication; the service MUST NOT retain any request/response/session data between calls (statelessness is a hard requirement, not just an assumption, since an external gateway owns all session/history concerns)

**Scale/Scope**: Two independent persona endpoints at launch (Geralt, Jaskier), each with distinct temperature/creative-license settings; shared lore corpus initially the Witcher novel text (design allows adding game-dialogue corpora later without a data-model change); no gateway, routing, caching, or multi-user session logic in this repo

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is still the unpopulated template (placeholder principles only) — there are no ratified project-specific principles or gates to check against yet. This plan proceeds without constitutional constraints. **Recommendation**: run `/speckit-constitution` to ratify real principles before the next feature, so future plans have real gates to evaluate against.

No violations to track; `Complexity Tracking` section below is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-character-persona-chat/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
src/main/java/org/epam/witcherchatbot/
├── WitcherChatBotApplication.java
├── chat/                 # Two REST controllers (GeraltChatController, JaskierChatController) + shared request/response DTOs
├── persona/               # Persona definitions: system-prompt template + generation params (temperature, etc.) per persona
├── llm/                   # LangChain4j ChatLanguageModel bean(s) — one configured instance per persona (same Groq model, different temperature)
├── lore/                  # Ingestion (book text -> chunks -> embeddings via LangChain4j EmbeddingStoreIngestor), PgVectorEmbeddingStore config, EmbeddingStoreContentRetriever (shared by both personas)
└── config/                # Cross-cutting Spring configuration (datasource, embedding model, etc.)

src/main/resources/
├── application.yaml
└── lore-corpus/          # Raw source text files to be ingested (user-supplied, not fetched by tooling)

src/test/java/org/epam/witcherchatbot/
├── chat/
├── persona/
└── lore/
```

**Structure Decision**: Single Maven module, organized package-by-feature (`chat`, `persona`, `llm`, `lore`, `config`) under the existing `org.epam.witcherchatbot` base package. Two personas map to two controllers and two differently-configured `ChatLanguageModel` beans, sharing the same lore-retrieval infrastructure. No gateway code lives in this repository — it is an explicitly separate, externally-owned service (per spec Clarifications) that will call this app's per-persona endpoints over the network.

## Complexity Tracking

*No constitution gates are defined yet, so no violations to justify.*
