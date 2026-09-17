# Quickstart: Validating the Witcher Character Persona Chat

## Prerequisites

- Java 21, and this repo's `./mvnw` wrapper (no local Maven install needed).
- A running PostgreSQL instance with the `pgvector` extension enabled, reachable from the app (e.g., via Docker: `docker run -e POSTGRES_PASSWORD=postgres -p 5432:5432 pgvector/pgvector:pg16`).
- A free Groq API key (https://console.groq.com), exported as an environment variable the app reads (e.g., `GROQ_API_KEY`).
- Lore source text files (Witcher book text you have the right to use locally) placed under `src/main/resources/lore-corpus/`.

Note: there is no gateway, session, or persona-selection endpoint to set up — each persona is called directly at its own URL (see `contracts/chat-api.md`).

## Setup

1. Configure `application.yaml` (or environment variables) with:
   - The Postgres connection details, used to build the shared `PgVectorEmbeddingStore`.
   - The Groq base URL (`https://api.groq.com/openai/v1`), API key (`${GROQ_API_KEY}`), and model name, used to build LangChain4j's two per-persona `OpenAiChatModel` beans (same model, different temperature per persona).
2. Run the lore ingestion step (one-time, and again whenever `lore-corpus/` changes) to populate the shared `pgvector` table with embedded chunks.
3. Start the app: `./mvnw spring-boot:run`.

## Validation scenarios

Each scenario maps to an acceptance scenario in [spec.md](./spec.md).

1. **Persona voice, called directly (User Story 1)**
   - `POST /api/v1/geralt/chat` with `{"message": "How's your day going?"}` → confirm the reply reads as terse/dry/world-weary.
   - `POST /api/v1/jaskier/chat` with the same message → confirm the reply reads as flamboyant/witty/boastful.

2. **Creative-license divergence (User Story 1, FR-006)**
   - `POST /api/v1/jaskier/chat` with `{"message": "Write a short ballad about a two-headed troll who loves cheese."}` → confirm Jaskier improvises an original in-character composition.
   - `POST /api/v1/geralt/chat` with the same message → confirm Geralt does *not* freely invent fanciful content; he should decline, deflect, or give a flat/grounded answer consistent with his stricter nature.

3. **Lore Q&A (User Story 2)**
   - `POST` to either endpoint with `{"message": "Who is Yennefer?"}` → confirm the reply is factually consistent with Witcher canon and delivered in the called persona's voice.
   - Ask an obscure/unanswerable question → confirm the reply admits uncertainty in-character (FR-010) rather than inventing facts.

4. **Stateless context handling (User Story 3, FR-009)**
   - Send a request with a `conversationContext` value referencing an earlier (simulated) exchange → confirm the reply is coherent with that supplied context.
   - Send a request with `conversationContext` omitted → confirm a valid in-character reply is still returned (no error).
   - Send the exact same request twice → confirm both calls succeed independently (no state carried between them by this service).

5. **Off-topic / abusive input (Edge cases, FR-007, FR-011)**
   - Send a message unrelated to the Witcher universe (e.g., "what's today's date?") to either endpoint → confirm an in-character deflection, not a generic refusal or a straight factual answer.
   - Send an abusive/harm-seeking message → confirm an in-character rebuff, not a raw safety-policy message.

6. **Language mirroring (FR-012)**
   - Send a message in Russian → confirm the reply is in Russian.
   - Send a message in English → confirm the reply is in English.

7. **Graceful degradation (Constraints)**
   - Temporarily use an invalid Groq API key or disconnect network access, then send a message → confirm the response is the called persona's in-character fallback line with `wasFallback: true` (200 OK), not a raw 5xx error.

## Expected outcome

All seven scenarios pass on repeated runs without any setup/teardown between them — every call is independent since this service is stateless.
