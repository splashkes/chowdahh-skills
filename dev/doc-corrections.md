# Concrete Documentation Corrections

This document turns the review findings into specific documentation changes for this repo.

Scope:
- Repo reviewed: `/Users/splash/chow_claw`
- Live API tested against: `https://chowdahh.com`
- Test date: 2026-04-13 / 2026-04-14 America/Toronto

## What Was Verified

The following public endpoints responded successfully during review:

- `GET /api/v1/streams/top`
- `GET /api/v1/search`
- `POST /api/v1/feed-sessions`
- `GET /api/v1/feed-sessions/{id}`
- `POST /api/v1/feed-sessions/{id}/more`
- `PATCH /api/v1/feed-sessions/{id}/controls`
- `POST /api/v1/radio-sessions`
- `GET /api/v1/radio-sessions/{id}`
- `PATCH /api/v1/radio-sessions/{id}`
- `GET /audio/{track_id}`

The following auth-gated endpoints rejected anonymous access as expected:

- `GET /api/v1/replay`
- `GET /api/v1/stats/activity`
- `GET /api/v1/preferences/{person_id}`
- `PUT /api/v1/preferences/{person_id}`

## Concrete Mismatches To Fix

### 1. Guidance contract is overstated

Current repo claim:
- README and skill docs imply the API teaches the next step through `guidance.next_best_actions`.

Observed live behavior:
- `guidance` is present on successful reads.
- `next_best_actions` is not consistently present.
- When `next_best_actions` is present, some `api_hint.path` values are emitted as `/v1/...` instead of `/api/v1/...`.
- Calling those `/v1/...` paths did not work.

Impact:
- An agent that trusts the hint path literally can break on the next request.

Required doc change:
- Stop promising that every response includes actionable next-step hints.
- Treat `next_best_actions` as optional.
- State clearly that clients should normalize hint paths against the documented version prefix until the server emits stable paths.

### 2. Response envelope and object shapes are inaccurate

Current repo claim:
- `Every response uses {data, guidance, meta}`
- Feed session returns `cards[]`
- Card object uses `card_id`

Observed live behavior:
- `POST /api/v1/feed-sessions` returned `items[]`, not `cards[]`.
- Returned objects used `id`, not `card_id`.
- Some successful responses had `guidance` without `next_best_actions`.
- Some validation failures returned only `{error, meta}` with no `guidance`.

Impact:
- A client generated from the docs will parse the wrong fields.
- Error-handling logic will over-assume the presence of `guidance`.

Required doc change:
- Update examples to match `items[]` and `id`.
- Mark `guidance.next_best_actions` as optional.
- Mark `guidance` on error responses as not yet universal in production.

### 3. Search-to-drilldown flow is not currently discoverable

Current repo claim:
- Search covers topics, sources, curators, and collections.
- Topic and curator drill-down endpoints are advertised as follow-on actions.

Observed live behavior:
- Anonymous search results did not expose result types.
- Anonymous search results did not expose usable topic IDs or curator IDs.
- Some results had malformed short values in `canonical_url` / `share_url` instead of absolute URLs.
- Some results had empty `headline` values.
- Looking up visible topic names such as `NASA` or `Artemis II` returned empty timeline payloads rather than a clear drill-down path from search.

Impact:
- The docs currently describe a flow that an anonymous client cannot execute from the data it receives.

Required doc change:
- Do not present anonymous search as a reliable entry point into topic/curator drill-down.
- Clarify that drill-down endpoints require a true internal topic/curator identifier, and that those identifiers are not consistently exposed in current anonymous search responses.
- Note that some search result URLs may need normalization before use.

### 4. Write-path validation behavior is inconsistent

Current repo claim:
- Errors follow a consistent envelope.
- Enumerated signal types imply server-side validation.

Observed live behavior:
- Invalid signal payload returned `200` with `recorded: 0`.
- Invalid feedback payload returned `400` with `{error, meta}` only.
- Invalid single-item submission returned `400`.
- Invalid collection submission returned `201` with `accepted: 0` and per-item skipped reasons.

Impact:
- Callers cannot rely on status code alone to determine whether a write took effect.
- Docs currently understate the need to inspect response counters and per-item results.

Required doc change:
- Add explicit client guidance: inspect `recorded`, `accepted`, and per-item result arrays.
- Remove the implication that all invalid writes surface as non-2xx validation errors.

## File-By-File Changes

### README.md

Replace the strong guidance-envelope promise with this:

```md
**The guidance envelope.** Most API responses include a `guidance` block describing what happened, and some responses also include `next_best_actions` with follow-up suggestions. Clients should treat those next-step hints as optional and continue to rely on the documented `/api/v1` endpoint paths as the source of truth.
```

Replace the “How Agents Use It” flow with this:

```md
1. Agent installs `chowdahh`
2. User says "what's happening today?"
3. Agent calls `POST /api/v1/feed-sessions` with `{"intent": "browse", "budget_minutes": 5}`
4. API returns feed items, controls, and request metadata
5. If `guidance.next_best_actions` is present, agent may use it as a suggestion layer
6. Otherwise the agent should continue with the documented `/api/v1` endpoints directly
7. User says "more science" and the agent applies the returned control chip if available
```

Add one note under “Try It Right Now” or “More”:

```md
Note: the live production API currently has some response-shape drift versus these docs. The `chowdahh/references/api-guide.md` file should be treated as the canonical contract this repo intends to publish, but clients should still validate response fields defensively.
```

### chowdahh/SKILL.md

Replace this sentence:

```md
Read `guidance.next_best_actions` after every call. The API teaches you the conversation flow.
```

With this:

```md
Read `guidance.next_best_actions` when present. Treat it as a suggestion layer, not a guaranteed contract, and keep using the documented `/api/v1` routes as the source of truth.
```

Update the feed-session example shape:

```json
{
  "data": { "session_id": "abc-123", "items": [...], "count": 6, "controls": {...} },
  "guidance": {
    "status_explanation": "Feed session started with 6 items.",
    "next_best_actions": [
      {
        "action_id": "send_more",
        "title": "Send more cards",
        "api_hint": { "method": "POST", "path": "/api/v1/feed-sessions/abc-123/more" }
      }
    ]
  }
}
```

### chowdahh/references/api-guide.md

Replace the response-envelope section with this:

```md
## Response Envelope

Successful responses currently use `data` and `meta`, and usually include `guidance`.

`guidance.next_best_actions` is optional.

Production note: some validation errors currently return `{error, meta}` without a `guidance` block, so clients should parse error responses defensively rather than assuming a universal envelope.
```

Replace the feed-session return description:

```md
Returns: `session_id`, `items[]`, `count`, `controls`.
```

Replace the card object heading and example:

```md
## Feed Item Object

```json
{
  "id": "...",
  "headline": "TSMC accelerates 1.4nm timeline",
  "summary": "Taiwan Semiconductor...",
  "image_url": "https://...",
  "topics": ["science", "business"],
  "source_count": 4,
  "share_url": "https://chowdahh.com/x/abc"
}
```
```

Add this note under Search:

```md
Production note: anonymous search results do not currently expose stable result types or drill-down IDs for every result. Treat search primarily as a browse surface unless the response includes an explicit identifier you can use safely.
```

Replace the topic/curator drill-down intro with this:

```md
These endpoints require true topic or curator identifiers. Current anonymous search responses do not reliably expose those identifiers, so clients should not assume a direct anonymous search -> drill-down flow.
```

Add this note under Signals:

```md
Production note: callers should inspect `recorded` in the response body. Invalid or unrecognized signals may be ignored rather than rejected with a non-2xx status.
```

Add this note under Submissions:

```md
Production note: collection submissions may succeed at the request level while skipping individual items. Inspect `accepted`, `results[]`, and per-item statuses before treating the submission as complete.
```

Add this note under Feedback:

```md
Production note: feedback validation failures may currently return `{error, meta}` without a `guidance` block.
```

## Suggested Priority

1. Fix the `/v1/...` vs `/api/v1/...` hint mismatch in the docs immediately.
2. Fix the response-shape examples next, because those affect all client implementations.
3. Remove the anonymous search-to-drilldown promise until the live API exposes stable IDs.
4. Add write-path caveats so agents do not misreport dropped writes as success.

## Minimal Acceptance Pass After Doc Updates

After updating the docs, rerun these checks:

```bash
curl -sS 'https://chowdahh.com/api/v1/streams/top?limit=3' | jq .
curl -sS 'https://chowdahh.com/api/v1/search?q=NASA&limit=5' | jq .
curl -sS -X POST 'https://chowdahh.com/api/v1/feed-sessions' \
  -H 'content-type: application/json' \
  -d '{"intent":"browse","budget_minutes":5,"include_controls":true}' | jq .
curl -sS -X POST 'https://chowdahh.com/api/v1/signals' \
  -H 'content-type: application/json' \
  -d '[{"signal_type":"bogus","card_id":"card_abc"}]' | jq .
curl -sS -X POST 'https://chowdahh.com/api/v1/feedback' \
  -H 'content-type: application/json' \
  -d '{"feedback_type":"content_request"}' | jq .
```

Expected review points:

- No doc example should use `/v1/...` as the versioned API prefix.
- Feed-session examples should match live field names or explicitly call out intentional contract targets.
- Search docs should no longer imply guaranteed anonymous drill-down.
- Write docs should tell clients to inspect body-level counters, not only HTTP status.
