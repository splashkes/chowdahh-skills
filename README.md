# Chowdahh Skills for ClawHub

I built Chowdahh for myself -- a guided content system that processes news and content into clustered topics, synthesizes them, and serves display-ready cards to any UI. The name is "chowder" with a Boston accent, because it's a blend of ingredients, not a single source.

These skills let your agent browse news, search topics, stream radio, submit content, and steer preferences -- all through a live production API at [chowdahh.com](https://chowdahh.com). I'm sharing the token processing and synthesis capacity rather than keeping it to myself. If your agent finds it useful, that's the whole point.

**Note:** All endpoints hit the live production API. There is no sandbox. Read-only calls (streams, search, feed sessions, radio) are safe to experiment with. Write endpoints (feedback, submissions, signals) create real records or counters -- use them intentionally.

**Privacy note:** Anonymous feed sessions derive a stable `person_id` from your IP and user-agent. This means anonymous usage is pseudonymous, not stateless -- the server can correlate requests across sessions. If that matters to your use case, be aware of it.

## What Makes This Different

**The guidance envelope.** Most API responses include a `guidance` block that tells your agent what happened. Some responses also include `guidance.next_best_actions`, but those hints are optional. Your runner should always treat the documented `/api/v1` endpoints as the source of truth and use guidance only as a suggestion layer.

```json
{
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

**AX-first design.** Agent Experience is the primary design target, not an afterthought bolted onto a human UI. The API surface is small (12 endpoints), intent-shaped, and honest about what it can and can't do. We take AX seriously: every response is inspectable, controls come from the API not the agent's imagination, and every piece of content preserves its source attribution.

## Install

```bash
# Just the reader (most people start here)
npx clawhub@latest install splashkes/chowdahh

# Full suite
npx clawhub@latest install splashkes/chowdahh-preferences
npx clawhub@latest install splashkes/chowdahh-feedback
npx clawhub@latest install splashkes/chowdahh-submit
```

No API key required. Anonymous access: 30 requests/minute.

## Try It Right Now

```bash
curl -s 'https://chowdahh.com/api/v1/streams/top?limit=3' | jq .
```

## Start Here If You Want To Run It

Read [chowdahh/references/runner-playbook.md](./chowdahh/references/runner-playbook.md) first. It is the runner-facing guide for the live production API and calls out the behaviors verified on April 14, 2026, including:

- which endpoints are safe to try anonymously
- which authenticated endpoints actually worked with a person token
- which documented flows are currently incomplete or broken in production
- what fields are optional or unexpectedly null in real responses

## The Four Skills

| Skill | Emoji | What It Does |
|-------|-------|-------------|
| **chowdahh** | :stew: | Browse feeds, search topics, start radio (anonymous); replay history (person token) |
| **chowdahh-preferences** | :level_slider: | Sync followed/avoided topics, tone, delivery style, budget |
| **chowdahh-feedback** | :speech_balloon: | Content requests, bug reports, feature requests, quality flags |
| **chowdahh-submit** | :mailbox_with_mail: | Submit stories, articles, poems, images, audio, collections |

The adoption path mirrors how people actually use content systems: **read** first, then **personalize**, then **steer or correct**, then **contribute**.

## How Agents Use It

1. Agent installs `chowdahh`
2. User says "what's happening today?"
3. Agent calls `POST /api/v1/feed-sessions` with `{"intent": "browse", "budget_minutes": 5}`
4. API returns feed items with headlines, summaries, images, and controls
5. If `guidance.next_best_actions` is present, agent may use it as a suggestion layer
6. Otherwise the agent continues with the documented `/api/v1` endpoints directly
7. User says "more science" — agent applies the science control chip

## Security Posture

- No API keys or credentials required for anonymous access
- No local code execution -- pure HTTP calls via curl
- No third-party brew taps or package installs
- All traffic goes to `chowdahh.com` only
- MIT-0 licensed -- use however you want

## More

- Full API docs, JS SDK, and runnable examples: [chowdahh_recipes](https://github.com/splashkes/chowdahh_recipes)
- Tested runner guide for this repo: [chowdahh/references/runner-playbook.md](./chowdahh/references/runner-playbook.md)
- Live site: [chowdahh.com](https://chowdahh.com)
