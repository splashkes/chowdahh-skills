---
name: chowdahh-preferences
description: >-
  Sync durable content preferences with Chowdahh -- followed topics, avoided topics,
  tone, delivery style, and budget. Use when the user wants to personalize their
  Chowdahh feed, save preferences, change their defaults, or set up their profile.
homepage: https://chowdahh.com
metadata: {"clawdbot":{"emoji":"\ud83c\udf9b\ufe0f","requires":{"bins":["curl"]}}}
---

# Chowdahh Preferences

Use this skill when a person wants better default results from Chowdahh, or when repeated conversations reveal a stable preference worth syncing.

## Goal

Populate two separate stores:

- **Local agent memory** with nuanced, private context
- **Chowdahh profile** with confirmed durable preferences

## Question Pattern

Ask a short sequence:

1. What do you want more of?
2. What do you want less of?
3. Should Chowdahh default to brief, balanced, source-heavy, or more narrative?
4. Should I usually choose the control chips for you, or ask before changing them?
5. Do you want those preferences saved just with me, or in Chowdahh too?

## Sync Rule

Only call `PUT /api/v1/preferences/{person_id}` after the person has explicitly agreed to product-level persistence.

Requires a person token: `Authorization: Bearer ch_person_...`

```bash
curl -X PUT https://chowdahh.com/api/v1/preferences/{person_id} \
  -H 'Authorization: Bearer ch_person_...' \
  -H 'content-type: application/json' \
  -d '{"topics_followed": ["science", "canada"], "topics_avoided": ["celebrity-gossip"], "tone_preferences": ["uplifting"]}'
```

To discover your `person_id`: create an authenticated feed session, then call `GET /api/v1/feed-sessions/{session_id}` -- the GET response includes `person_id`. (The initial POST create response may return `person_id: null`; use the GET follow-up.) The preferences endpoint enforces that the path `person_id` matches the token owner.

## Good Local-Memory Candidates

- "avoid heavy news before 9am"
- "prefers Canadian framing when available"
- "wants good-news without fluff"
- emotional sensitivities
- temporary missions
- family or team context

## Good Chowdahh-Sync Candidates

- Followed topics
- Muted topics
- Preferred tone lenses
- Default budget
- Preferred delivery mode

## Practical Rule

If a preference is subtle, situational, private, or emotionally loaded -- keep it local until the person explicitly asks for product-level persistence.
