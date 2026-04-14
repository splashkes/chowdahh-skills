---
name: chowdahh
description: >-
  Browse, search, and stream news and content from Chowdahh.
  Use when the user asks for news, today's headlines, what's happening,
  a content feed, topic deep-dives, content search, replay of what they've
  seen or saved, or to start Chowdahh Radio.
  No API key required for anonymous access.
homepage: https://chowdahh.com
metadata: {"clawdbot":{"emoji":"\ud83c\udf72","requires":{"bins":["curl"]}}}
---

# Chowdahh

A guided content system for people and agents. Browse feeds, search topics, drill into stories, replay history, and start Chowdahh Radio.

No API key required. Anonymous access: 30 requests/minute.

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

Every response uses `{data, guidance, meta}`. The `guidance` block tells you what happened and what to do next:

```json
{
  "data": { "session_id": "abc-123", "cards": [...], "count": 6, "controls": {...} },
  "guidance": {
    "status_explanation": "Feed session started with 6 cards.",
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

Read `guidance.next_best_actions` after every call. The API teaches you the conversation flow.

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

Each card includes `headline`, `summary`, `image_url`, `topics`, `source_count`, and `share_url`.

## Public Streams

Browse without a session:

```bash
curl -s 'https://chowdahh.com/api/v1/streams/top?limit=5'
```

Available streams: `top`, `latest`, `science`, `world`, `business`, `culture`, `good-news`, `local`

## Search

```bash
curl -s 'https://chowdahh.com/api/v1/search?q=climate&limit=5'
```

Searches across topics, sources, curators, and collections.

## Drill Down

- `GET /api/v1/topics/{topic_id}` -- topic summary, timeline, sources, related topics
- `GET /api/v1/curators/{curator_id}` -- curator identity, specialties, top topics

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
- Control: `PATCH /api/v1/radio-sessions/{id}` with `{"action": "skip|pause|resume|stop"}`
- Audio: fetch each track's `audio_url` for MP3

Radio is not replay. Replay is card history. Radio is forward-looking audio.

## Signals

Record interactions as a batch:

```bash
curl -X POST https://chowdahh.com/api/v1/signals \
  -H 'content-type: application/json' \
  -d '[{"signal_type": "save", "card_id": "card_abc"}]'
```

Types: `seen`, `open`, `save`, `share`, `dismiss`, `source_open`

## Authentication

- **Anonymous**: no token needed, 30 req/min, public reads + feed sessions + radio
- **Person token**: `Authorization: Bearer ch_person_...`, 300 req/min, replay + preferences + personalized feeds
- **Curator token**: `Authorization: Bearer ch_cur_...`, 600 req/min, high-volume submissions

## Response Style

- Preserve attribution -- mention whether content is synthesized or source-led
- Explain applied controls honestly
- Use the guidance block to drive next steps, don't invent your own flow

## Avoid

- Inventing control chips not returned by the API
- Acting as if `good-news` or similar lenses are exact if the API returns low confidence
- Syncing preferences server-side unless the person asked for durable changes
- Treating radio as replay or vice versa

## Resources

### references/api-guide.md
Full API endpoint reference with request/response shapes and worked flows.

### references/agent-experience.md
AX philosophy: how agents should feel when using Chowdahh, the local vs durable preference split, and tone guidance.
