---
name: create-topic
description: Create a local topic in the running electron-app, with a title, markdown brief, and optional repository PURL scope. The topic appears in the app's Topics surface immediately. Use when the user asks to create or start a topic, make a topic for work, or scope a topic to a resolved repo. NOT for reading/updating an existing topic (use topic-context) or searching/opening local topics (use search-local-topics).
---

# Create Topic

Create a local subject brief in the running electron-app through its Principal
MCP Bridge. A topic has a title, a Markdown description that serves as an agent
brief, and optional repository scope (`repos`). It is not a bundle of trails.

## When to use

- The user asks to create or start a topic.
- The user wants a topic as a brief for work.
- The user wants a topic scoped to a particular repository.

Use `topic-context` to read/update an existing topic, and
`search-local-topics` to find or open local topics.

## Prerequisites

The electron-app must be running. The bridge listens on port `3044`:

```bash
curl -s http://localhost:3044/health
```

If the health check fails, ask the user to launch the app rather than guessing
another port. The bridge is local and unauthenticated.

## Create a topic

`POST /api/topics` accepts a non-empty `title`, optional Markdown
`description`, optional `repos` array of PURLs, and optional local sharing
intent `visibility` (`private` or `sharable`). It returns
`{ success: true, topic }`; the registry generates the topic id and timestamps.

```bash
curl -s -X POST http://localhost:3044/api/topics \
  -H 'content-type: application/json' \
  -d '{
    "title": "Investigate token refresh",
    "description": "## Goal\nFind where token refresh happens and when it fires.",
    "repos": ["pkg:github/owner/repo"]
  }' | jq
```

For a topic without an explicit repository scope, omit `repos`:

```bash
curl -s -X POST http://localhost:3044/api/topics \
  -H 'content-type: application/json' \
  -d '{
    "title": "Investigate token refresh",
    "description": "## Goal\nFind where token refresh happens and when it fires."
  }' | jq '.topic.id'
```

Capture and report the returned `topic.id`; use it to refer to the topic later.
The local sharing `visibility` field is only intent metadata. Publishing to
web-ade is a separate app workflow and is not performed by this skill.

## Scope a topic to a repository

`repos` contains PURL strings such as `pkg:github/owner/repo`. To resolve a
local clone to its canonical PURL, list registered repositories:

```bash
curl -s http://localhost:3044/api/repos \
  | jq '.repos[] | {path, remoteUrl, purl}'
```

If the clone is not registered, register its absolute path:

```bash
curl -s -X POST http://localhost:3044/api/repos \
  -H 'content-type: application/json' \
  -d '{"path":"/absolute/path/to/repository"}' \
  | jq '.repo.purl'
```

Then pass that PURL in `repos` when creating the topic. A local-only repository
may resolve to a `pkg:generic/local/...` PURL; that scopes the topic locally but
is not portable to web-ade.

## Validate description references

Topic descriptions should use PURL-qualified Markdown links for repository
files:

```markdown
[token refresh](pkg:github/owner/repo#src/auth/token.ts)
```

After creating a topic with file references, validate them:

```bash
curl -s -X POST \
  "http://localhost:3044/api/topics/$TOPIC_ID/validate-links" | jq
```

The result includes a summary and findings for malformed links, missing or
out-of-scope files, and path-like inline-code suggestions. Correct issues by
updating the topic description with `topic-context`, then validate again.

## Keep the brief current

Use `topic-context` after creation to read or update the description as work
progresses. Use `search-local-topics` to find or open a local topic.

## Common errors

| Response | Meaning |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | The app/bridge is not running. Ask the user to launch the app. |
| `400 title (non-empty string) is required` | The request is missing a non-empty title. |
| `400 repos must be an array of PURL strings` | `repos` is not an array of strings. |
| `500 failed to create topic` | The registry write failed; check the app logs. |

## References

- Topic HTTP routes: `electron-app/src/main/topics/topicRoutes.ts`
- Repo resolution routes: `electron-app/src/main/repos/repoRoutes.ts`
- Create input type: `CreateTopicInput` in
  `electron-app/src/shared/main-process-api-interfaces/TopicAPI.ts`
- Sibling skills: `topic-context` (read/update), `search-local-topics`
  (find/open).
