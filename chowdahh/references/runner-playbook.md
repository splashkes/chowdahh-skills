# Runner Playbook

Tested against the live production API at `https://chowdahh.com` on April 14, 2026.

This is the shortest accurate path for a new runner. It describes what worked, what came back from the server, and what you should not assume.

## Ground Rules

- There is no sandbox. Every request hits production.
- Anonymous access is enough for streams, search, feed sessions, radio, and topic lookup.
- A person token enables higher rate limits and some authenticated endpoints.
- A `200` or `201` status does not always mean the write you attempted was fully accepted.
- `guidance` is common but not universal. `guidance.next_best_actions` is optional.

## What Worked In Live Testing

Anonymous:

- `GET /api/v1/streams`
- `GET /api/v1/streams/top?limit=3`
- `GET /api/v1/search?q=NASA&limit=5`
- `GET /api/v1/topics/NASA`
- `POST /api/v1/feed-sessions`
- `GET /api/v1/feed-sessions/{session_id}`
- `PATCH /api/v1/feed-sessions/{session_id}/controls`
- `POST /api/v1/feed-sessions/{session_id}/more`
- `POST /api/v1/signals`
- `POST /api/v1/radio-sessions`
- `GET /api/v1/radio-sessions/{radio_session_id}`
- `PATCH /api/v1/radio-sessions/{radio_session_id}`
- `GET /audio/{track_id}`
- `POST /api/v1/feedback` with an invalid payload
- `POST /api/v1/submissions/items` with an invalid payload
- `POST /api/v1/submissions/collections` with an invalid payload

With a person token:

- `POST /api/v1/feed-sessions`
- `GET /api/v1/stats/activity`

- `GET /api/v1/replay` (fixed April 14, 2026)
- `GET /api/v1/preferences/{person_id}` (discover `person_id` via `GET /api/v1/feed-sessions/{id}`)

## Fastest Safe Smoke Test

### 1. Discover streams

```bash
curl -sS 'https://chowdahh.com/api/v1/streams' | jq .
```

What to expect:

- You get a `data` payload with stream slugs.
- Friendly stream names are not guaranteed. In live testing, `name` was `null` for all default streams.

### 2. Pull a public stream

```bash
curl -sS 'https://chowdahh.com/api/v1/streams/top?limit=3' | jq .
```

What to expect:

- The response uses `data`, `guidance`, and `meta`.
- Real items use `id`, not `card_id`.
- `headline`, `image_url`, `canonical_url`, and `share_url` can be missing or `null` depending on endpoint and item.

### 3. Search

```bash
curl -sS 'https://chowdahh.com/api/v1/search?q=NASA&limit=5' | jq .
```

What to expect:

- Search is usable as a browse surface.
- Anonymous search results do not reliably expose drill-down identifiers.
- Some results may have empty headlines or `null` URLs.

### 4. Start a feed session

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/feed-sessions' \
  -H 'content-type: application/json' \
  -d '{"intent":"browse","budget_minutes":5,"include_controls":true}' | jq .
```

What to expect:

- The create response returns `session_id`, `items`, `count`, `controls`, and `why_this_set`.
- Controls are grouped under `data.controls.groups`.
- Each option has `label`, `slug`, and `selected`.
- `guidance.next_best_actions` may be present, but do not depend on it.

### 5. Read session state

```bash
curl -sS 'https://chowdahh.com/api/v1/feed-sessions/{session_id}' | jq .
```

What to expect:

- This is session metadata only.
- It does not return the original `items` again.
- Cache the items from the create and `more` responses yourself.

### 6. Apply a control, then fetch more

```bash
curl -sS -X PATCH 'https://chowdahh.com/api/v1/feed-sessions/{session_id}/controls' \
  -H 'content-type: application/json' \
  -d '{"apply":["science"]}' | jq .

curl -sS -X POST 'https://chowdahh.com/api/v1/feed-sessions/{session_id}/more?limit=3' | jq .
```

What to expect:

- The control PATCH response is minimal. It does not return reranked items.
- Invalid slugs are currently accepted with the same `200` response shape as valid slugs.
- Only apply slugs you got from the previous `controls` payload.
- Use the next `more` call to inspect the effect.

### 7. Record a signal

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/signals' \
  -H 'content-type: application/json' \
  -d '[{"signal_type":"save","card_id":"<item-id>"}]' | jq .
```

What to expect:

- The response reports `recorded` and `total`.
- Invalid signal types may be silently skipped instead of rejected.
- Treat `recorded`, not the HTTP status code, as the acceptance signal.

### 8. Start radio and fetch audio

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/radio-sessions' \
  -H 'content-type: application/json' \
  -d '{"mode":"briefing","duration_minutes":5,"topic_lenses":["science"]}' | jq .
```

Then:

```bash
curl -sS 'https://chowdahh.com/api/v1/radio-sessions/{radio_session_id}' | jq .
curl -sS 'https://chowdahh.com/audio/{track_id}' -o sample.mp3
```

What to expect:

- The create call returns `radio_session_id`. Keep it. Later GET and PATCH calls do not echo it back.
- The initial state may be `ready`, then later GET may show `playing`.
- `pause` can return `200` while the state still reads `playing`.
- `audio_url` is a relative path such as `/audio/{track_id}`.
- Use `GET` to test audio delivery. In live testing, `HEAD` returned `405`, while `GET` returned `200 audio/mpeg`.
- Track titles are not guaranteed. The tested payload had `title: null`.

## Person Token Reality Check

### Feed sessions do work with a person token

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/feed-sessions' \
  -H 'Authorization: Bearer ch_person_...' \
  -H 'content-type: application/json' \
  -d '{"intent":"browse","budget_minutes":5,"include_controls":true}' | jq .
```

What to expect:

- `guidance.account_state.auth_mode` becomes `person_token`.
- Rate limits increase to the authenticated tier.

### Stats works, but not exactly as documented

```bash
curl -sS 'https://chowdahh.com/api/v1/stats/activity?period=this_week&group_by=topic' \
  -H 'Authorization: Bearer ch_person_...' | jq .
```

What to expect:

- The endpoint requires authentication.
- In live testing, `group_by=topic`, `group_by=type`, and `group_by=day` all returned the same shape.
- The response came back as `data.stats.by_type`, not a topic grouping.
- The keys were raw activity names such as `category_on`, `category_off`, `load_more`, `music_play`, `music_listen`, `open`, and `show_full`.

### Replay works

```bash
curl -sS 'https://chowdahh.com/api/v1/replay?period=this_month&limit=5' \
  -H 'Authorization: Bearer ch_person_...' | jq .
```

What to expect:

- Returns `data.events` (array) and `data.count`.
- Each event has `signal_type`, `headline` (may be empty), and `occurred_at`.
- Fixed April 14, 2026.

### Preferences works (person_id discovery required)

To discover your `person_id`, create an authenticated feed session and then GET it:

```bash
# 1. Create session
SESSION_ID=$(curl -sS -X POST 'https://chowdahh.com/api/v1/feed-sessions' \
  -H 'Authorization: Bearer ch_person_...' \
  -H 'content-type: application/json' \
  -d '{"intent":"browse","budget_minutes":2}' | jq -r '.data.session_id')

# 2. GET session to find person_id (POST create returns person_id: null)
PERSON_ID=$(curl -sS "https://chowdahh.com/api/v1/feed-sessions/$SESSION_ID" \
  -H 'Authorization: Bearer ch_person_...' | jq -r '.data.person_id')

# 3. Read preferences
curl -sS "https://chowdahh.com/api/v1/preferences/$PERSON_ID" \
  -H 'Authorization: Bearer ch_person_...' | jq .
```

What to expect:

- The endpoint enforces that the path `person_id` must match the token owner.
- Returns `data.topics_followed` and other preference fields.

## Failure Cases Worth Knowing

### Feedback validation

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/feedback' \
  -H 'content-type: application/json' \
  -d '{"feedback_type":"content_request"}' | jq .
```

Observed behavior:

- Missing required fields returned `400`.
- The error shape was `{error, meta}` with no `guidance`.

### Single-item submission validation

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/submissions/items' \
  -H 'content-type: application/json' \
  -d '{"title":"Only title no source"}' | jq .
```

Observed behavior:

- Missing URL returned `400 validation_error`.

### Collection submission validation

```bash
curl -sS -X POST 'https://chowdahh.com/api/v1/submissions/collections' \
  -H 'content-type: application/json' \
  -d '[{"title":"Only title no source"}]' | jq .
```

Observed behavior:

- The request returned `201`.
- `data.accepted` was `0`.
- `data.results[0].status` was `skipped`.
- Treat `accepted` and per-item results as the real success signal.

## New-Runner Rules

- Cache IDs from create responses. Several follow-up endpoints do not echo them back.
- Treat nulls and missing fields as normal.
- Treat `guidance.next_best_actions` as optional.
- Treat write counters such as `recorded` and `accepted` as the real outcome.
- Treat feed-control PATCH responses as advisory only; verify via the next `more` call.
- Replay and preferences both work with a person token as of April 14, 2026. Discover `person_id` via `GET /api/v1/feed-sessions/{id}` before calling preferences.
