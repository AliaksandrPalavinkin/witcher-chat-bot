---

description: "Task list for Witcher Character Persona Chat"
---

# Tasks: Witcher Character Persona Chat

**Input**: Design documents from `/specs/001-character-persona-chat/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/chat-api.md, quickstart.md

**Tests**: Not explicitly requested in the spec; this task list includes contract/unit tests for the RAG and stateless-behavior guarantees anyway since they encode hard requirements (FR-006, FR-009) that are easy to silently break. Persona-voice quality (SC-003/SC-004) is validated manually per `quickstart.md`, not via automated tests.

**Organization**: Tasks are grouped by user story (US1, US2, US3) per `spec.md`, after Setup and Foundational phases.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- Paths are relative to the repo root; base package `org.epam.witcherchatbot`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Add the dependencies and base config every persona endpoint needs.

- [ ] T001 Add `spring-boot-starter-web` dependency to `pom.xml`.
- [ ] T002 Add LangChain4j BOM/dependencies to `pom.xml`: `langchain4j-core`, `langchain4j-open-ai`, `langchain4j-pgvector`, `langchain4j-embeddings-all-minilm-l6-v2` (per `research.md` §1-3).
- [ ] T003 Add PostgreSQL JDBC driver dependency to `pom.xml` (required by `langchain4j-pgvector`'s `PgVectorEmbeddingStore`).
- [ ] T004 [P] Create package skeleton: `src/main/java/org/epam/witcherchatbot/{chat,persona,llm,lore,config}` (per `plan.md` Project Structure).
- [ ] T005 [P] Create `src/main/resources/lore-corpus/` directory with a `.gitkeep` and a short `README.md` explaining that book text files are supplied locally by the project owner and are not committed (per `research.md` §4 — copyrighted material).
- [ ] T006 Add Postgres + pgvector, Groq, and app-level config properties to `src/main/resources/application.yaml`: `spring.datasource.*` (host/port/db/user/password), `groq.base-url=https://api.groq.com/openai/v1`, `groq.api-key=${GROQ_API_KEY}`, `groq.model` (e.g. `llama-3.3-70b-versatile`), all externalized via environment variables — no secrets committed.

**Checkpoint**: Project compiles with new dependencies; no persona logic yet.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared infrastructure every persona endpoint depends on — LLM client config, lore retrieval, and the common request/response contract. **No user story can be completed until this phase is done.**

- [ ] T007 [P] Define `Persona` model in `src/main/java/org/epam/witcherchatbot/persona/Persona.java`: fields `id` (enum `GERALT`, `JASKIER`, `YENNEFER`), `displayName`, `systemPrompt`, `temperature` (double), `allowsCreativeInvention` (boolean), `fallbackLine` (per `data-model.md` Persona).
- [ ] T008 [P] Author the three persona system prompts as text (Geralt: terse/dry/weary/pragmatic, strictly lore-grounded per FR-002/FR-006; Jaskier: flamboyant/witty/boastful/poetic, may improvise per FR-006; Yennefer: sharp-tongued/precise/sarcastic/commanding, strictly lore-grounded per FR-002/FR-006). Every prompt MUST also explicitly instruct the model to keep responses within FR-005's mature-but-not-explicit tone boundary — authentic personality and dark-fantasy flavor preserved, but graphic violence, sexual content, and explicit profanity filtered or softened — regardless of persona. Wire the prompts into `PersonaRegistry` (or equivalent) in `src/main/java/org/epam/witcherchatbot/persona/PersonaRegistry.java`, alongside each persona's `temperature` (low for Geralt/Yennefer, higher for Jaskier per `research.md` §8) and `fallbackLine` (a static in-character line used ONLY for upstream LLM failure/rate-limit degradation, per FR-011 and `research.md` §5 — NOT used for ordinary lore-uncertainty, which is handled dynamically by the model itself per T025/FR-010).
- [ ] T009 Define the shared `ChatRequest` DTO in `src/main/java/org/epam/witcherchatbot/chat/ChatRequest.java`: `message` (String, required, non-empty — 400 if blank per contract), `conversationContext` (String, optional, per FR-009 / data-model.md Chat Request).
- [ ] T010 Define the shared `ChatResponse` DTO in `src/main/java/org/epam/witcherchatbot/chat/ChatResponse.java`: `personaId`, `content`, `language`, `wasFallback` (boolean) (per `contracts/chat-api.md` and data-model.md Chat Response).
- [ ] T011 Implement a simple language-detection utility in `src/main/java/org/epam/witcherchatbot/chat/LanguageDetector.java` that maps an incoming message to a BCP-47-style language tag, to satisfy FR-012 (reply mirrors detected input language, exercised at minimum for Russian and English per spec Assumptions).
- [ ] T012 Configure three named `OpenAiChatModel` beans in `src/main/java/org/epam/witcherchatbot/llm/GroqChatModelConfig.java` — one per persona, same `groq.base-url`/`groq.api-key`/`groq.model`, differing only in `temperature` from `PersonaRegistry` (per `research.md` §1 and §8).
- [ ] T013 Implement the retry/fallback wrapper in `src/main/java/org/epam/witcherchatbot/llm/ResilientChatClient.java`: bounded retry with short backoff on transient Groq errors, and on persistent failure/rate-limit return the calling persona's `fallbackLine` with `wasFallback=true` instead of propagating a raw error. This path is strictly for upstream infrastructure failure (per `research.md` §5, FR-011) — it is NOT the mechanism for FR-010's "unsure about a lore fact" behavior, which T025 handles via prompt instruction while the LLM call itself still succeeds.
- [ ] T014 [P] Configure the local embedding model bean (`AllMiniLmL6V2EmbeddingModel`) in `src/main/java/org/epam/witcherchatbot/lore/EmbeddingModelConfig.java` (per `research.md` §3).
- [ ] T015 [P] Configure the shared `PgVectorEmbeddingStore` bean in `src/main/java/org/epam/witcherchatbot/lore/VectorStoreConfig.java`, reading Postgres connection details from `application.yaml` (per `research.md` §2, `data-model.md` LoreChunk).
- [ ] T016 Implement the lore ingestion component in `src/main/java/org/epam/witcherchatbot/lore/LoreIngestionRunner.java`: reads files from `src/main/resources/lore-corpus/`, splits with `DocumentSplitters.recursive(...)`, embeds and stores via `EmbeddingStoreIngestor`, idempotent per `metadata.source` (skip/replace already-ingested files) (per `research.md` §4, `data-model.md` LoreChunk validation rule).
- [ ] T017 Implement `LoreRetrievalService` in `src/main/java/org/epam/witcherchatbot/lore/LoreRetrievalService.java` wrapping an `EmbeddingStoreContentRetriever` over the shared `PgVectorEmbeddingStore`, returning the top-k relevant `LoreChunk` contents for a given message (used by all three personas).
- [ ] T018 Implement `PersonaChatService` in `src/main/java/org/epam/witcherchatbot/chat/PersonaChatService.java`: given a `Persona` and a `ChatRequest`, retrieves lore context via `LoreRetrievalService`, builds the prompt (system prompt + retrieved lore + `conversationContext` if present + `message`), calls the persona's `ResilientChatClient`, detects reply language via `LanguageDetector`, and assembles a `ChatResponse`. This is the single shared code path all three controllers call into.

**Checkpoint**: Foundation ready — `PersonaChatService` can be invoked for any persona; controllers can now be added per story.

---

## Phase 3: User Story 1 - Talk to a chosen persona via its own endpoint (Priority: P1) 🎯 MVP

**Goal**: A caller can call `/api/v1/geralt/chat`, `/api/v1/jaskier/chat`, or `/api/v1/yennefer/chat` directly and get back a reply in that persona's distinctive voice, with Jaskier able to improvise creatively while Geralt and Yennefer stay lore-grounded (FR-001, FR-002, FR-004, FR-005, FR-006, FR-008).

**Independent Test**: Call each endpoint with the same casual message and confirm distinct, in-character voices; call each endpoint with an identical open-ended creative request and confirm Jaskier improvises while Geralt/Yennefer decline or stay grounded (spec User Story 1, Acceptance Scenarios 1-5).

### Implementation for User Story 1

- [ ] T019 [P] [US1] Implement `GeraltChatController` in `src/main/java/org/epam/witcherchatbot/chat/GeraltChatController.java`: `POST /api/v1/geralt/chat`, validates `ChatRequest.message` is non-empty (400 otherwise, per contract), delegates to `PersonaChatService` with `Persona.GERALT`.
- [ ] T020 [P] [US1] Implement `JaskierChatController` in `src/main/java/org/epam/witcherchatbot/chat/JaskierChatController.java`: `POST /api/v1/jaskier/chat`, same validation, delegates to `PersonaChatService` with `Persona.JASKIER`.
- [ ] T021 [P] [US1] Implement `YenneferChatController` in `src/main/java/org/epam/witcherchatbot/chat/YenneferChatController.java`: `POST /api/v1/yennefer/chat`, same validation, delegates to `PersonaChatService` with `Persona.YENNEFER`.
- [ ] T022 [US1] Add a global `@ExceptionHandler`/`@ControllerAdvice` in `src/main/java/org/epam/witcherchatbot/chat/ChatExceptionHandler.java` mapping blank/missing `message` to HTTP 400 with a clear error body, per `contracts/chat-api.md` Errors section (shared by all three controllers).
- [ ] T023 [US1] Tune each persona's system prompt (from T008) and `temperature` so casual-chat voice differences (Acceptance Scenarios 1-3) and the creative-license divergence (Acceptance Scenarios 4-5) are both satisfied; validate manually via `quickstart.md` scenarios 1-2, including confirming (FR-008) each response's `personaId` matches the endpoint called.

**Checkpoint**: All three persona endpoints are independently callable and voice/creative-license behavior is correct — this alone is a demoable MVP.

---

## Phase 4: User Story 2 - Ask Witcher lore questions in persona (Priority: P1)

**Goal**: Any persona endpoint answers Witcher-lore questions factually, using the shared RAG pipeline, and admits uncertainty in-character instead of inventing facts (FR-003, FR-010).

**Independent Test**: Ask each endpoint a well-known lore question and confirm factual, in-voice answers; ask an obscure/unanswerable question and confirm an in-character "I don't know" rather than fabrication (spec User Story 2, Acceptance Scenarios 1-2).

### Implementation for User Story 2

- [ ] T024 [US2] Populate `src/main/resources/lore-corpus/` with at least one Witcher source text file (user-supplied) and run `LoreIngestionRunner` (T016) end-to-end against a local Postgres+pgvector instance to confirm chunks are embedded and stored.
- [ ] T025 [US2] Extend `PersonaChatService` (T018) prompt construction so retrieved lore chunks are clearly presented to the model as background context distinct from conversation history, and so the model is instructed (system-prompt-level, per persona) to say so in-character when the retrieved context doesn't cover the question, rather than inventing an answer (FR-010).
- [ ] T026 [P] [US2] Add a unit test in `src/test/java/org/epam/witcherchatbot/lore/LoreRetrievalServiceTest.java` verifying `LoreRetrievalService` returns relevant chunks for a known query against a seeded in-memory/test vector store.
- [ ] T027 [US2] Validate manually via `quickstart.md` scenario 3 (lore Q&A + obscure-question uncertainty) against all three endpoints.

**Checkpoint**: Lore Q&A works consistently across all three personas, layered on top of the US1 MVP.

---

## Phase 5: User Story 3 - Maintain conversational context across a multi-turn exchange (Priority: P2)

**Goal**: A caller-supplied `conversationContext` string keeps replies coherent across turns, without this service storing any state itself; omitting it still produces a valid reply (FR-009).

**Independent Test**: Send a request with `conversationContext` referencing an earlier topic and confirm the reply is coherent with it; send a request with no `conversationContext` and confirm a valid reply is still returned; repeat an identical request twice and confirm both succeed independently (spec User Story 3, Acceptance Scenarios 1-2).

### Implementation for User Story 3

- [ ] T028 [US3] Confirm/adjust `PersonaChatService` (T018) prompt assembly to explicitly fold `conversationContext` in as prior-conversation context (when present) ahead of the current `message`, and to skip that section entirely (not error) when `conversationContext` is null/blank (FR-009, edge case: malformed/empty context).
- [ ] T029 [P] [US3] Add a unit test in `src/test/java/org/epam/witcherchatbot/chat/PersonaChatServiceTest.java` asserting: (a) a request with `conversationContext` set produces a prompt that includes it, (b) a request with `conversationContext` omitted still produces a valid `ChatResponse` with no exception, and (c) no field/state on `PersonaChatService` or its collaborators is mutated as a result of a call (statelessness).
- [ ] T030 [US3] Validate manually via `quickstart.md` scenario 4 (stateless context handling), including sending the same request twice and confirming independent, non-cross-contaminated replies.

**Checkpoint**: All three user stories (P1, P1, P2) are independently functional; full spec scope for v1 is covered.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Requirements that cut across all three user stories rather than belonging to one.

- [ ] T031 [P] Implement in-character off-topic deflection: ensure each persona's system prompt (T008) instructs the model to stay in character and deflect when `message` is unrelated to the Witcher universe, rather than refusing or breaking character (FR-007). Validate via `quickstart.md` scenario 5.
- [ ] T032 [P] Implement in-character abuse/harm deflection: ensure each persona's system prompt (T008) instructs the model to stay in character and rebuff abusive/harmful requests rather than emitting a generic safety-refusal message (FR-011). Validate via `quickstart.md` scenario 5.
- [ ] T033 [P] Verify FR-012 language mirroring end-to-end (Russian and English at minimum) across all three endpoints via `quickstart.md` scenario 6.
- [ ] T033a [P] Verify FR-005's tone/content boundary end-to-end: send a message likely to provoke graphic/explicit content to each endpoint and confirm the reply stays in-character but softens/filters the graphic details, per `quickstart.md` scenario 6 (tone/content boundary) and the FR-005 instruction added to each persona's system prompt in T008.
- [ ] T034 Verify graceful Groq-failure degradation end-to-end (invalid key / network cut) returns `wasFallback: true` with HTTP 200 rather than a 5xx, per `quickstart.md` scenario 8 and `ResilientChatClient` (T013).
- [ ] T034a [P] Verify SC-001's ≤10s response-time target: time a request to each of the three endpoints under normal conditions and confirm each completes within 10 seconds, per `quickstart.md` scenario 9. If T013's retry/backoff (transient-error path) risks exceeding this budget, tune its backoff/retry-count accordingly.
- [ ] T035 [P] Add a Testcontainers-based integration test in `src/test/java/org/epam/witcherchatbot/lore/LoreIngestionIntegrationTest.java` spinning up a `pgvector/pgvector` container, running ingestion (T016) against a small fixture text file, and asserting re-running ingestion for the same `metadata.source` does not duplicate chunks (idempotency, per `data-model.md` LoreChunk validation rule).
- [ ] T036 Update `README.md`/`CONTRIBUTING.md` with local run instructions (Postgres+pgvector via Docker, `GROQ_API_KEY`, lore corpus placement) consistent with `quickstart.md` Prerequisites/Setup.
- [ ] T037 Run the full `quickstart.md` validation script end-to-end (all 9 scenarios) as a final sign-off pass.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately.
- **Foundational (Phase 2)**: Depends on Setup. **Blocks all user stories** — `PersonaChatService`, the LLM beans, and the vector store must exist before any controller can be wired up.
- **User Story 1 (Phase 3)**: Depends on Foundational only. This is the MVP.
- **User Story 2 (Phase 4)**: Depends on Foundational; in practice builds on the controllers from Phase 3 (reuses the same endpoints), but its own tasks (lore corpus population, retrieval-quality prompt work) are additive and don't modify US1's files.
- **User Story 3 (Phase 5)**: Depends on Foundational; likewise layers on top of the existing endpoints via `PersonaChatService` prompt-assembly changes.
- **Polish (Phase 6)**: Depends on Phases 3-5 being complete.

### User Story Dependencies

- **US1 (P1)**: No dependency on US2/US3 — independently testable per spec.
- **US2 (P1)**: Independently testable (lore Q&A can be verified without any US3 context-passing), but shares controllers/service with US1.
- **US3 (P2)**: Independently testable (context-passing behavior can be verified without lore-specific questions), but shares controllers/service with US1/US2.

### Within Each User Story

- Foundational service/config tasks before controller wiring.
- Controllers for all three personas can be built in parallel ([P] tasks, different files).
- Manual `quickstart.md` validation is the last task in each story's phase.

### Parallel Opportunities

- T004, T005 (Setup) in parallel.
- T007, T008 (Persona model + prompts) in parallel; T014, T015 (embedding model + vector store config) in parallel.
- T019, T020, T021 (three controllers) fully in parallel — different files, same shared service dependency already built in Phase 2.
- T026 (US2 test) and T029 (US3 test) can run in parallel with each other and with polish tasks once their respective implementation tasks are done.
- T031, T032, T033, T033a, T034a, T035 (Polish) in parallel.

---

## Parallel Example: User Story 1

```bash
# Launch all three persona controllers together (after Phase 2 is complete):
Task: "Implement GeraltChatController in src/main/java/org/epam/witcherchatbot/chat/GeraltChatController.java"
Task: "Implement JaskierChatController in src/main/java/org/epam/witcherchatbot/chat/JaskierChatController.java"
Task: "Implement YenneferChatController in src/main/java/org/epam/witcherchatbot/chat/YenneferChatController.java"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup.
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories).
3. Complete Phase 3: User Story 1 — three persona endpoints with correct voice/creative-license behavior.
4. **STOP and VALIDATE**: Run `quickstart.md` scenarios 1-2 against all three endpoints.
5. Demo: three distinct personas, each independently callable.

### Incremental Delivery

1. Setup + Foundational → shared infrastructure ready.
2. Add User Story 1 → validate → MVP (three personas, distinct voices, correct creative-license split).
3. Add User Story 2 → validate → lore Q&A works across all personas.
4. Add User Story 3 → validate → multi-turn context via caller-supplied summaries works.
5. Polish → off-topic/abuse deflection, language mirroring, graceful degradation, ingestion idempotency, docs.

---

## Notes

- [P] tasks = different files, no dependencies.
- [Story] label maps a task to its user story for traceability.
- No test tasks were requested by the spec; the handful of test tasks included (T026, T029, T035) target hard-to-verify-manually guarantees (retrieval relevance, statelessness, ingestion idempotency) rather than general coverage.
- Persona-voice quality (SC-003) and creative-license read-back (SC-004) are inherently subjective/manual and are validated via `quickstart.md`, not automated tests.
- Commit after each task or logical group; the pre-commit SonarCloud hook (`.githooks/pre-commit`) will run on every commit per `CONTRIBUTING.md`.
