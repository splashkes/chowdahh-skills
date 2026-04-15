---
name: chowdahh-submit
description: >-
  Submit content to Chowdahh -- stories, articles, poems, images, audio, video,
  or entire collections. Use when the user wants to contribute content,
  submit a story, upload a corpus, or share something with Chowdahh.
homepage: https://chowdahh.com
metadata: {"clawdbot":{"emoji":"\ud83d\udcec","requires":{"bins":["curl"]}}}
---

# Chowdahh Submit

Use this skill when a person wants to contribute a story, poem, image, audio, video, or a whole collection to Chowdahh.

## Default Behavior

1. Identify whether this is a single item or a collection.
2. Confirm that the person wants it submitted.
3. Confirm the handling policy:
   - `preserve` -- keep original intact
   - `light_synthesis` -- normalize lightly
   - `full_synthesis` -- allow full rewrite
4. Confirm media handling: `preserve_embeds`, `derive_previews`, or `allow_transcodes`.
5. Submit to the matching endpoint.

## Endpoints

**These hit the live production API. There is no sandbox. Each call creates a real submission record.**

**Single item:**
```bash
curl -X POST https://chowdahh.com/api/v1/submissions/items \
  -H 'content-type: application/json' \
  -d '{"title": "My Story", "source_url": "https://example.com/story", "synthesis_mode": "preserve"}'
```

**Collection:**
```bash
curl -X POST https://chowdahh.com/api/v1/submissions/collections \
  -H 'content-type: application/json' \
  -d '[{"title": "Item 1", "source_url": "https://..."}, {"title": "Item 2", "source_url": "https://..."}]'
```

**Check status:**
```bash
curl https://chowdahh.com/api/v1/submissions/{submission_id}
```

Statuses: `queued`, `processing`, `ready`, `failed`.

Production note: invalid single-item submissions returned `400 validation_error` in live testing, but invalid collection submissions returned `201` with `accepted: 0` and per-item `skipped` results. Inspect the body, not just the status code.

## Before Submitting

Restate to the person:

- Title
- Source URL or object URL
- Creator attribution if known
- Requested preservation/synthesis mode

## After Submission

Return the submission ID, processing state, and topic/library ID if present. Offer one next action: open it, save it, or check back later.

## Transformation Policy

- `synthesis_mode`: `preserve` | `light_synthesis` | `full_synthesis`
- `voice_preservation`: `preserve_verbatim` | `normalize_lightly` | `allow_rewrite`
- `media_policy`: `preserve_embeds` | `derive_previews` | `allow_transcodes`

## Avoid

- Silently rewriting content without explicit permission
- Flattening a collection into a single summary unless the person asked for that
- Losing media relationships between text, audio, video, and image objects
