# Chowdahh API Reference

Base URL: `https://chowdahh.com`
Version prefix: `/api/v1`
Media type: `application/json` (UTF-8)

## Response Envelope

Successful responses use `{data, meta}` and usually include `guidance`. The `guidance` block is common on successful responses but not universal, and `guidance.next_best_actions` is optional.

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

Production note, tested on April 14, 2026:

- `GET /api/v1/streams`, `GET /api/v1/search`, feed-session endpoints, radio endpoints, signals, and validation failures on feedback and submissions all matched this broad envelope model.
- `GET /api/v1/replay` returned `{error, meta}` with `500 service_unavailable` even with a valid person token.

Error codes: `invalid_request`, `unauthorized`, `forbidden`, `not_found`, `conflict`, `rate_limited`, `validation_error`, `expired_session`, `invalid_control`, `processing`, `service_unavailable`.

## Authentication

| Mode | Header | Rate Limit | Access |
|------|--------|------------|--------|
| Anonymous | none | 30/min | public reads, feed sessions, radio, search (derives stable `person_id` from IP+UA -- pseudonymous, not stateless) |
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
Returns: `session_id`, `state`, and a minimal `controls` object (currently only `rank_mode`). Does **not** return groups, options, selected flags, or reranked items. To see items under the new ranking, call `POST .../more` after this call.

**There is no reliable way to confirm a control took effect from this response alone.** Both valid and invalid slugs produce the same minimal response. Agents should: (1) only apply slugs from a prior `controls` response, (2) treat the PATCH as fire-and-forget, and (3) verify the effect by inspecting items from the next `.../more` call.

**Warning:** The server currently accepts any slug without validation -- invalid control names return `200` with "Controls updated" rather than `invalid_control`. Do not rely on the server to reject hallucinated chips.

### Public Streams

```
GET /api/v1/streams/{slug}?limit=10
```
Discover available streams via `GET /api/v1/streams`. Default set includes: `top`, `latest`, `science`, `world`, `tech`, `business`, `health`, `culture`, `sports`, `good-news`, `local`.

Production note: the stream listing exposes stable `slug` values, but friendly `name` values may be `null`. Do not require `name` for discovery.

### Search

```
GET /api/v1/search?q=query&limit=10
```
Searches clusters by topic match. Results are card objects with `id`, `headline`, `summary`, etc. Some results may have empty `headline` values or `null` for `image_url`, `share_url`, and `canonical_url`. Always handle missing fields gracefully. Results do not currently expose drill-down IDs for curators.

### Topic Drill-Down

```
GET /api/v1/topics/{topic_name}
```
Callable anonymously using a topic name (e.g. `GET /api/v1/topics/NASA`). Returns `data.topic` (the topic name as a string) and `data.timeline` (array, often empty). This is a shallow lookup -- useful for confirming a topic exists, but do not expect a rich object with sources, related topics, or summaries.

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

Production note, tested on April 14, 2026:

- `GET /api/v1/stats/activity` required a person token and returned data successfully.
- The response came back as `data.stats.by_type`, even when `group_by=topic`, `group_by=type`, or `group_by=day` was requested.
- The live keys were raw interaction names such as `category_on`, `category_off`, `load_more`, `music_play`, `music_listen`, `open`, and `show_full`.
- `GET /api/v1/replay` now works with a valid person token (fixed April 14, 2026).

### Radio

**Start radio**
```
POST /api/v1/radio-sessions
{"mode": "briefing", "duration_minutes": 5, "topic_lenses": ["science"]}
```
Modes: `headlines`, `briefing`, `topic_run`.
Returns: `radio_session_id`, `state`, `queue_length`, `tracks[]` (each with `audio_url`).

Sessions may start as `ready` or `playing` -- the initial state is not guaranteed. Always check `data.state` and send `{"action": "resume"}` if `ready`.

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

Production note: use `GET` to test audio availability. A `HEAD` request returned `405` during live testing, while `GET` returned `200 audio/mpeg`.

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

To discover your `person_id`: create an authenticated feed session, then call `GET /api/v1/feed-sessions/{session_id}` with your person token -- the GET response includes `person_id`. Note: the initial POST create response may return `person_id: null`; use the GET follow-up.

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
