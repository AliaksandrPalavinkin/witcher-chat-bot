# Contract: Persona Chat REST API

Two independent, stateless persona endpoints. No authentication (this service is called by a trusted internal gateway, per spec Clarifications — not exposed directly to end users). No shared state between calls: everything a reply depends on must be in the request itself.

## `POST /api/v1/geralt/chat`

## `POST /api/v1/jaskier/chat`

Same request/response shape for both; only the persona (voice, temperature, creative-license behavior per FR-006) differs.

**Request**:
```json
{
  "message": "Tell me about the Wild Hunt.",
  "conversationContext": "User previously asked about Ciri and the Wild Hunt's pursuit of her."
}
```

- `message` (string, required, non-empty): the caller's current message.
- `conversationContext` (string, optional): a recent-history excerpt or summary supplied by the caller (typically an external gateway condensing the last ~10-15 turns) so the reply can stay coherent. Omit if there is no prior context.

**Response 200**:
```json
{
  "personaId": "GERALT",
  "content": "...",
  "language": "en",
  "wasFallback": false
}
```

- `personaId`: always the persona matching the endpoint called (`GERALT` or `JASKIER`).
- `content`: the persona's reply.
- `language`: language the reply is written in, mirroring the detected language of `message` (FR-012).
- `wasFallback`: `true` if `content` is the persona's in-character fallback line because the upstream LLM call failed or was rate-limited (FR-010/FR-011), rather than a normally-generated reply.

**Behavior notes**:
- Off-topic or abusive `message` values do not error — they produce an in-character deflection/rebuff `content`, returned the same way as any other reply (FR-007/FR-011), with `wasFallback: false`.
- On Jaskier's endpoint, a creative request about a topic absent from the lore corpus produces an improvised, clearly-in-character composition (FR-006); the same request on Geralt's endpoint produces a response consistent with his stricter, non-inventive nature (FR-006) — the two endpoints are expected to diverge here by design.
- No conversation state is created, updated, or read as a side effect of this call — repeating the same request produces an independent, freshly-generated reply.

**Errors**:
- `400` if `message` is empty or missing.
