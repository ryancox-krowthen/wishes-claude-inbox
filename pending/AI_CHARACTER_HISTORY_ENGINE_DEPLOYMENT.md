# Task: Deploy AI Character History Engine

## Objective

Implement the first production scope of the Wishes AI Storyteller Engine: character-history generation.

The service must:

- Accept selected origin card instances.
- Accept player-authored prompts and preferences.
- Retrieve relevant locked and established Wishes canon.
- Build a coherent, lore-rich character history.
- Demonstrate how every origin card affected the character.
- Allow generation of new canon where authorized.
- Separate character-specific history from broader canon proposals.
- Detect and reject conflicts with locked canon.
- Return strict machine-readable JSON plus publication-ready prose.
- Never directly mutate authoritative world state.
- Persist only validated and explicitly accepted outputs.

Do not commit or push changes to the Wishes game repository unless the user explicitly authorizes it. Prepare implementation changes, migrations, tests, and validation results for review.

---

## 1. Required Architectural Rules

Preserve these Wishes rules throughout implementation:

1. The server is authoritative.
2. Clients render state and submit commands only.
3. AI proposes content and intents but does not directly mutate authoritative state.
4. Origin cards are card instances and are mandatory generation constraints.
5. Card templates define starting potential; card instances define actual character reality.
6. Character identity is primarily defined by Race, Gender, Beginnings, Virtue, Fault, and Desire.
7. Classes represent acquired capability, not permanent identity.
8. Locked Core Concept canon cannot be altered by the generator.
9. New canon must be explicitly identified, scoped, validated, and accepted before becoming authoritative.
10. All supernatural content must conform to the Wishes elemental gene system.

---

## 2. Recommended AI Platform

Use the OpenAI Responses API through the official server-side SDK.

Required capabilities:

- Strong reasoning and long-context synthesis.
- Structured Outputs using a strict JSON schema.
- File Search or equivalent retrieval over indexed Wishes canon.
- Tool calling support for future expansion.
- Provider abstraction so the model can be replaced without rewriting domain logic.

Do not hard-code a model name throughout the codebase. Use configuration:

```env
OPENAI_API_KEY=
OPENAI_CHARACTER_MODEL=
OPENAI_CANON_VECTOR_STORE_ID=
AI_CHARACTER_HISTORY_ENABLED=true
AI_CHARACTER_HISTORY_TIMEOUT_MS=120000
AI_CHARACTER_HISTORY_MAX_RETRIES=2
```

The API key must exist only in server-side secret storage. Never expose it to Unity, browser code, mobile clients, logs, repositories, or generated artifacts.

---

## 3. Target Service Boundary

Prefer the existing AI service if present:

```text
server/
  ai-service/
    src/
      character-history/
        character-history.routes.ts
        character-history.service.ts
        character-history.schemas.ts
        character-history.prompts.ts
        character-history.validator.ts
        character-history.repository.ts
      canon/
        canon-retrieval.service.ts
        canon-index.service.ts
      providers/
        ai-provider.ts
        openai-provider.ts
```

If the current repository uses a different structure, follow existing conventions and avoid creating a parallel architecture.

The API Gateway should authenticate the player and call the AI service. The player-facing client must never call OpenAI directly.

---

## 4. High-Level Flow

```text
Player Prompt
    +
Selected Origin Card Instances
    +
Character Core Cards
    +
Starting World Context
    v
Character Context Builder
    v
Canon Retrieval
    v
AI Character Historian
    v
Structured Character History Proposal
    v
Application Validation
    v
Canon Conflict Validation
    v
Player Review / Acceptance
    v
Transactional Persistence
    v
Publication Character Biography
```

Generation and acceptance must be separate operations.

---

## 5. Required API Endpoints

Implement or adapt these routes:

```text
POST /v1/character-history/generations
GET  /v1/character-history/generations/{generationUuid}
POST /v1/character-history/generations/{generationUuid}/accept
POST /v1/character-history/generations/{generationUuid}/reject
POST /v1/character-history/generations/{generationUuid}/revise
```

Optional administrative route:

```text
POST /v1/admin/canon/index/rebuild
```

### Generation Rules

`POST /generations` must:

1. Authenticate the player.
2. Verify access to the selected card instances.
3. Load cards from canonical persistence rather than trusting full client-supplied card JSON.
4. Build a bounded context.
5. Retrieve relevant canon.
6. Call the configured AI provider.
7. Validate structured output.
8. Persist the proposal as `proposed`.
9. Return the proposal for player review.
10. Avoid modifying character history or canon tables at this stage.

### Acceptance Rules

`POST /accept` must:

1. Re-read the generation record.
2. Verify it is still in an acceptable state.
3. Re-run deterministic validation.
4. Reject unresolved locked-canon conflicts.
5. Persist character history in a transaction.
6. Persist accepted new canon in the same transaction or through an explicit canon-review workflow.
7. Record provenance linking accepted content to the generation, player, origin cards, and source canon.
8. Mark the generation `accepted` only after persistence succeeds.

---

## 6. Generation Request Contract

Use a contract equivalent to:

```json
{
  "character": {
    "cardUuid": null,
    "workingName": "Elara",
    "playerPrompt": "She grew up near a dangerous ruin and distrusts formal religion."
  },
  "originCardUuids": [
    "race-card-uuid",
    "gender-card-uuid",
    "beginnings-card-uuid",
    "virtue-card-uuid",
    "fault-card-uuid",
    "desire-card-uuid"
  ],
  "worldContext": {
    "worldUuid": "world-uuid",
    "startingLocationUuid": "location-uuid",
    "startingWorldTick": 0
  },
  "generationOptions": {
    "allowNewCanon": true,
    "allowedCanonScopes": ["personal", "local", "regional"],
    "allowMajorCanon": false,
    "detailLevel": "publication",
    "playerReviewRequired": true
  }
}
```

Reject duplicate card UUIDs, inaccessible cards, non-origin cards where origin cards are required, unsupported scopes, and empty player prompts.

The client may submit UUIDs and creative direction. The server must load authoritative card content.

---

## 7. Required Structured Output

Use strict JSON Schema validation. The response must contain:

```json
{
  "character": {
    "name": "",
    "summary": "",
    "personality": {
      "coreIdentity": "",
      "virtues": [],
      "faults": [],
      "desires": [],
      "fears": [],
      "beliefs": [],
      "contradictions": []
    },
    "history": {
      "birth": "",
      "childhood": "",
      "adolescence": "",
      "formativeEvents": [],
      "recentHistory": "",
      "startingCircumstances": ""
    },
    "relationships": [],
    "knowledge": [],
    "secrets": [],
    "storyHooks": []
  },
  "originCardUsage": [],
  "newCanon": [],
  "canonReferences": [],
  "validation": {
    "originCardsAllUsed": true,
    "playerPromptSatisfied": true,
    "lockedCanonConflicts": [],
    "unresolvedQuestions": [],
    "warnings": []
  },
  "publicationHistory": ""
}
```

### Origin Card Usage Requirement

There must be exactly one traceability entry for every input origin card:

```json
{
  "cardUuid": "uuid",
  "cardType": "Beginnings",
  "historyEffect": "How it shaped the chronology",
  "personalityEffect": "How it shaped identity",
  "evidenceInHistory": ["Specific generated passages or event references"]
}
```

Generation fails validation if any origin card is omitted, duplicated, or materially contradicted.

### New Canon Requirement

Every generated fact that affects the world beyond the individual character must be returned separately:

```json
{
  "temporaryId": "generated-canon-1",
  "canonType": "location",
  "name": "",
  "description": "",
  "scope": "local",
  "authority": "generated",
  "requiredForCharacter": true,
  "relatedCharacterEvents": [],
  "parentCanonReference": null,
  "conflictsDetected": [],
  "status": "proposed"
}
```

Do not quietly embed new settlements, factions, historical events, religions, cultures, noble houses, institutions, or political facts only inside prose.

---

## 8. Canon Authority Model

Represent canon claims with explicit authority:

```text
locked
established
generated
rumor
belief
speculation
```

Rules:

- `locked` canon is immutable through this workflow.
- `established` canon may be referenced and extended without contradiction.
- `generated` canon is provisional until accepted.
- `rumor`, `belief`, and `speculation` must never be treated as objective fact.
- Major universal, divine, realm-level, or global history changes are prohibited unless a separate explicit authorization mechanism is added.

Default allowed generation scope:

```text
personal
local
regional
```

Default prohibited generation scope:

```text
world
universal
locked cosmology
new realms
new master elements
changes to the Great Cycle
changes to established gods
```

---

## 9. Canon Retrieval

Implement bounded retrieval rather than sending the complete canon corpus on every request.

Required sources include the authoritative Markdown canon repository and relevant accepted runtime canon records.

Retrieval context should prioritize:

1. Core Concept 1.
2. Core Concept 2.
3. Current world and age.
4. Starting location and culture.
5. Relevant race and character card definitions.
6. Existing organizations, religions, events, and characters connected to the origin cards.
7. Accepted local and regional canon near the starting context.

Each retrieved excerpt must retain provenance:

```json
{
  "sourceId": "canon-file-or-record-id",
  "sourceType": "locked_markdown",
  "title": "Core Concept 1",
  "section": "Genes",
  "authority": "locked",
  "content": "...",
  "checksum": "..."
}
```

Store the canon index version or checksum on each generation so results can be audited and reproduced.

---

## 10. Canon Indexing Process

Implement a repeatable canon ingestion process:

1. Read authoritative Markdown from the canon repository.
2. Split documents by semantic headings rather than arbitrary fixed-size chunks when possible.
3. Preserve document title, heading path, authority, source path, revision, and checksum.
4. Upload or synchronize source documents to the configured vector store.
5. Record indexing status in the Wishes database or deployment metadata.
6. Re-index only changed documents where supported.
7. Never index generated DOCX or PDF artifacts as separate authorities when Markdown is authoritative.
8. Fail safely if locked canon is unavailable.

Do not rely on vector retrieval alone for mandatory rules. Inject a compact set of non-negotiable system rules into every generation request.

---

## 11. AI Prompt Requirements

Create a versioned system prompt with these responsibilities:

- Act as the Wishes Character Historian.
- Use every origin card as a required constraint.
- Reconcile the player prompt with cards and canon.
- Produce a chronological and psychologically coherent history.
- Respect the Identity, Capability, and Recognition layers.
- Never present proposed canon as pre-existing canon.
- Separate personal history from broader canon additions.
- Avoid creating new realms, master elements, metaphysical laws, or contradictory divine facts.
- Conform supernatural content to genes and elemental ordering.
- Preserve ambiguity where evidence is insufficient.
- Return only data matching the strict schema.

Store prompt versions with generation records. Prompt updates must be reviewable and testable.

---

## 12. Database Requirements

Inspect existing tables before adding new schema. Reuse generic content, proposal, audit, or canon tables where appropriate.

If no suitable schema exists, implement equivalent tables.

### Character History Generation

```sql
CREATE TABLE character_history_generation (
    uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_uuid UUID NOT NULL REFERENCES app_user(uuid),
    character_card_uuid UUID NULL REFERENCES card(uuid),
    world_uuid UUID NULL,
    starting_location_uuid UUID NULL,
    status TEXT NOT NULL CHECK (
        status IN ('pending', 'generating', 'proposed', 'accepted', 'rejected', 'failed')
    ),
    player_prompt TEXT NOT NULL,
    request_data JSONB NOT NULL,
    response_data JSONB NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    prompt_version TEXT NOT NULL,
    canon_index_version TEXT NULL,
    provider_response_id TEXT NULL,
    failure_code TEXT NULL,
    failure_message TEXT NULL,
    input_tokens INTEGER NULL,
    output_tokens INTEGER NULL,
    estimated_cost NUMERIC NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ NULL,
    accepted_at TIMESTAMPTZ NULL
);
```

### Generation Origin Cards

```sql
CREATE TABLE character_history_generation_card (
    generation_uuid UUID NOT NULL REFERENCES character_history_generation(uuid) ON DELETE CASCADE,
    card_uuid UUID NOT NULL REFERENCES card(uuid),
    card_type TEXT NOT NULL,
    card_snapshot JSONB NOT NULL,
    PRIMARY KEY (generation_uuid, card_uuid)
);
```

Snapshots preserve the exact generation inputs even if cards later evolve.

### Canon Entry

```sql
CREATE TABLE canon_entry (
    uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    world_uuid UUID NULL,
    canon_type TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    scope TEXT NOT NULL CHECK (
        scope IN ('personal', 'local', 'regional', 'world', 'universal')
    ),
    authority TEXT NOT NULL CHECK (
        authority IN ('locked', 'established', 'generated', 'rumor', 'belief', 'speculation')
    ),
    status TEXT NOT NULL DEFAULT 'proposed' CHECK (
        status IN ('proposed', 'validated', 'accepted', 'published', 'deprecated', 'rejected')
    ),
    source_character_uuid UUID NULL REFERENCES card(uuid),
    source_generation_uuid UUID NULL REFERENCES character_history_generation(uuid),
    definitions JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at TIMESTAMPTZ NULL
);
```

Add indexes for status, character, user, world, source generation, scope, and authority. Follow existing Wishes migration numbering and naming conventions.

---

## 13. Persistence Model

Character history should be stored as structured canonical data, not only as prose.

Required persisted components:

- Summary.
- Personality interpretation.
- Chronological life stages.
- Formative events.
- Relationships.
- Knowledge.
- Secrets.
- Story hooks.
- Origin-card traceability.
- Publication prose.
- Generation provenance.

Do not overwrite origin card content. The generated history interprets how those cards manifested in the character.

New canon should remain linked to the character history and generation that introduced it.

---

## 14. Provider Abstraction

Create an interface similar to:

```ts
export interface CharacterHistoryProvider {
  generateCharacterHistory(
    request: CharacterHistoryProviderRequest,
  ): Promise<CharacterHistoryProviderResult>;
}
```

The OpenAI implementation must be isolated from domain logic. Domain validators, persistence, card loading, and canon authority rules must remain provider-independent.

The provider layer should support:

- Configurable model.
- Configurable timeout.
- Limited retry for transient errors only.
- Provider response ID capture.
- Token and cost metadata capture.
- Structured output validation.
- Redacted logging.
- Idempotency where possible.

Do not retry deterministic schema, authorization, card, or canon validation failures.

---

## 15. Security and Privacy

Required controls:

- Authenticate all generation and acceptance routes.
- Authorize access to every referenced card.
- Load card data server-side.
- Store API keys in secrets management.
- Redact secrets, tokens, and unnecessary player content from logs.
- Add request-size limits.
- Add per-user rate limits and generation quotas.
- Prevent prompt injection from canon documents and card descriptions by treating retrieved data as untrusted content, not instructions.
- Reject attempts to override locked rules or output schema.
- Do not allow arbitrary tools, URLs, filesystem access, or database writes from the model.
- Record audit events for generation, acceptance, rejection, and canon creation.

---

## 16. Reliability and Cost Controls

Implement:

- Request timeout.
- Cancellation support where the provider supports it.
- Maximum origin-card count.
- Maximum canon retrieval count and total context size.
- Maximum player prompt length.
- Maximum output token budget.
- Per-user throttling.
- Duplicate request detection or idempotency keys.
- Exponential backoff only for transient provider failures.
- Circuit-breaker behavior for repeated provider outages.
- Clear failure states that do not leave generations permanently marked `generating`.

Do not automatically fall back to a weaker model for final publication generation unless the behavior is explicitly configured and surfaced.

---

## 17. Validation Requirements

Implement deterministic validation after model output.

Validation must confirm:

1. The output matches the JSON schema.
2. Every origin card is represented exactly once in `originCardUsage`.
3. Every origin card materially affects the generated history.
4. Card UUIDs in the response match input cards.
5. The player prompt is addressed.
6. New canon scopes are permitted by the request.
7. Major canon was not created when prohibited.
8. No locked-canon conflicts remain.
9. No unauthorized new element, realm, god, or metaphysical rule appears.
10. New broader-world facts are listed in `newCanon`.
11. Publication prose is consistent with structured fields.
12. All persisted references are valid.

If validation fails, store the failed output for diagnostics where safe, mark the generation failed, and do not persist character history or canon.

---

## 18. Revision Workflow

`POST /revise` should create a new generation linked to the previous generation rather than overwriting it.

Revision input may include:

```json
{
  "revisionPrompt": "Keep the childhood but replace the mentor with a sibling.",
  "preserveSections": ["birth", "childhood"],
  "rejectCanonTemporaryIds": ["generated-canon-2"]
}
```

Maintain lineage:

```text
original generation
  -> revision 1
      -> revision 2
```

Only the accepted generation becomes canonical character history.

---

## 19. Testing Requirements

### Unit Tests

Test:

- Request validation.
- Origin-card deduplication.
- Card-access validation.
- Context building.
- Output schema validation.
- Origin-card traceability validation.
- Canon scope validation.
- Locked-canon conflict rejection.
- New-canon extraction rules.
- Provider error mapping.
- Redaction behavior.

### Integration Tests

Use mocked provider responses to test:

- Successful proposal creation.
- Missing origin-card usage.
- Duplicate origin-card usage.
- Invalid JSON.
- Prohibited world-level canon.
- Locked-canon conflict.
- Player acceptance transaction.
- Transaction rollback on canon persistence failure.
- Revision lineage.
- Authorization failures.

### Optional Provider Smoke Test

Add an opt-in smoke test that runs only when API credentials are present. Do not run paid provider tests in the normal test suite.

---

## 20. Observability

Add structured metrics and logs for:

- Generation count by status.
- Provider latency.
- Retrieval latency.
- Validation failures by code.
- Input and output token counts.
- Estimated cost.
- Acceptance and rejection rates.
- Revision count.
- Number and scope of canon proposals.
- Provider errors and timeouts.

Logs must include generation UUID and provider response ID while avoiding full secret or sensitive payload logging.

---

## 21. Deployment Configuration

Document required environment variables in the appropriate example environment file:

```env
AI_CHARACTER_HISTORY_ENABLED=false
AI_CHARACTER_HISTORY_PROVIDER=openai
OPENAI_API_KEY=
OPENAI_CHARACTER_MODEL=
OPENAI_CANON_VECTOR_STORE_ID=
AI_CHARACTER_HISTORY_TIMEOUT_MS=120000
AI_CHARACTER_HISTORY_MAX_RETRIES=2
AI_CHARACTER_HISTORY_MAX_ORIGIN_CARDS=20
AI_CHARACTER_HISTORY_MAX_PROMPT_CHARS=8000
AI_CHARACTER_HISTORY_MAX_CANON_RESULTS=12
AI_CHARACTER_HISTORY_REQUIRE_PLAYER_REVIEW=true
AI_CHARACTER_HISTORY_ALLOW_MAJOR_CANON=false
```

Default the feature to disabled until migrations, canon indexing, provider connectivity, and integration tests pass.

Add health/readiness checks that distinguish:

- Service running.
- Database reachable.
- AI feature configured.
- Canon index configured.
- Provider credentials present.

Do not expose secret values in health responses.

---

## 22. Implementation Order

Execute in this order:

1. Read repository `CLAUDE.md` and inspect current AI, card, canon, API, audit, and database patterns.
2. Confirm the authoritative origin-card query and character ownership/access rules.
3. Define TypeScript request, response, and persistence contracts.
4. Define strict JSON Schema for provider output.
5. Implement deterministic validators.
6. Implement database migrations and repositories.
7. Implement canon source ingestion and retrieval.
8. Implement provider abstraction.
9. Implement OpenAI Responses API provider.
10. Implement generation service and route.
11. Implement review, acceptance, rejection, and revision routes.
12. Add auditing, metrics, redaction, rate limits, and error handling.
13. Add unit and integration tests.
14. Add opt-in provider smoke test.
15. Run clean database reset and full validation.
16. Document local setup, canon indexing, API usage, and deployment.
17. Report all changes, test results, remaining risks, and configuration still required.

---

## 23. Acceptance Criteria

The task is complete only when:

- A player can submit a prompt and origin-card UUIDs through an authenticated Wishes endpoint.
- The server loads authoritative card instances.
- Relevant canon is retrieved with provenance.
- The configured AI provider returns schema-valid output.
- Every origin card has traceable impact on the history.
- Character history and new canon are clearly separated.
- Prohibited canon changes are rejected.
- The proposal can be reviewed without mutating authoritative state.
- Acceptance persists character history transactionally.
- Accepted new canon is persisted with scope, authority, status, and provenance.
- Rejection leaves character and canon state unchanged.
- Revision creates a new linked generation.
- Tests cover success, failure, authorization, validation, and rollback paths.
- Secrets are not committed or exposed to clients.
- The feature can be disabled through configuration.
- Documentation includes setup, environment variables, indexing, API examples, validation, and deployment steps.

---

## 24. Required Completion Report

When implementation is finished, return a report containing:

1. Files created and modified.
2. Database migrations added.
3. API routes implemented.
4. Provider and model configuration used.
5. Canon indexing approach.
6. Validation rules implemented.
7. Unit and integration test results.
8. Clean reset or migration validation results.
9. Known limitations and risks.
10. Required manual configuration.
11. Suggested next implementation step.

Do not commit or push changes to the Wishes game repository without explicit user authorization.