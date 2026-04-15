# Community Announcement Draft

*(Not for committing -- copy/paste to forums, Discord, X, etc.)*

---

Hey all -- sharing something I built for myself and want to open up.

**Chowdahh** (like "chowder" with a Boston accent) is a guided content system designed for agents. Not a wrapper around an existing news API -- it processes news into clusters, synthesizes topics, and serves display-ready cards with images, attribution, and honest controls. I wanted to do this cool thing for myself, and now I'm sharing the token processing and synthesis capacity so anyone's agent can use it on any UI.

The interesting part is the **guidance envelope**. Most API responses include a guidance block that tells your agent what happened, and some responses also suggest what to do next:

> "Here are 5 cards. I can tilt toward science, send more, or switch to good news. Here's the exact API call for each option."

Your agent still needs the documented endpoints as a fallback source of truth, but the guidance layer makes the interaction easier to steer. I've been calling this **AX-first** -- Agent Experience as the primary design target.

The API surface is intentionally small: feed sessions, search, topics, streams, replay, radio, preferences, submissions, and feedback. Twelve endpoints, not fifty.

**Try it right now** without installing anything:

```bash
curl -s 'https://chowdahh.com/api/v1/streams/top?limit=3' | jq .
```

No API key needed. Anonymous access gives you 30 requests per minute.

**Install the ClawHub skill:**

```bash
npx clawhub@latest install splashkes/chowdahh
```

There are four skills total -- lookup, preferences, feedback, and submit -- but most people just need the first one.

I built this because I wanted my agent to handle my morning content without me opening ten tabs. The token processing happens server-side, so I can share the capacity instead of hoarding it. If you find it useful, that's the whole point. We take AX seriously around here.

Production API, full test suite, OpenAPI spec, JS SDK, and examples at: https://github.com/splashkes/chowdahh_recipes

[chowdahh.com](https://chowdahh.com) is live now.

---

### Short version (for X / Twitter)

Built Chowdahh -- a guided content system for agents. Processes news into display-ready cards. Most responses include a guidance envelope, with optional next-step hints. No API key needed.

`npx clawhub@latest install splashkes/chowdahh`

We take Agent Experience seriously. chowdahh.com

---

### One-liner

Chowdahh: AX-first content for agents. No API key. Guidance helps steer the flow, with documented endpoints as fallback. `npx clawhub@latest install splashkes/chowdahh`
