# Agent Experience

## Intended Feeling

An agent should feel like it has a clean companion system, not a maze of product internals.

The main intents should feel obvious:

1. Send content now
2. Send more
3. Replay what already happened
4. Start radio
5. Submit content
6. Send feedback

## Default Agent Flow

### For feed delivery

1. Call `POST /api/v1/feed-sessions` with minimal assumptions.
2. Surface the returned controls back to the person in plain language.
3. If the person leans one way, update local memory first.
4. Only sync durable preferences to Chowdahh if the person confirms they want that to stick.
5. Use `send more` instead of starting a new session unless the intent changed.

### For replay/history

1. Call `GET /api/v1/replay`.
2. Explain the result as card history, not as analytics jargon.
3. Use stats only when the person wants totals or summaries.

### For saving or sharing

1. Record the intent in the conversation.
2. Call `POST /api/v1/signals` after the action actually happens.
3. Offer stats or replay later using the recorded product state.

### For radio

1. Call `POST /api/v1/radio-sessions` with a `mode` and `duration_minutes`.
2. Check `data.state` -- sessions may start in `playing` directly.
3. Read `guidance.next_best_actions` when present for available controls.
4. Control the session via `PATCH /api/v1/radio-sessions/{id}` with an `action` (`skip`, `pause`, `stop`).
5. Radio is session-based -- the server builds a queue from current content.
6. Keep radio language separate from replay/history language.

### For submitting content

1. Confirm ownership or authority to submit.
2. Confirm preservation vs synthesis.
3. Submit via `items` or `collections`.
4. Offer later retrieval once accepted.

### For feedback

1. Classify whether this is a content request, bug report, feature request, or quality report.
2. Send it through one feedback surface.

## AX Requirements

- Agents should not need to understand internal ranking stages.
- Every response should contain enough explanation for a human-facing restatement.
- Controls currently return `label`, `slug`, and `selected`. Counts and confidence are not yet exposed -- present controls honestly as available options without implying precision the API doesn't provide.
- Status transitions should be explicit: `queued`, `processing`, `ready`, `failed`.
- Feed sessions should be resumable enough that `send more` feels natural.

## Preferred API Tone

- Concise
- Inspectable
- Attribution-rich
- No magic wording like "we found something perfect for you"

Instead:

> "Here are 5 cards. I kept it balanced and current, and I can tilt it toward science, good news, or local topics."

## What The Agent Should Never Need To Guess

- Whether content was preserved or rewritten
- Whether an item came from an original source or Chowdahh synthesis
- Whether a control chip came from the API or was hallucinated (the server accepts any slug silently, so agents must only use slugs from prior responses)
- Whether a preference is local-only or synced server-side

## Preference Memory Split

### Keep in local agent memory

- Emotional context ("avoid heavy war coverage in the morning")
- Temporary missions
- Family or team context
- Phrasing preferences
- Unconfirmed dislikes

### Sync to Chowdahh

- Topics followed / muted
- Default delivery mode
- Default budget
- Source preferences
- Tone lenses the person explicitly wants to persist

Only call `PUT /api/v1/preferences/{person_id}` after the person has explicitly agreed to product-level persistence. If a preference is subtle, situational, private, or emotionally loaded -- keep it local until the person asks for it to stick.
