---
name: chowdahh
description: >-
  Browse, search, and stream news and content from Chowdahh.
  Use when the user asks for news, today's headlines, what's happening,
  a content feed, content search, or to start Chowdahh Radio.
  No API key required for anonymous access. Replay and preferences
  require a person token.
homepage: https://chowdahh.com
metadata: {"clawdbot":{"emoji":"\ud83c\udf72","requires":{"bins":["curl"]}}}
---

# Chowdahh

A guided content system for people and agents. Browse feeds, search topics, and start Chowdahh Radio.

No API key required for anonymous access (feeds, search, streams, radio): 30 requests/minute. Replay, preferences, and personalized feeds require a person token (`Authorization: Bearer ch_person_...`): 300 requests/minute.

## Quick Start

```bash
# Browse top stories
curl -s 'https://chowdahh.com/api/v1/streams/top?limit=5'

# Search
curl -s 'https://chowdahh.com/api/v1/search?q=NASA&limit=5'

# Start a feed session
curl -X POST https://chowdahh.com/api/v1/feed-sessions \
  -H 'content-type: application/json' \
  -d '{"intent": "browse", "budget_minutes": 5, "include_controls": true}'
```

## The Guidance Envelope

Successful responses use `{data, guidance, meta}`. Error responses use `{error, meta}` and may omit `guidance`. The `guidance` block tells you what happened and suggests what to do next:

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
    ],
    "account_state": { "auth_mode": "anonymous", "rate_limit": { "limit": 30, "remaining": 28 } }
  },
  "meta": { "request_id": "f3704a2e6d2f0f82" }
}
```

Read `guidance.next_best_actions` when present. Treat it as a suggestion layer, not a guaranteed contract, and keep using the documented `/api/v1` routes as the source of truth.

## Default Behavior

1. Start with `POST /api/v1/feed-sessions` and keep the first request minimal: `intent`, `budget_minutes`, `include_controls`.
2. Surface returned controls back to the person in plain language.
3. Use `POST /api/v1/feed-sessions/{session_id}/more` for "send more" -- stay in the same session.
4. Ask follow-up questions only if the result is too broad or the person changes direction.

## Feed Sessions

- `POST /api/v1/feed-sessions` -- start a session (intent, budget_minutes, include_controls)
- `GET /api/v1/feed-sessions/{id}` -- get session state
- `POST /api/v1/feed-sessions/{id}/more` -- send more cards
- `PATCH /api/v1/feed-sessions/{id}/controls` -- apply or remove control chips

Items typically include `id`, `headline`, `summary`, `topics`, and `source_count`. However, `headline` may be empty on some search results, and `image_url`, `share_url`, and `canonical_url` may be `null` or absent. Always handle missing or blank fields gracefully.

## Public Streams

Browse without a session:

```bash
curl -s 'https://chowdahh.com/api/v1/streams/top?limit=5'
```

Discover available streams via `GET /api/v1/streams`. Default set includes: `top`, `latest`, `science`, `world`, `tech`, `business`, `health`, `culture`, `sports`, `good-news`, `local`

## Search

```bash
curl -s 'https://chowdahh.com/api/v1/search?q=climate&limit=5'
```

Searches clusters by topic match. Results are card objects — they do not currently expose result types or drill-down IDs for topics or curators.

## Drill Down

- `GET /api/v1/topics/{topic_name}` -- callable anonymously by topic name (e.g. `/api/v1/topics/NASA`). Returns a topic object but timeline may be empty for anonymous requests.
- `GET /api/v1/curators/{curator_id}` -- requires a valid internal curator ID. No reliable anonymous path exists to discover curator IDs.

## Replay and Stats

Requires a person token (`Authorization: Bearer ch_person_...`):

- `GET /api/v1/replay?period=this_month` -- cards seen, opened, saved, shared, dismissed
- `GET /api/v1/stats/activity` -- aggregates over signals

## Chowdahh Radio

Radio is session-based audio delivery. Start a session, get tracks with MP3 URLs.

```bash
curl -X POST https://chowdahh.com/api/v1/radio-sessions \
  -H 'content-type: application/json' \
  -d '{"mode": "briefing", "duration_minutes": 5}'
```

- Modes: `headlines`, `briefing`, `topic_run`
- Sessions typically start in `playing` state directly
- Retain `radio_session_id` from the create response -- GET and PATCH do not echo it back
- Control: `PATCH /api/v1/radio-sessions/{id}` with `{"action": "skip|pause|stop"}`. State transitions may not be synchronous -- `pause` may return while state remains `playing`
- Audio URLs are relative paths (e.g. `/audio/abc-123`) -- prefix with `https://chowdahh.com`

Radio is not replay. Replay is card history. Radio is forward-looking audio.

## Signals

Record interactions as a batch:

```bash
curl -X POST https://chowdahh.com/api/v1/signals \
  -H 'content-type: application/json' \
  -d '[{"signal_type": "save", "card_id": "card_abc"}]'
```

Types: `seen`, `open`, `save`, `share`, `dismiss`, `source_open`

Note: invalid or unrecognized signal types are silently skipped. Always inspect `recorded` in the response body — a `200` status does not guarantee signals were accepted.

## Authentication

- **Anonymous**: no token needed, 30 req/min, public reads + feed sessions + radio
- **Person token**: `Authorization: Bearer ch_person_...`, 300 req/min, replay + preferences + personalized feeds
- **Curator token**: `Authorization: Bearer ch_cur_...`, 600 req/min, high-volume submissions

## Response Style

- Preserve attribution -- mention whether content is synthesized or source-led
- Explain applied controls honestly
- Use `guidance.next_best_actions` as a suggestion layer when present, but always be prepared to fall back to the documented `/api/v1` endpoints directly

## Avoid

- Inventing control chips not returned by the API -- the server silently accepts any slug, so hallucinated chips won't error but won't work either. Only apply slugs from a prior `controls` response
- Acting as if `good-news` or similar lenses are exact if the API returns low confidence
- Syncing preferences server-side unless the person asked for durable changes
- Treating radio as replay or vice versa

## Resources

### references/api-guide.md
Full API endpoint reference with request/response shapes and worked flows.

### references/agent-experience.md
AX philosophy: how agents should feel when using Chowdahh, the local vs durable preference split, and tone guidance.
