# Chowdahh API Reference

Base URL: `https://chowdahh.com`
Version prefix: `/api/v1`
Media type: `application/json` (UTF-8)

## Response Envelope

Successful responses use `{data, guidance, meta}`. The `guidance` block is present on most successful responses but is optional — `guidance.next_best_actions` may be absent.

```json
{
  "data": { ... },
  "guidance": {
    "status_explanation": "Human-readable summary of what happened.",
    "next_best_actions": [
      { "action_id": "...", "title": "...", "api_hint": { "method": "POST", "path": "..." } }
    ],
    "account_state": { "auth_mode": "anonymous", "rate_limit": { "limit": 30, "remaining": 28 } }
  },
  "meta": { "request_id": "..." }
}
```

Error responses use `{error, meta}` and may include a `guidance` block when contextual recovery actions are available. Clients should not assume `guidance` is present on error responses.

Error codes: `invalid_request`, `unauthorized`, `forbidden`, `not_found`, `conflict`, `rate_limited`, `validation_error`, `expired_session`, `invalid_control`, `processing`, `service_unavailable`.

## Authentication

| Mode | Header | Rate Limit | Access |
|------|--------|------------|--------|
| Anonymous | none | 30/min | public reads, feed sessions, radio, search |
| Person | `Authorization: Bearer ch_person_...` | 300/min | + replay, preferences, personalized feeds |
| Curator | `Authorization: Bearer ch_cur_...` | 600/min | + high-volume submissions |

Person and curator tokens are currently issued through the Chowdahh account system. There is no self-serve token creation endpoint at this time.

Optional delegation headers: `X-Chowdahh-Agent-Id`, `X-Chowdahh-Agent-Name`, `X-Chowdahh-Acting-For`.

## Endpoints

### Feed Sessions

**Start session**
```
POST /api/v1/feed-sessions
{"intent": "browse", "budget_minutes": 5, "include_controls": true}
```
Returns: `session_id`, `items[]`, `count`, `controls`.

**Get session**
```
GET /api/v1/feed-sessions/{session_id}
```
Returns session metadata only: `session_id`, `intent`, `delivery_mode`, `budget_minutes`, `position`, `card_count`, `state`. Does **not** return the original items or controls — clients should cache items from the create and send-more responses if they need to re-render.

**Send more**
```
POST /api/v1/feed-sessions/{session_id}/more?limit=5
```
Returns next cards with position counter. Preserves controls.

**Update controls**
```
PATCH /api/v1/feed-sessions/{session_id}/controls
{"apply": ["science"], "remove": ["good-news"]}
```
Returns: `session_id`, `state`, `controls` (updated control object). Does **not** return reranked items -- call `POST .../more` after a control change to fetch items under the new ranking. The returned `controls` may not immediately reflect `selected` state for the applied slug.

### Public Streams

```
GET /api/v1/streams/{slug}?limit=10
```
Discover available streams via `GET /api/v1/streams`. Default set includes: `top`, `latest`, `science`, `world`, `tech`, `business`, `health`, `culture`, `sports`, `good-news`, `local`.

### Search

```
GET /api/v1/search?q=query&limit=10
```
Searches clusters by topic match. Results are card objects with `id`, `headline`, `summary`, etc. Some results may have empty `headline` values or `null` for `image_url`, `share_url`, and `canonical_url`. Always handle missing fields gracefully. Results do not currently expose drill-down IDs for curators.

### Topic Drill-Down

```
GET /api/v1/topics/{topic_name}
```
Callable anonymously using a topic name (e.g. `GET /api/v1/topics/NASA`). Returns a topic object but the `timeline` array may be empty for anonymous requests. Useful for checking whether a topic exists; richer payloads (timeline, sources, related topics) may require authenticated context.

### Curator Info

```
GET /api/v1/curators/{curator_id}
```
Returns: identity, specialties, top topics. Requires a valid internal curator ID -- anonymous search results do not currently expose curator IDs, so there is no reliable anonymous path to this endpoint.

### Replay and Stats

```
GET /api/v1/replay?period=this_month&signal_type=save&limit=20
```
Requires person token.

```
GET /api/v1/stats/activity?period=this_week&group_by=topic
```

### Radio

**Start radio**
```
POST /api/v1/radio-sessions
{"mode": "briefing", "duration_minutes": 5, "topic_lenses": ["science"]}
```
Modes: `headlines`, `briefing`, `topic_run`.
Returns: `radio_session_id`, `state`, `queue_length`, `tracks[]` (each with `audio_url`).

Sessions typically start in `playing` state directly. Check `data.state` before sending control actions.

**Important:** Only the create response includes `radio_session_id`. GET and PATCH responses return `state`, `position`, `queue_length`, and `tracks` but do not echo the session ID back. Clients must retain the `radio_session_id` from the create call.

**Audio URLs are relative paths** (e.g. `/audio/abc-123`). Prefix with the base URL to fetch: `https://chowdahh.com/audio/abc-123`.

**Get radio state**
```
GET /api/v1/radio-sessions/{radio_session_id}
```
Returns: `state`, `position`, `queue_length`, `tracks[]`.

**Control radio**
```
PATCH /api/v1/radio-sessions/{radio_session_id}
{"action": "skip"}
```
Returns: `state`, `position`, `queue_length`, `tracks[]`.
Actions: `pause`, `resume`, `skip`, `stop`.

**Note on state transitions:** Control actions return `200` but may not immediately change `state`. In particular, `pause` may return while `state` remains `playing`. Poll with GET if you need to confirm a transition. Do not build a strict state machine that requires `pause` -> `paused` to be synchronous.

**Stream audio**
```
GET /audio/{track_id}
```
Returns `audio/mpeg`. Synthesized on first request, cached after.

### Signals

```
POST /api/v1/signals
[{"signal_type": "save", "card_id": "card_abc"}]
```
Types: `seen`, `open`, `save`, `share`, `dismiss`, `source_open`.

Note: invalid or unrecognized signal types are silently skipped. Callers should inspect `recorded` in the response body — a `200` status does not mean all signals were accepted.

### Preferences

```
GET /api/v1/preferences/{person_id}
PUT /api/v1/preferences/{person_id}
```
Requires person token matching the person_id.

### Submissions

```
POST /api/v1/submissions/items
{"title": "...", "source_url": "..."}
```

```
POST /api/v1/submissions/collections
[{"title": "...", "source_url": "..."}, ...]
```

```
GET /api/v1/submissions/{submission_id}
```
Statuses: `queued`, `processing`, `ready`, `failed`.

Note: collection submissions may succeed at the request level (201) while skipping individual items. Inspect `accepted`, `results[]`, and per-item statuses before treating the submission as complete.

### Feedback

```
POST /api/v1/feedback
{"feedback_type": "content_request", "title": "More Canadian science coverage"}
```
Types: `content_request`, `bug_report`, `feature_request`, `quality_report`.

Note: feedback validation failures return `{error, meta}` without a `guidance` block.

## Worked Flows

### Browse content
1. `POST /api/v1/feed-sessions` with `{"intent": "browse", "budget_minutes": 5, "include_controls": true}`
2. Read `guidance.next_best_actions` for follow-ups
3. Present cards, offer control chips from `data.controls`
4. Record signals: `POST /api/v1/signals`

### Continue session
1. `POST /api/v1/feed-sessions/{id}/more`
2. Cards continue from previous position

### Adjust controls
1. `PATCH /api/v1/feed-sessions/{id}/controls` with `{"apply": ["science"]}`
2. Response confirms the control change but does not return reranked items
3. `POST /api/v1/feed-sessions/{id}/more` to fetch items under the new ranking

### Start radio
1. `POST /api/v1/radio-sessions` with mode and duration
2. Check `data.state` -- the session may already be `playing`
3. `PATCH` with `skip`, `pause`, or `stop` to control

## Feed Item Object

```json
{
  "id": "...",
  "headline": "TSMC accelerates 1.4nm timeline",
  "summary": "Taiwan Semiconductor...",
  "image_url": "https://...",
  "topics": ["science", "business"],
  "source_count": 4
}
```

`headline` may be empty on some search results. `image_url`, `share_url`, and `canonical_url` may be `null` or absent. Always check before rendering.
