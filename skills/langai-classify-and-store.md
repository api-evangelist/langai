---
name: Classify and store customer interactions
description: >-
  Classify a piece of customer text against a Lang.ai project to get its tags and
  intents, and persist documents with metadata so they appear in the project dashboard.
api: openapi/langai-api-openapi.yml
operations:
  - analyzeDocument
  - saveDocument
  - getProjectTags
generated: '2026-07-19'
method: generated
source: https://docs.lang.ai/
---

# Classify and store customer interactions

Use this for the runtime path: a new support ticket or chatbot message arrives and you
need Lang.ai's tags and intents for it, optionally persisting it for reporting.

## Before you start

- `Authorization: Bearer <api-token>` on every request, against
  `https://{company}.lang.ai/api/v1`.
- You need a `projectId` whose status is `Completed`. Confirm with `getProjectTags`
  (`GET /projects/{projectId}`) if unsure — a bad id returns 404.

## Choose the right operation

- **`analyzeDocument`** (`POST /analyze`) — classify **without** persisting. Use for
  routing, triage, or preview. Nothing lands in the dashboard.
- **`saveDocument`** (`POST /documents`) — classify **and** persist into the project.
  Use when the interaction should be counted in reporting.

Both return the same shape:

```json
{
  "tags":    [{ "id": "tqGMbCpTBZOJaram", "name": "My first tag" }],
  "intents": [{ "name": "intent", "features": ["feature", "feature>feature"] }]
}
```

`tags[]` are the classifier's stable labels — key off `tags[].id`, not `name`, since
names are user-editable. In `intents[].features`, a `>` separates a second-level feature
from its parent (`feature>feature`).

An empty `tags[]` is a valid result, not an error: the text matched no tag.

## Steps

1. **Classify — `analyzeDocument`.** Send `{"text": "...", "projectId": "..."}`. Both
   fields are required; omitting either returns 400.

2. **Persist — `saveDocument`.** Send `text` and `projectId` plus:
   - `metadata` — a free-form object of key/value pairs (e.g. channel, queue, region).
     New metadata keys sent via the API become selectable in the project's setup section
     and filterable in dashboards, so send a consistent key set.
   - `date` — a valid ISO 8601 date. Defaults to request time if omitted. **Always send
     the real interaction timestamp** when backfilling, or the whole batch lands on
     today's date and skews time-series dashboards.
   - `id` — your own document id. Always send one (see below).

## Rules

- **Always supply your own `id` on `saveDocument`.** A document saved with an existing id
  overwrites the stored one, so retries and replays are idempotent and will not duplicate.
  Use your source system's ticket/message id. Without an `id`, a retry after a timeout
  silently double-counts the interaction. There is no `Idempotency-Key` header.
- Backfills are safe to re-run under the same rule — re-sending the same ids updates in
  place.
- On 404, the `projectId` is wrong or belongs to another tenant — projects are scoped to
  one instance.
- On 429, back off exponentially with jitter, then replay; with stable ids the replay is
  safe. No rate-limit headers are published.
- Errors are plain JSON with meaningful HTTP status codes — do not expect RFC 9457
  `application/problem+json` or a machine-readable error code field.

## References

- Conventions: `conventions/langai-conventions.yml`
- Errors: `errors/langai-problem-types.yml`
- Data model: `data-model/langai-data-model.yml`
- Docs: https://docs.lang.ai/ · Dashboards:
  https://help.lang.ai/en/articles/2724433-getting-started-with-dashboards
