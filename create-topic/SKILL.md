---
name: create-topic
description: Create a local topic in the running electron-app, with a title, Markdown brief, and optional references to repositories, subsystem models, C4 sketches, and related topics. Supports the production or development Principal MCP Bridge. Use when the user asks to create or start a topic, make a topic for work, or create a topic with references. NOT for reading/updating an existing topic (use topic-context) or searching/opening local topics (use search-local-topics).
---

# Create Topic

Create a local subject brief in the running electron-app through its Principal
MCP Bridge. A topic has a title, a Markdown description that serves as an agent
brief, and four optional reference types:

| Reference        | Field                                           |
| ---------------- | ----------------------------------------------- |
| Repositories     | `repos: string[]` (PURLs)                       |
| Subsystem models | `subsystemModels: [{ modelId, trailIds? }]`     |
| C4 sketches      | `references: [{ kind: "c4-sketch", sketchId }]` |
| Related topics   | `references: [{ kind: "topic", topicId }]`      |

The description belongs to the topic; it is not a separate document attachment.
Related topics are other briefs with their own descriptions. A topic is not a
bundle of trails.

## When to use

- The user asks to create or start a topic.
- The user wants a topic as a brief for work.
- The user wants a topic scoped to a particular repository.
- The user wants a topic referencing models, sketches, or other topics.

Use `topic-context` to read/update an existing topic, and
`search-local-topics` to find or open local topics.

## Prerequisites

The intended electron-app build must be running. Choose it explicitly:

- Production: `http://localhost:3044`
- Development: `http://localhost:3054`

```bash
BASE=http://localhost:3044  # Use http://localhost:3054 for development.
curl -fsS "$BASE/health"
```

If the health check fails, ask the user to launch the app rather than guessing
another port or silently switching builds. The bridge is local and unauthenticated.

## Discover the running contract

Use `principal-ai-desktop-app-tools` for bridge discovery. Fetch the contract
before creating a topic, especially when attaching sketches or related topics:

```bash
curl -fsS "$BASE/routes" \
  -H 'content-type: application/json' \
  -d '{"routes":[{"method":"POST","path":"/api/topics"}]}'
```

This is a read-only documentation lookup, not topic creation. Read the returned
fields and notes. If `body.references` is absent, the build is too old for sketch
and topic attachments. Ask for an updated/restarted build; do not submit a
payload that could silently lose its references. Do not switch to production
when testing development.

## Create a topic

`POST /api/topics` accepts a non-empty `title`, optional Markdown
`description`, optional `repos` array of PURLs, optional `subsystemModels` array
of `{ modelId, trailIds? }` refs, and optional `references` array of C4 sketch
and topic refs. It returns `{ success: true, topic }`; the registry
generates the topic id and timestamps.

```bash
curl -fsS -X POST "$BASE/api/topics" \
  -H 'content-type: application/json' \
  -d '{
    "title": "Investigate token refresh",
    "description": "## Goal\nFind where token refresh happens and when it fires.",
    "repos": ["pkg:github/owner/repo"]
  }' | jq
```

For a topic without an explicit repository scope, omit `repos`:

```bash
curl -fsS -X POST "$BASE/api/topics" \
  -H 'content-type: application/json' \
  -d '{
    "title": "Investigate token refresh",
    "description": "## Goal\nFind where token refresh happens and when it fires."
  }' | jq '.topic.id'
```

Capture and report the returned `topic.id`; use it to refer to the topic later.
This HTTP handler does not expose publishing, image assets, status, or visibility
updates. Publishing is a separate app workflow and is not performed by this skill.

## Create a topic with all four reference types

Use actual, existing artifact IDs. The following IDs are placeholders; replace
them before submitting:

```bash
curl -fsS -X POST "$BASE/api/topics" \
  -H 'content-type: application/json' \
  -d '{
    "title": "Review desktop architecture",
    "description": "## Goal\nReview the architecture sketch alongside the implementation model and related brief.",
    "repos": ["pkg:github/principal-ade/electron-app"],
    "subsystemModels": [{"modelId":"sg-123-abc","trailIds":["tl-topic-open"]}],
    "references": [
      {"kind":"c4-sketch","sketchId":"c4s-123-abc"},
      {"kind":"topic","topicId":"topic-123-related"}
    ]
  }'
```

- Discover related topics with `GET /api/topics?q=<URL-encoded-search>` and read
  the intended target with `GET /api/topics/<URL-encoded-id>`.
- Use subsystem-model and sketch IDs supplied by the user, existing artifacts,
  or the `create-subsystem-model` / `create-c4-sketch` authoring workflows. Do not
  invent IDs or create unrelated artifacts just to fill every section.
- Repository PURLs go in `repos`; model IDs go in `subsystemModels`. Only
  sketches and other topics go in `references`.
- IDs are stable artifact identities, not filesystem paths, S3 URLs, or presigned
  URLs. Reference titles and contents are resolved by the app.
- `references` is optional; `[]` explicitly means no sketch/topic attachments.
  IDs are trimmed and duplicate kind/id pairs are stored once, in first-seen
  order. This route validates shape, not target existence. Missing targets appear
  unavailable in the rail, so HTTP success alone does not prove they resolve.
- Sketches are authored architecture hypotheses, not grounded implementation
  evidence. Keep that distinction clear in the brief.
- Attachments are independent of Markdown links in the description. Do not
  insert links as a substitute for `references`, or promise that
  `principal://` links are clickable in builds without Markdown navigation.

After creation, fetch the `/api/topics/:id` contract, read the topic back, and
verify all requested fields. The topic tab shows Projects, Models, C4 sketches,
and Related topics. Sketch references open the embedded C4 viewer; related-topic
references open the target brief.

```bash
curl -fsS "$BASE/api/topics/$TOPIC_ID"
```

If the user wants it opened, fetch the `/api/topics/:id/activate` contract and
then `POST "$BASE/api/topics/$TOPIC_ID/activate"`. `delivered: 1` only confirms
dispatch to a window, not that the renderer displayed the tab.

## Scope a topic to a repository

`repos` contains PURL strings such as `pkg:github/owner/repo`. To resolve a
local clone to its canonical PURL, list registered repositories:

```bash
curl -fsS "$BASE/api/repos" \
  | jq '.repos[] | {path, remoteUrl, purl}'
```

If the clone is not registered, register its absolute path:

```bash
curl -fsS -X POST "$BASE/api/repos" \
  -H 'content-type: application/json' \
  -d '{"path":"/absolute/path/to/repository"}' \
  | jq '.repo.purl'
```

Then pass that PURL in `repos` when creating the topic. A local-only repository
may resolve to a `pkg:generic/local/...` PURL; that scopes the topic locally but
is not portable to web-ade.

## Associate subsystem models

A topic may reference the **subsystem models** it concerns — the architecture or
flows the work touches. Each ref is `{ modelId, trailIds? }`: `modelId` is the
model's store id (`sg-...`, a file under `~/.principal/subsystem-models/`), and
`trailIds` optionally narrows the link to specific runtime flows.

```bash
curl -fsS -X POST "$BASE/api/topics/$TOPIC_ID/subsystem-models" \
  -H 'content-type: application/json' \
  -d '{"modelId":"sg-1781...","trailIds":["tl-auth-refresh"]}' | jq
```

Remove one with `DELETE /api/topics/$TOPIC_ID/subsystem-models/$MODEL_ID`; both are
idempotent. You can also pass `subsystemModels` directly when creating the topic.
Associations are local-only — they do not travel when a topic is published.

## Validate description references

Topic descriptions should use PURL-qualified Markdown links for repository
files:

```markdown
[token refresh](pkg:github/owner/repo#src/auth/token.ts)
```

After creating a topic with file references, validate them:

```bash
curl -fsS -X POST \
  "$BASE/api/topics/$TOPIC_ID/validate-links" | jq
```

The result includes a summary and findings for malformed links, missing or
out-of-scope files, and path-like inline-code suggestions. Correct issues by
updating the topic description with `topic-context`, then validate again.

## Keep the brief current

Use `topic-context` after creation to read or update the description as work
progresses. Use `search-local-topics` to find or open a local topic.

## Common errors

| Response                                             | Meaning                                                                            |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `curl: (7) Failed to connect to localhost port 3044` | The app/bridge is not running. Ask the user to launch the app.                     |
| `400 title (non-empty string) is required`           | The request is missing a non-empty title.                                          |
| `400 repos must be an array of PURL strings`         | `repos` is not an array of strings.                                                |
| `400 references must be an array ...`                | A reference has an unsupported kind or lacks its non-empty `topicId` / `sketchId`. |
| Topic creation contract lacks `references`           | Update/restart the intended build before attaching sketches or topics.             |
| `500 failed to create topic`                         | The registry write failed; check the app logs.                                     |

## References

- Topic HTTP routes: `electron-app/src/main/topics/topicRoutes.ts`
- Repo resolution routes: `electron-app/src/main/repos/repoRoutes.ts`
- Create input type: `CreateTopicInput` in
  `electron-app/src/shared/main-process-api-interfaces/TopicAPI.ts`
- Sibling skills: `topic-context` (read/update), `search-local-topics`
  (find/open).
