# LangChain Persistent Memory: Implementation Plan

This file tracks the approved changes for adding persistent conversation memory and test-case generation history. Keep it updated before beginning each implementation phase. Do not start a phase marked **Awaiting approval** until the user approves it.

## Objective

Integrate LangChain-backed MongoDB conversation memory into the existing RAG test-case generation flow. Preserve the current RAG and baseline generation behavior while recording queries, generated cases, identifiers, and timestamps for future turns and sessions.

## Current Flow

- The generator UI lives in `client/src/components/processing/PromptSchemaManager.js` under **Prompt & Schema**.
- Baseline generation sends a composed prompt to `POST /api/test-prompt`.
- RAG generation preprocesses the user story, searches test cases and user stories, reranks/deduplicates/summarizes retrieval results, builds a prompt, then calls `POST /api/test-prompt`.
- `server/index.js` handles `/api/test-prompt` using `chatCompletion` from `src/scripts/utilities/llmService.js` and returns parsed output and model/token metadata.
- The current request does not include a conversation ID, and the generation route does not persist chat messages or generated test cases. Existing UI result caching is browser-local and is not conversation memory.
- The root `package.json` does not currently include LangChain packages.

## Phases

### Phase 1: Repository assessment and implementation plan

**Status: Complete**

- Trace the existing generation UI, RAG retrieval stages, LLM endpoint, and MongoDB configuration.
- Record the design and approval gates in this document.
- No application source code or dependencies changed in this phase.

### Phase 2: Persistent memory and history foundation

**Status: Complete**

Implemented:

- Added `@langchain/core` 1.2.18 and `@langchain/mongodb` 1.3.1; they resolve against MongoDB driver 6.20.0 and Node 24.
- Added `src/scripts/utilities/conversationMemoryService.js` with injectable LangChain MongoDB chat history and generation-history helpers.
- Added dedicated configurable collections: `MEMORY_MESSAGES_COLLECTION_NAME` defaults to `conversation_memory`; `GENERATION_HISTORY_COLLECTION_NAME` defaults to `test_case_generation_history`. The service rejects using the same collection or reusing the configured test-case/user-story collections.
- Added UUID creation for conversation, execution, and memory records. LangChain messages carry memory-record and execution IDs in message metadata; generation execution records store their execution UUID as MongoDB `_id`.
- Generation records support the query, generated test cases, generation mode, provider/model, status/error, correlated memory-record IDs, and ISO timestamps.
- Added `npm test` and six focused `node:test` cases using fake collections; no live MongoDB or LLM request is required for these tests.
- Validation: `npm test` passed all 7 discovered tests (including the existing preprocessing test); editor diagnostics reported no errors.
- No HTTP route was added in Phase 2; API integration and its Postman requests remain in Phase 3.

### Phase 3: Generator API integration

**Status: Complete**

Implemented:

- Extended `POST /api/test-prompt` with optional `conversationId`, `userQuery`, and `generationMode`; legacy callers receive a new UUID conversation while retaining the existing prompt/messages contract.
- Loaded and passed prior LangChain messages to the existing LLM adapter. History defaults to 10 messages and 12,000 characters, is configurable with `MEMORY_MAX_MESSAGES` and `MEMORY_MAX_CHARS`, and is additionally capped against the estimated remaining token budget.
- Initialized a unique `sessionId` index on conversation memory and a `{ conversationId, createdAt }` index on generation history.
- Persisted user/assistant turns and execution records for successful and failed generations. Responses include conversation ID, execution ID, correlated memory-record IDs, history count, and generation status.
- Added `postman/langchain-memory.postman_collection.json` and `postman/langchain-memory.postman_environment.json`; the environment contains local/non-secret variables only. Requests cover API health, first turn, same-conversation continuation, and a new isolated conversation.
- Added offline tests for context continuity/limits, successful persistence, and failed-generation status.
- Validation: `npm test` passed all 12 tests; server syntax check, editor diagnostics, Postman JSON, and embedded test scripts passed.
- Live Postman generation requests were not run because they require configured MongoDB and LLM provider access.

### Phase 4: UI conversation lifecycle

**Status: Complete**

Implemented:

- Persisted/restored the active conversation ID and latest generation identifiers/status in browser storage.
- Passed the active conversation ID, original user query, and generation mode with both RAG and baseline test-case generation requests. A synchronized ref keeps sequential “Run Both Modes” requests in the same conversation.
- Added a new-conversation action that starts a fresh conversation on the next generation without deleting MongoDB history or clearing the displayed results.
- Added a conversation/status strip to the existing Prompt & Schema screen, showing the active ID, last execution, mode, and status.
- Marked the AI comparison request `persistMemory: false` and added an API opt-out so analysis-only prompts do not enter test-case generation history.
- No history-browsing menu was added in Phase 4; that capability is implemented separately in Phase 6.
- Validation: client production build passed; all 12 Node tests passed; server syntax check, editor diagnostics, and browser rendering passed. Live generation was not exercised because it requires configured MongoDB and LLM services.

### Phase 5: Verification and reference documentation

**Status: Complete**

Implemented and verified:

- Kept the root `README.md` scoped to the original application as a clean baseline, per user direction; LangChain-specific implementation, API, Postman, and recovery notes remain in this plan.
- `npm test`: all 12 tests passed.
- `npm run build`: passed; Vite reports the existing large-bundle-size warning.
- `node --check server/index.js`, API health (`OK`), Postman JSON/environment/scripts, and 24 local README links passed validation.
- Editor diagnostics reported no errors in the changed source and Markdown files.
- Live MongoDB/LLM generation was not exercised; the Postman generation sequence requires configured services and may incur provider usage.

### Phase 6: Generation history browser

**Status: Complete**

Implemented and verified:

- Added read-only `GET /api/generation-history` with bounded pagination (up to 100 rows/page and page offsets capped at 100,000) and optional conversation ID, generation mode, and status filters. List responses omit generated-test-case payloads.
- Added read-only `GET /api/generation-history/:executionId` to return a full run by its UUID `_id`, including query and generated test cases.
- Added a dedicated **Generation History** sidebar screen with conversation/mode/status filters, pagination, loading/empty/error states, refresh, and a detail dialog.
- Extended the Postman collection with recent-list, filtered-list, and detail requests.
- No update/delete actions or changes to existing history records were added.
- Validation: `npm test` passed all 15 tests; client build, API syntax/health, editor diagnostics, and Postman JSON/scripts passed. The build continues to show the existing large-bundle warning.
- Live history endpoints were not called because that would query the configured MongoDB collection; Postman generation/history sequences require configured services.

### Phase 7: Search history persistence

**Status: Complete**

Implemented and verified:

- Added blank `SEARCH_HISTORY_COLLECTION_NAME=` to local `.env` for the user to set, and `SEARCH_HISTORY_COLLECTION_NAME="search_history"` to `.env.example`; code defaults to `search_history` when unset.
- Search history uses a collection distinct from test cases, user stories, conversation memory, and generation history.
- Successful vector, BM25, hybrid, rerank, and user-story search calls now persist a UUID, type, query, sanitized filters, result count, up to 50 result IDs/scores, and ISO timestamp. Full result documents are not copied.
- Successful search responses include `searchHistoryId`; history write failures fail the request visibly.
- Added a Postman BM25 search request that asserts/captures `searchHistoryId`, plus focused tests for collection isolation, metadata bounds, and input validation.
- Validation: `npm test` passed all 18 tests; client build, API syntax/health, Postman validation, environment-key presence check, and editor diagnostics passed. Vite continues to report the existing large-bundle warning.
- No live search requests were sent to MongoDB during verification.

## Design Constraints and Risks

- Do not change the existing test-case and user-story collections or their Atlas Search indexes.
- A new conversation should be isolated from prior messages; reusing its conversation ID should restore context after a browser refresh or new app session.
- Bound the number/size of retrieved history messages so adding memory does not silently exceed configured LLM token limits.
- Do not store API keys or raw environment configuration in history records.
- The current `/api/env` route exposes environment values; this implementation must not copy those values into client-facing memory responses. Broader route hardening is outside this feature unless separately approved.
- RAG-retrieved reference documents remain distinct from conversation history and should continue to be supplied by the existing retrieval pipeline.

## Approval Gate

Phases 1–7 are complete. No subsequent phase is currently approved.
