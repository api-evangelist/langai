---
name: Build a Lang.ai classifier from a dataset
description: >-
  Create a Lang.ai project from a CSV dataset of customer interactions, wait for the
  algorithm to finish extracting intents, and inspect the tags that were produced.
api: openapi/langai-api-openapi.yml
operations:
  - createProject
  - listProjects
  - getProjectTags
generated: '2026-07-19'
method: generated
source: https://docs.lang.ai/
---

# Build a Lang.ai classifier from a dataset

Use this when you have a body of customer text — support tickets, chatbot transcripts,
survey responses — and you need a Lang.ai project that can classify new text against it.

## Before you start

- Base URL is per tenant: `https://{company}.lang.ai/api/v1`, where `{company}` is the
  customer's own Lang.ai instance subdomain. There is no shared production host.
- Every request needs `Authorization: Bearer <api-token>`. Tokens come from the Settings
  section of the instance. Generating a new token invalidates the previous one, so never
  rotate mid-run.
- You need at least one team id for `team_ids`; the API does not expose a team-listing
  operation, so obtain it from the instance UI.

## Steps

1. **Create the project — `createProject`** (`POST /project`).
   This is not a JSON body. Send `multipart/related` (RFC 2387) with a boundary string:
   - Part 1 must be the metadata, `Content-Type: application/json; charset=UTF-8`, with
     `name`, `csvOptions` (`textColumn` required, `dateColumn` optional), and `team_ids`.
   - Part 2 must be the dataset, `Content-Type: text/csv` or `text/plain`.
   - Set `Content-Type: multipart/related; boundary={boundary_string}` on the request.
   - Prefix each boundary with two hyphens and close the final boundary with two trailing
     hyphens.
   Any extra CSV columns beyond `textColumn`/`dateColumn` are retained as document
   metadata and become filterable in the project dashboard, so include the columns you
   will want to slice by later.
   The response is `{"id": "<projectId>"}` — the id only, not the finished project.

2. **Wait for processing — `listProjects`** (`GET /projects`).
   Project creation is asynchronous. Poll and read `status` for your project id: it moves
   through `Processing` to `Completed`, or to `Errored`. Do not classify against a
   project until it is `Completed`. Back off between polls; there are no published rate
   limits and a 429 is possible.

3. **Inspect the tags — `getProjectTags`** (`GET /projects/{projectId}`).
   Returns the project plus `tags[]`, each with `id`, `name`, `createdAt`, `updatedAt`
   and `isDraft`. Tags are the user-defined groupings of the intents and features the
   algorithm extracted; a tag with `isDraft: true` is still being worked on and should
   not be treated as a stable label.

## Rules

- **Do not blindly retry `createProject`.** It has no idempotency contract — a retry
  after a timeout can create a duplicate project. Call `listProjects` first and match on
  `name` before retrying.
- On 400, check the multipart structure before the field values — a wrong part order or a
  missing `charset=UTF-8` on the metadata part is the most common cause.
- On 401, the token is missing or has been invalidated by a newer token.
- On 429, back off exponentially with jitter; no `Retry-After` or `RateLimit` headers are
  published.

## References

- Conventions: `conventions/langai-conventions.yml`
- Errors: `errors/langai-problem-types.yml`
- Data model: `data-model/langai-data-model.yml`
- Docs: https://docs.lang.ai/ · Tags: https://help.lang.ai/en/articles/3175538-using-tags
