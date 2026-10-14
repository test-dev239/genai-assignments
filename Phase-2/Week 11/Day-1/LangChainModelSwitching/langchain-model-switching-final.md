# LangChain Model Switching: Implemented Details

## Status

This document records the implementation currently present in the repository. Phases 0–6 are implemented and automated/local rollout checks pass. Live-provider smoke tests remain pending until the exposed API credentials have been rotated.

## Phase-wise Implementation

### Phase 0: Prerequisites and Baseline

- Confirmed the repository `.gitignore` excludes `.env`.
- Corrected `backend/tsconfig.json` from `ignoreDeprecations: "6.0"` to `"5.0"` to match the installed TypeScript 5.9 compiler; backend typecheck and build then passed.
- Removed the Groq key fragment and request/response body logging, and removed raw Inception error-body logging.
- **Still required before live calls:** revoke and rotate credentials that were exposed in earlier diagnostics/attachments. Replacement keys must stay local.

### Phase 1: LangChain Adapter

- Added `@langchain/core` and `@langchain/openai` to the backend.
- Added `LangChainClient`, implementing the existing `LLMClient` contract with `ChatOpenAI` and OpenAI-compatible provider URLs.
- Normalized response text and token usage; added mock injection for offline tests.
- Kept the existing direct provider clients available during the adapter stage.

### Phase 2: Provider Registry and Configuration

- Added `modelRegistry.ts` to resolve ordered provider/model candidates and their credentials/endpoints.
- Added `LLM_TASK_ROUTES` JSON configuration, startup validation, and legacy `LLM_PROVIDER` single-candidate compatibility.
- Added tests for route ordering, defaults, malformed configuration, unsupported providers, and missing credentials.

### Phase 3: Task Selection and Generate Integration

- Added `modelRouter.ts` and connected `POST /api/generate-tests` to the `testGeneration` task.
- The router reports the selected provider/model, and the route uses those values for response metadata and cost calculation.
- At this stage the router selected the primary candidate; retry/fallback behavior was added in Phase 4.

### Phase 4: Retry and Fallback

- Added bounded retries for rate limits, network errors, and transient HTTP failures; added `Retry-After` handling and a 30-second overall deadline.
- Added immediate fallback for unavailable models and authentication errors (401/403), without retrying that candidate; invalid-request errors still stop immediately.
- Disabled LangChain's internal retries so the router owns the attempt budget.
- Added sanitized failure classification/logging; raw provider bodies and credentials are not logged.
- Invalid JSON or schema-invalid output returns an error and does not currently trigger model fallback.

### Phase 5: Generate Route Contract Tests

- Added an injectable Express router factory for tests without changing the production route contract.
- Added HTTP-level tests for input validation, response metadata, token totals, pricing, unknown-price behavior, invalid JSON/schema output, and sanitized provider failures.

### Phase 6: Final Verification

- All 25 backend tests passed, including authentication failover from an Inception candidate to a following Groq candidate.
- Workspace typechecks and backend/frontend production builds passed.
- Combined dev servers started and `GET /api/health` returned HTTP 200.
- Live-provider requests were intentionally not made. Complete them only after rotating the exposed keys.

## Request Flow

1. The frontend submits a story to `POST /api/generate-tests`.
2. The Express route validates the request, builds the system and user prompts, and calls `generateWithTaskModel('testGeneration', ...)`.
3. `modelRegistry.ts` validates configured provider/model candidates and resolves each candidate's API key and base URL.
4. `modelRouter.ts` selects the first candidate for `testGeneration` and creates its `LangChainClient`.
5. The LangChain adapter invokes the configured OpenAI-compatible chat-completions endpoint.
6. Retryable or unavailable-model failures may move execution to the next configured candidate under the policy below.
7. The route parses and validates the returned JSON with the existing response schema, computes cost from the successful provider/model, and returns the existing response structure.

The frontend does not receive provider credentials or make provider-specific calls. The generate API response retains `cases`, token counts, and model/provider/cost metadata, and includes `fallbackUsed` plus `fallbackInfo` to identify a secondary candidate and the model/provider it replaced. This fallback metadata is available in the API response, not displayed in the UI.

## Dependencies and Adapter

Backend dependencies:

- `@langchain/core` `^1.2.17`
- `@langchain/openai` `^1.6.2`

`backend/src/llm/langchainClient.ts` implements the existing `LLMClient` interface. It:

- Creates `ChatOpenAI` with the configured API key, base URL, and model.
- Uses temperature `0.2` and a `4096` token output limit.
- Disables the SDK's own retries (`maxRetries: 0`) so the model router controls retry limits.
- Passes a remaining-time timeout and abort signal to each invocation.
- Normalizes string or text-block content and token usage to `LLMResponse`.

The existing direct Groq and Inception clients remain in the repository, but the generate route now uses the LangChain adapter.

## Model Configuration

Set ordered candidates as JSON in the backend root `.env`:

```dotenv
LLM_TASK_ROUTES={"testGeneration":["groq:openai/gpt-oss-120b","inception:mercury-2.5"]}
```

The first candidate is primary. Every configured provider needs its corresponding API key present at startup. Keys are read from `groq_API_KEY` and `INCEPTION_API_KEY`; base URLs and model defaults use the existing provider-specific settings.

If `LLM_TASK_ROUTES` is absent, the registry preserves the old configuration: `LLM_PROVIDER` plus the selected provider's model variable produce one `testGeneration` candidate. The provider defaults to Groq when `LLM_PROVIDER` is unset.

Startup rejects malformed route JSON, empty routes, unknown providers, missing model names, and missing credentials. Configuration errors do not print key values.

## Retry and Fallback Policy

The model router allows at most two retries per candidate and applies a 30-second total deadline to the request. LangChain's internal retries are disabled to avoid nested, unbounded retry behavior.

| Failure | Behavior |
|---|---|
| HTTP 429 | Retry the same candidate within its retry limit. Honor `Retry-After` as seconds or an HTTP date; if the wait exceeds the remaining deadline, move to the next candidate. |
| Network errors, HTTP 408/425, or HTTP 5xx | Retry with exponential backoff starting at 250 ms, then move to the next candidate if retries are exhausted. |
| HTTP 404 or recognized unavailable-model code | Move to the next candidate immediately. |
| HTTP 401/403 | Move to the next candidate immediately without retrying the failed candidate. |
| Invalid request or unclassified error | Stop immediately rather than masking the error with a fallback. |
| Invalid JSON or schema-invalid output | Return the existing invalid-output error; output validation does not currently trigger model fallback. |

After all candidates are exhausted, the route returns the existing generic 502 error. Backend retry logs contain provider, model, attempt, failure category, and status where available; raw provider bodies and API keys are not logged.

## Cost and Response Metadata

The router returns the provider and model that succeeded, plus `fallbackUsed` (`false` for the primary candidate, `true` for any later candidate). `fallbackInfo` is `null` when the primary succeeds; after fallback it identifies the source provider/model, destination provider/model, reason, and HTTP status when available. The generate route uses the selected provider/model for response metadata and `calculateCostUSD`. The existing pricing table remains authoritative; unknown provider/model rates return `null` rather than an estimate.

## Verification

Automated backend tests use an injected fake chat model and do not require provider credentials. They cover:

- LangChain message/content/usage normalization.
- Legacy and JSON route configuration, ordering, defaults, and invalid configuration.
- Primary model selection and task-route errors.
- `Retry-After`, bounded 429 retries, network and 5xx backoff, unavailable-model fallback, and request deadlines.
- Authentication failover without retries, invalid-request stop behavior, and sanitized exhausted errors.
- Generate endpoint request validation, successful provider/model metadata, `fallbackUsed` and `fallbackInfo`, token totals, pricing, unknown-price handling, invalid JSON/schema responses, and sanitized provider failures.

Verified commands:

```powershell
npm test --workspace=backend
npm run typecheck
npm run build --workspace=backend
npm run build --workspace=frontend
npm run dev
```

Phase 6 verification before the authentication-policy update: 24 backend tests passed, workspace typechecks passed, and both backend and frontend production builds passed. After the policy update, all 25 backend tests passed and backend typecheck/build passed. The combined development servers started successfully, and `GET /api/health` returned HTTP 200. Tests use fake model responses and do not require provider credentials. No live-provider request was made; rotate the exposed keys before manually testing either provider.

## Security

Credentials previously included in `.env` diagnostics or attachments must be revoked and rotated before live testing. Keep replacement keys only in the local root `.env`; do not commit or share them. `.env` is listed in the repository `.gitignore`.