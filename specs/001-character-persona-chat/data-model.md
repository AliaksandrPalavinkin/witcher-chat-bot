# Data Model: Witcher Character Persona Chat

## Persona

Represents one of the three independently callable personas. Defined in code/config (not user-editable at runtime in v1) — a small fixed set, not a database table. Each persona backs exactly one REST endpoint (`/api/v1/geralt/chat`, `/api/v1/jaskier/chat`, `/api/v1/yennefer/chat`).

| Field | Type | Notes |
|---|---|---|
| `id` | enum/string (`GERALT`, `JASKIER`, `YENNEFER`) | Stable identifier, also used as the `personaId` tag on every response (FR-008) |
| `displayName` | string | e.g., "Geralt of Rivia", "Jaskier (Dandelion)", "Yennefer of Vengerberg" |
| `systemPrompt` | string (template) | Voice/tone/knowledge-scope instructions injected as the LLM system message (FR-002); MUST also encode the FR-005 mature-but-not-explicit tone boundary for every persona |
| `temperature` | double | LLM sampling temperature for this persona's `ChatLanguageModel` instance — low for Geralt and Yennefer (strict, lore-grounded, FR-006), higher for Jaskier (creative license, FR-006) |
| `allowsCreativeInvention` | boolean | Drives FR-006 behavior: whether the persona may improvise original content not grounded in the lore corpus (`true` for Jaskier, `false` for Geralt and Yennefer) |
| `fallbackLine` | string | Static in-character line used ONLY when the LLM call itself fails/is rate-limited (FR-011 / upstream degradation). This is distinct from FR-010's "unsure about a lore fact" behavior, which is a normal, successful LLM response produced dynamically via the system prompt, not this static line. |

**Validation rules**: exactly the three supported personas at launch (FR-001); adding a persona means adding a new enum value + prompt + params + a new controller endpoint, not a schema change.

## Chat Request

The input to a single call on one persona's endpoint. Not persisted — exists only for the duration of the request (service is stateless, FR-009).

| Field | Type | Notes |
|---|---|---|
| `message` | string | The caller's message for this turn; must be non-empty |
| `conversationContext` | string, optional | Caller-supplied recent-history excerpt or summary (e.g., last 10-15 messages, condensed by the external gateway) used to keep the reply coherent; may be absent |

**Validation rules**: `message` must be non-empty (400 if missing/blank); `conversationContext`, if present, is treated as opaque text handed to the model — this service does not parse or validate its internal structure.

## Chat Response

The output of a single call, returned synchronously.

| Field | Type | Notes |
|---|---|---|
| `personaId` | Persona id | Always matches the endpoint called; included so a caller can't misattribute a reply (FR-008) |
| `content` | string | The generated reply text |
| `language` | string (BCP-47 tag or similar) | Detected language of the incoming `message`, which the reply mirrors (FR-012) |
| `wasFallback` | boolean | `true` when `content` is the persona's fallback line due to an upstream LLM failure/rate-limit, so callers (and the external gateway) can distinguish a real reply from a degraded one |

## LoreChunk (vector store record)

A chunk of source lore text plus its embedding, used for retrieval-augmented generation. Shared across both persona endpoints. Persisted via LangChain4j's `PgVectorEmbeddingStore` (backing table typically named `embeddings` or as configured), not a hand-rolled JPA entity.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | LangChain4j-managed identifier |
| `content` | string | The chunked lore text (from book source files) |
| `embedding` | vector | Produced by the local `AllMiniLmL6V2EmbeddingModel` (LangChain4j) |
| `metadata.source` | string | Originating file/book, for traceability and future filtering (e.g., "Sword of Destiny") |
| `metadata.corpusType` | string | e.g., `BOOK` now, `GAME_DIALOGUE` as a future addition — supports extending the corpus without a schema change |

**Validation rules**: Ingestion must be idempotent per `metadata.source` (re-running ingestion for an already-ingested file should not duplicate chunks).

## Relationships

- Each `Persona` has exactly one dedicated endpoint, its own `temperature`/`allowsCreativeInvention` settings, and its own system prompt — there is no shared "active persona" concept (that notion belonged to the earlier single-endpoint design and has been replaced by one endpoint per persona).
- A `Chat Request`/`Chat Response` pair exists only for the lifetime of one HTTP call; nothing links two requests together server-side. Continuity across turns is achieved purely through the caller supplying `conversationContext` each time (per spec User Story 3).
- `LoreChunk` has no direct relationship to `Persona`; it is queried by semantic similarity to the current `message` to build the RAG context injected into whichever persona's prompt is being built, with each persona's own `allowsCreativeInvention`/`temperature` governing how strictly that context is followed versus how freely the model may go beyond it.
