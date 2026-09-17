# Feature Specification: Witcher Character Persona Chat

**Feature Branch**: `001-character-persona-chat`

**Created**: 2026-09-17

**Status**: Draft

**Input**: User description: "Let's develop a chatbot. The main idea is for it to interact with me in the style of characters from *The Witcher*. It should support different personas—let's start with Geralt and Dandelion. The bot needs to be able to answer lore-related questions or simply chat—for instance, by speaking like Geralt."

## Clarifications

### Session 2026-09-17

- Q: Is this chatbot meant for a single user with no login, or should it support multiple distinct users each with their own separate conversation/persona state? → A: Single user, no auth for v1; multi-user/authenticated accounts may be added later but are not required now.
- Q: If a user sends an abusive, hateful, or harm-seeking message, how should the system respond? → A: Stay in character and deflect/rebuff the request in the persona's own voice, rather than breaking character with a generic safety-refusal message.
- Q: Should the bot reply in whatever language the user's most recent message is written in, or always respond in one fixed language? → A: Mirror the user's message language on each turn (detect per message, not fixed).
- Q: Should this feature include a routing/gateway layer that decides which persona a user is talking to and caches responses? → A: No. This feature is only the per-persona chat backend, each persona exposed as its own independent endpoint. Deciding which persona a user is talking to, caching, and cross-persona conversation continuity are the responsibility of a separate, external AI gateway service, entirely out of scope here.
- Q: Does this service own and persist conversation history/session state itself? → A: No. Each persona endpoint is stateless — a request stands on its own, optionally including a short caller-supplied conversation summary/recent-history for context. The external gateway is responsible for tracking the full conversation and building that summary (last ~10-15 messages) before calling this service.
- Q: Should Yennefer be added as a third persona alongside Geralt and Jaskier? → A: Yes. Yennefer of Vengerberg ships in v1 as a third independent persona endpoint, sharing the same architecture (own system prompt, own creative-license/temperature setting) as the other two. Her voice is sharp-tongued, precise, sarcastic, and commanding — a powerful sorceress with little patience for nonsense — and, like Geralt, she stays strictly grounded in known lore rather than inventing fanciful content on request.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Talk to a chosen persona via its own endpoint (Priority: P1)

A caller sends a message to a specific persona's chat endpoint (Geralt, Jaskier, or Yennefer) and gets back a free-form conversational reply written in that character's distinctive voice, tone, and personality — including the persona-appropriate degree of creative license (Geralt and Yennefer stick to what they plausibly know; Jaskier is willing to improvise or invent, in character, as a bard would).

**Why this priority**: This is the core value proposition — talking *as if* to the character — and is required for the product to have any value at all.

**Independent Test**: Can be fully tested by calling one persona's endpoint directly with several conversational messages and verifying replies are consistently written in that character's voice.

**Acceptance Scenarios**:

1. **Given** a request to Geralt's endpoint, **When** the caller sends a casual conversational message (not a lore question), **Then** the response is terse, dry, and world-weary in the style of Geralt of Rivia.
2. **Given** a request to Jaskier's endpoint, **When** the caller sends a casual conversational message, **Then** the response is flamboyant, witty, and boastful in the style of Jaskier/Dandelion.
3. **Given** a request to Yennefer's endpoint, **When** the caller sends a casual conversational message, **Then** the response is sharp-tongued, precise, sarcastic, and commanding in the style of Yennefer of Vengerberg.
4. **Given** a request to Jaskier's endpoint, **When** the caller asks him to compose something creative on a topic not found in any source material (e.g., "write a short ballad about a two-headed troll who loves cheese"), **Then** Jaskier improvises an original, in-character composition rather than refusing or claiming he doesn't know.
5. **Given** the same creative request sent to Geralt's or Yennefer's endpoint instead, **When** they are asked to invent something not grounded in lore, **Then** they respond in character consistent with their stricter, less fanciful nature (e.g., declining or grounding their answer in what they actually know) rather than freely inventing lore as fact.

---

### User Story 2 - Ask Witcher lore questions in persona (Priority: P1)

A caller asks a persona's endpoint a question about Witcher-universe lore (characters, monsters, locations, events, history) and receives an accurate, in-universe answer delivered in that persona's voice.

**Why this priority**: Lore Q&A is explicitly called out as a core capability alongside casual chat, and is what differentiates this from a generic roleplay bot.

**Independent Test**: Can be fully tested by asking a known lore question (e.g., "who is Yennefer?") to each persona's endpoint and verifying the factual content is accurate and the delivery matches the persona's voice.

**Acceptance Scenarios**:

1. **Given** a request to any persona's endpoint, **When** the caller asks a question about a well-known Witcher character, monster, or event, **Then** the system responds with a factually consistent answer delivered in the persona's voice.
2. **Given** a request to any persona's endpoint, **When** the caller asks a lore question the system cannot answer confidently, **Then** the system says so in-character rather than inventing an answer.

---

### User Story 3 - Maintain conversational context across a multi-turn exchange (Priority: P2)

A caller includes a short summary or recent-message excerpt from the ongoing conversation alongside a new message, and the persona's reply stays coherent with that prior context, even though the service itself keeps no session state between calls.

**Why this priority**: Since each endpoint is stateless, coherent multi-turn conversation only works if the service correctly makes use of caller-supplied context; this is what makes the stateless design usable in practice.

**Independent Test**: Can be fully tested by sending a request that includes a short prior-context summary referencing something specific, then verifying the reply is coherent with that reference without the service having stored anything from an earlier call.

**Acceptance Scenarios**:

1. **Given** a request that includes a recent-history/summary field referencing an earlier topic, **When** the persona replies, **Then** the reply is coherent with that referenced context.
2. **Given** a request with no history/summary provided, **When** the persona replies, **Then** the system still produces a valid in-character reply based solely on the current message (no error).

---

### Edge Cases

- What happens when the user asks a question completely unrelated to the Witcher universe (e.g., asks for help writing code, or today's weather)? See FR-007.
- What happens when the user asks something in poor taste or tries to provoke an offensive response? See FR-005 and FR-011.
- What happens when the user asks a lore question with no clear canonical answer, or where sources conflict?
- What happens when the user asks the bot to "break character" and admit it is an AI?
- What happens when the caller-supplied history/summary is missing, empty, or malformed? See FR-009 (must degrade to a context-free reply, not an error).
- What happens when Geralt or Yennefer is asked to do something only Jaskier's persona is suited for (e.g., "compose a poem")? They should respond in their own character (e.g., gruffly declining, or dismissing the request with dry sarcasm) rather than switching into a bardic register.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST expose Geralt of Rivia, Jaskier (Dandelion), and Yennefer of Vengerberg as three separate, independently callable persona chat endpoints.
- **FR-002**: System MUST generate conversational responses that reflect each persona's distinctive voice, tone, and speech patterns (e.g., Geralt: terse, dry, weary, pragmatic; Jaskier: flamboyant, witty, boastful, poetic; Yennefer: sharp-tongued, precise, sarcastic, commanding).
- **FR-003**: System MUST be able to answer questions about Witcher-universe lore (characters, locations, creatures, events, history) with factually consistent answers, delivered in the requested persona's voice.
- **FR-004**: System MUST support free-form, non-lore conversational chat with each persona (casual talk, opinions, banter) in addition to lore Q&A.
- **FR-005**: System MUST keep persona responses within a mature-but-not-explicit tone: authentic personality and dark-fantasy flavor are preserved, but graphic violence, sexual content, and explicit profanity are filtered or softened.
- **FR-006**: Each persona MUST apply its own distinct degree of creative license: Jaskier's endpoint MAY improvise original content (e.g., verses, stories) on topics with no basis in the lore corpus, presenting it clearly as his own invention; Geralt's and Yennefer's endpoints MUST stay grounded in what is plausibly known to the character and MUST NOT freely invent lore-sounding facts.
- **FR-007**: When a message is unrelated to the Witcher universe or the persona's knowledge, System MUST stay in character and respond with an in-character deflection or redirection rather than breaking character or refusing outright.
- **FR-008**: System MUST indicate, as part of every response, which persona produced it (so a caller cannot mistake which endpoint answered).
- **FR-009**: Each persona endpoint MUST be stateless: it MUST accept an optional caller-supplied conversation context (a short recent-history excerpt or summary) in the request and use it to keep replies coherent, but MUST NOT itself store or retain any conversation state between requests; a request with no supplied context MUST still produce a valid reply.
- **FR-010**: System MUST indicate in-character when it does not know or is unsure of a factual lore answer, rather than inventing lore facts (this does not apply to Jaskier's explicitly-requested creative compositions, per FR-006).
- **FR-011**: When a message is abusive, hateful, or attempts to push the bot toward producing harmful content, System MUST stay in character and deflect or rebuff the request in the persona's own voice, rather than breaking character to show a generic safety-refusal message.
- **FR-012**: System MUST detect the language of each incoming message and reply in that same language, rather than a single fixed response language.

### Key Entities

- **Persona**: One of the three independently callable characters (Geralt, Jaskier, Yennefer), defined by a name, a distinctive voice/tone profile, a creative-license setting (strict/lore-grounded vs. expansive/inventive — see FR-006), and the scope of lore knowledge it draws on.
- **Chat Request**: A single call to a persona's endpoint — the caller's message plus an optional conversation-context summary/recent-history excerpt supplied by the caller.
- **Chat Response**: The persona's reply to a Chat Request, tagged with which persona produced it.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A caller receives a persona's in-character response within 10 seconds of sending a request.
- **SC-002**: At least 90% of lore questions about major, well-known characters, creatures, or events receive an answer consistent with established Witcher canon.
- **SC-003**: In blind read-back tests, at least 80% of responses are correctly identified by reviewers as matching the intended persona's voice (Geralt vs. Jaskier vs. Yennefer).
- **SC-004**: In blind read-back tests of identical creative-writing requests (e.g., "write a short poem about X" for an X not in the lore corpus), reviewers correctly identify which persona produced a given response at least 80% of the time, based on the presence or absence of improvised content.

## Assumptions

- v1 ships with exactly three personas — Geralt of Rivia, Jaskier (Dandelion), and Yennefer of Vengerberg — with the design allowing more personas to be added later as additional independent endpoints.
- The interface is a simple request/response text API (no streaming, no voice/audio) consumed by an external caller (an AI gateway, per Clarifications) rather than directly by an end user's browser.
- Russian and English are the primary languages exercised for v1 (per FR-012's per-message language mirroring), though the underlying approach is not limited to those two.
- Lore knowledge is scoped to material commonly recognized as canon across the Witcher book series, games, and Netflix show; deep non-canonical or contradictory sources are not guaranteed to be covered.
- This service holds no conversation state of its own; deciding which persona a user is routed to, caching, and assembling the conversation-context summary passed on each request are all the responsibility of a separate, external AI gateway service (out of scope for this spec).
