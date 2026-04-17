# Architecture: API Endpoint Executability Validator

## 1. Design Overview

The agent implements a **Wave-Parallel Multi-Agent pattern**:

```
runAgent()  (orchestrator)
  ├── buildDependencyGraph(endpoints)   → Map<slug, providerSlug | null>
  ├── topologicalSort(depGraph)         → [wave0[], wave1[], ...]
  ├── ContextStore = Map<string, string[]>   (shared ID registry)
  └── for each wave (sequentially):
        outcomes = await Promise.all(wave.map(ep => testEndpoint(ep, store)))
        update ContextStore from successful responses
        accumulate results
  └── compile and return TestReport
```

**"One agent per endpoint"** is realized by `testEndpoint()` — an independent, self-contained async function that makes its own request construction and classification decisions, runs concurrently with other endpoints in the same wave via `Promise.all`, and does not mutate shared state (read-only access to the ContextStore).

**Sequential waves** satisfy dependency ordering without hardcoding execution order — the order is derived dynamically by the topological sort.

---

## 2. Execution Waves

Waves are computed at runtime from the dependency graph. For the 16 test endpoints:

| Wave | Endpoints | Reason |
|------|-----------|--------|
| 0 | LIST_MESSAGES, SEND_MESSAGE, LIST_LABELS, GET_PROFILE, CREATE_DRAFT, LIST_THREADS, LIST_FOLDERS, LIST_EVENTS, CREATE_EVENT, LIST_CALENDARS, LIST_REMINDERS | No path parameters — can run immediately |
| 1 | GET_MESSAGE, TRASH_MESSAGE, ARCHIVE_MESSAGE, GET_EVENT, DELETE_EVENT | Require `{messageId}` or `{eventId}` from Wave 0 results |

Wave 0 has 11 endpoints running in parallel. Wave 1 has 5 running in parallel. Total concurrency is ~11x vs. a sequential loop at best.

---

## 3. Generic Dependency Resolution

**No hardcoded app names, paths, or IDs.**

### Step 1: Detect path parameters
```
detectPathParams("/gmail/v1/users/me/messages/{messageId}") → ["messageId"]
```

### Step 2: Derive base resource name
```
resourceNameFromParam("messageId") → "message"   (strip "Id" suffix)
resourceNameFromParam("eventId")   → "event"
resourceNameFromParam("user_id")   → "user"       (strip "_id" suffix)
```

### Step 3: Find a provider endpoint
For each path parameter, find the first GET endpoint where:
- It is not the endpoint being analyzed
- It has no path parameters of its own (it can run in Wave 0)
- Its `tool_slug` contains both the resource name AND the word "list", OR its `path` contains the resource name as a path segment

### Step 4: ContextStore population
After each successful response in a wave, `extractIdsFromResponse()`:
- Scans all top-level array fields for objects with an `"id"` field
- Singularizes the array key: `"messages"` → `"message"`, `"events"` → `"event"`
- Stores the IDs under both camelCase and snake_case keys:
  - `"messageId"` and `"message_id"`
- Also handles single-resource responses (e.g. `CREATE_EVENT` returning `{ id: "...", summary: "..." }`)
- Stores at most 5 IDs per key to cap memory usage

Wave 1 endpoints call `buildRequest()` which looks up their `{paramName}` directly in the store.

---

## 4. Request Body Construction

Body fields are constructed from each field's `name`, `type`, and `description`:

| Field pattern | Generated value |
|---------------|----------------|
| `type=string` + description mentions "RFC 2822" or name=`"raw"` | Valid base64url-encoded RFC 2822 email |
| `type=string` + description mentions "RFC3339" or "dateTime" | `new Date(now + 1 day).toISOString()` |
| `type=object` + name=`"start"` | `{ dateTime: <tomorrow ISO>, timeZone: "UTC" }` |
| `type=object` + name=`"end"` | `{ dateTime: <tomorrow+1h ISO>, timeZone: "UTC" }` |
| `type=object` + name=`"message"` | `{ raw: <base64url email> }` |
| `type=integer` | `1` |
| `type=boolean` | `false` |
| `type=string` (default) | `"test-value"` |

This generalizes to any app: only field metadata (names, types, descriptions) is needed — no app-specific knowledge.

---

## 5. Avoiding False Negatives

A **false negative** is classifying a valid endpoint as `invalid_endpoint` because our request was constructed incorrectly. Three safeguards:

**Safeguard 1: 400 → "error", not "invalid_endpoint"**
A 400 Bad Request means the endpoint exists but our request body was flawed. We conservatively classify it as `"error"` rather than `"invalid_endpoint"`. Only 404 and 405 are classified as `invalid_endpoint`.

**Safeguard 2: Unresolved path params → "error", not called**
If `{messageId}` cannot be resolved from the ContextStore, we do not call the endpoint with a fake/garbage ID (which would produce a 404 on a real endpoint). Instead we report `"error"` with a clear explanation that the path param was unresolvable.

**Safeguard 3: Semantic body construction**
We parse field descriptions for RFC-standard hints (RFC 2822, RFC3339) and construct precisely correct values. An incorrect body on a POST endpoint would yield a 400 → `"error"`, not `"invalid_endpoint"`, preserving the endpoint's potential validity.

---

## 6. Classification Logic

| HTTP Status | Classification | Reasoning |
|------------|---------------|-----------|
| 2xx | `valid` | Endpoint exists and request succeeded |
| 401, 403 | `insufficient_scopes` | Endpoint exists; auth/permissions missing |
| 404, 405 | `invalid_endpoint` | Path or method does not exist in the API |
| 400 | `error` | Endpoint exists; our request body was flawed |
| 5xx | `error` | Server-side error |
| Network exception | `error` | Timeout, connection failure |
| Unresolved path param | `error` | Endpoint not called; cannot determine validity |

---

## 7. Known Fake Endpoints

Three endpoints in the test set do not exist in the real APIs:

- `GMAIL_LIST_FOLDERS` (`GET /gmail/v1/users/me/folders`) — Gmail has no `/folders` endpoint; use `/labels` instead
- `GMAIL_ARCHIVE_MESSAGE` (`POST /gmail/v1/users/me/messages/{messageId}/archive`) — archiving in Gmail is done via `POST /messages/{id}/modify` with `removeLabelIds: ["INBOX"]`
- `GOOGLECALENDAR_LIST_REMINDERS` (`GET /calendar/v3/calendars/primary/reminders`) — this endpoint does not exist in the Google Calendar API

All three will return 404 → correctly classified as `invalid_endpoint`.

---

## 8. GOOGLECALENDAR_DELETE_EVENT

This endpoint needs a real `{eventId}` to delete. The design handles it without special-casing:

1. `GOOGLECALENDAR_CREATE_EVENT` is in Wave 0 (no path params, POST with body)
2. The body is auto-constructed: `{ summary: "test-value", start: {...}, end: {...} }`
3. CREATE returns 201 with `{ id: "abc123", ... }`
4. `extractIdsFromResponse` sees the top-level `id` field → stores `"eventId": ["abc123"]`
5. `GOOGLECALENDAR_DELETE_EVENT` is in Wave 1 → resolves `{eventId}` to `"abc123"` → DELETE succeeds → `valid`

The newly created test event is immediately deleted by the DELETE test — no permanent side effect.

---

## 9. Architecture Pattern

**Pattern: Orchestrator + Wave-Parallel Sub-Agents**

The `runAgent()` orchestrator handles global concerns (dependency graph, wave ordering, store management) while each `testEndpoint()` call is an independent, stateless (read-only store) unit. This is the "map-reduce over dependency waves" pattern.

**Why not a single sequential loop?**
Parallelism within waves reduces total runtime significantly. 11 Wave 0 endpoints running in parallel takes roughly the same time as the slowest one alone.

**Why not full parallelism (all 16 at once)?**
Endpoints with path parameter dependencies would fail without real IDs from prior endpoints, causing false "error" classifications.

**Why not an LLM agent loop?**
The core classification decision is deterministic (HTTP status codes). LLM reasoning is better suited for body construction in cases where field descriptions are rich but ambiguous — a viable future enhancement, but unnecessary for correctness here.

---

## 10. Tradeoffs

**What works well:**
- Generic dependency resolution scales to any REST API following standard naming conventions
- Wave-parallel execution minimizes total runtime
- Conservative classification prevents false negatives
- No external LLM API required — all logic is deterministic
- ContextStore is read-only during wave execution, eliminating concurrent mutation bugs

**Known limitations:**
- Body construction quality depends on field description text; sparse descriptions may result in 400s (classified as `"error"`, not false negatives)
- Singularization is naive (strip trailing "s") — irregular plurals would fail
- Dependency resolution picks the first matching provider; if multiple providers exist, only one is tracked

**Improvements:**
- **LLM-assisted body construction** — `buildBodyWithLLM()` calls Groq (`llama-3.1-8b-instant`) with the endpoint description and field definitions to generate a valid request body. Falls back to the heuristic builder if Groq is unavailable or returns invalid JSON. Eliminates reliance on hardcoded field-name patterns for exotic endpoints.
- **Exponential backoff retry** — `proxyExecuteWithRetry()` wraps every `proxyExecute` call. Retries on 5xx or network exceptions with delays of 1 s and 2 s (max 2 retries). 4xx responses are not retried.
- **Explicit CREATE-before-DELETE dependency** — `buildDependencyGraph` now detects DELETE endpoints and wires them to the corresponding POST (CREATE) endpoint rather than a LIST endpoint. This guarantees a freshly created resource ID is in the ContextStore before DELETE runs, eliminating the GET vs. DELETE race condition.

**Remaining improvements with more time:**
- Configurable concurrency cap for large endpoint sets (rate limiting)
