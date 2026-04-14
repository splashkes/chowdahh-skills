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

### Public Streams

```
GET /api/v1/streams/{slug}?limit=10
```
Discover available streams via `GET /api/v1/streams`. Default set includes: `top`, `latest`, `science`, `world`, `tech`, `business`, `health`, `culture`, `sports`, `good-news`, `local`.

### Search

```
GET /api/v1/search?q=query&limit=10
```
Searches clusters by topic match. Results are card objects with `id`, `headline`, `summary`, etc. Results do not currently expose stable result types or drill-down IDs for topics or curators.

### Topic Drill-Down

```
GET /api/v1/topics/{topic_id}
```
Returns: summary, timeline, sources, related topics, canonical URL.

### Curator Info

```
GET /api/v1/curators/{curator_id}
```
Returns: identity, specialties, top topics.

Note: these endpoints require true internal identifiers. Anonymous search results do not reliably expose those identifiers, so clients should not assume a direct anonymous search → drill-down flow.

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

Note: sessions may transition directly to `playing` on creation. Check `data.state` before sending control actions -- a `resume` on an already-playing session is harmless but unnecessary.

**Get radio state**
```
GET /api/v1/radio-sessions/{radio_session_id}
```

**Control radio**
```
PATCH /api/v1/radio-sessions/{radio_session_id}
{"action": "skip"}
```
Actions: `pause`, `resume`, `skip`, `stop`.
States: `ready` or `playing` (on create) -> `paused` / `ended`.

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
2. Session re-ranks with new controls applied

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

`image_url` may be absent on some items (notably search results). Do not assume all fields are present -- check before rendering.
```
