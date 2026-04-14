---
name: chowdahh-feedback
description: >-
  Send feedback to Chowdahh -- content requests, bug reports, feature requests,
  or quality flags. Use when the user wants to request more content on a topic,
  report a problem, suggest a feature, or flag a bad summary.
homepage: https://chowdahh.com
metadata: {"clawdbot":{"emoji":"\ud83d\udcac","requires":{"bins":["curl"]}}}
---

# Chowdahh Feedback

Use this skill when a person wants to ask for more content, report a bug, request a feature, or flag a quality problem in current Chowdahh output.

## Default Behavior

1. Classify the feedback:
   - `content_request` -- wants more or different content
   - `bug_report` -- something is broken
   - `feature_request` -- wants new capability
   - `quality_report` -- current output is wrong or weak
2. Restate the issue briefly before sending it.
3. Include topic, card, or session context when available.
4. Submit via `POST /api/v1/feedback`.

## Endpoint

```bash
curl -X POST https://chowdahh.com/api/v1/feedback \
  -H 'content-type: application/json' \
  -d '{"feedback_type": "content_request", "title": "More Canadian science coverage", "context": {"topic": "science"}}'
```

Required fields: `feedback_type` and `title`.

## Content Requests

Use when the person wants more or different content:

- "send me more Canada science"
- "I want more constructive news in the morning"
- "show more like that topic"

## Quality Reports

Use when a current Chowdahh result is wrong or weak:

- Bad summary
- Wrong grouping
- Missing context
- Wrong attribution

## Bug Reports

Use when something in the product is broken:

- Cards not loading
- Radio not playing
- Controls not applying

## Feature Requests

Use when the person wants something new:

- "I wish I could share cards to Slack"
- "Can you add a local news stream for my city?"

## Avoid

- Forcing all feedback into bug-report language
- Treating content steering as a preference sync when it is really immediate feedback
