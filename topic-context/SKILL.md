---
name: topic-context
description: Read a local topic's brief and keep its description current — list/search local topics, fetch a topic, append discovered context, replace one ## / ### description section, or open a topic in the running app. Use when the user asks what a topic is about, which local topics exist, to open a topic, or to update its description/status. Uses the local Principal MCP Bridge at http://localhost:3044. NOT for creating topics — use create-topic. This skill does not browse published topics or edit a topic's title, repository scope, status metadata, or assets.
---

# Topic Context

Use this skill to read a local topic's description as an agent brief and leave
useful, current context for the next person or agent. Topics are subject briefs
with repository scope; they are not bundles of trails.

The skill talks to the running electron-app's local Principal MCP Bridge, not
to web-ade. It is the read-and-update companion to `create-topic`.

## When to use

- You were given a brief such as `Fetch http://localhost:3044/api/topics/<id>
  to begin working on topic "…"`.
- The user asks what a topic is about, what local topics exist, or asks to open
  a topic in the app.
- The user asks to append context or update a status/findings section in a
  topic's description.
- The user asks to validate the references in a topic description.

Use `create-topic` to create a topic. This skill only updates descriptions; it
does not change a topic's title, repository scope, structured status, attached
assets, or subsystem-model associations. It does not browse topics published on
web-ade.

## Prerequisites

The electron-app must be running. The bridge listens on port `3044`:

```bash
curl -s http://localhost:3044/health
```

If the health check fails, ask the user to launch the app. The bridge is local
and unauthenticated; no GitHub token is needed for these routes.

## Routes

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/topics` | List/search local topics. Accepts optional `?q=` substring filter. Returns `{ success, topics, count }`. |
| `GET` | `/api/topics/:id` | Fetch a topic. Returns `{ success, topic }`. |
| `POST` | `/api/topics/:id/description/append` | Append text to the description. Body: `{ text }`. |
| `POST` | `/api/topics/:id/description/section` | Replace one `##` or `###` section, or append it if absent. Body: `{ heading, body, level? }`. |
| `POST` | `/api/topics/:id/activate` | Open the topic as a tab in the app. Returns `{ success, delivered, windowOpened }`. |
| `POST` | `/api/topics/:id/validate-links` | Validate PURL-qualified file references in the description. Read-only. |
| `POST` | `/api/topics/:id/subsystem-models` | Associate a subsystem model. Body: `{ modelId, trailIds? }`. |
| `DELETE` | `/api/topics/:id/subsystem-models/:modelId` | Remove a subsystem-model association. |

The topic read and description-write routes return `{ success, topic }`.
Section upsert also returns `action: "replaced" | "inserted"`. Link validation
returns `{ success, topicId, summary, findings }`.

The local topic shape includes:

```ts
{
  id: string;
  title: string;
  description?: string;
  repos?: string[]; // PURLs, e.g. "pkg:github/owner/repo"
  createdAt: string;
  updatedAt: string;
  createdBy?: { githubId: number; githubLogin: string };
  status?: { state: string; label?: string; waitingOn?: object };
  assets?: Array<{ id: string; mime: string; data?: string; url?: string }>;
  subsystemModels?: Array<{ modelId: string; trailIds?: string[] }>;
}
```

## Find a topic

If the user gave you an id, fetch it directly. Otherwise, list or search topics:

```bash
curl -s http://localhost:3044/api/topics \
  | jq '.topics[] | {id, title, descriptionPreview, state, updatedAt}'

curl -s "http://localhost:3044/api/topics?q=storage" \
  | jq '.topics[] | {id, title, descriptionPreview}'
```

The list is a lightweight directory, sorted by most recently updated. Each
entry has `id`, `href`, `title`, `descriptionPreview`, `state`, `createdAt`,
and `updatedAt`; it does not include the full description.

Fetch the topic before making any update:

```bash
curl -s "http://localhost:3044/api/topics/$TOPIC_ID" | jq
```

Read the `description` as the working brief. A topic can declare its repository
scope in `repos`; use that scope when interpreting file references.

## Open a topic in the app

```bash
curl -s -X POST "http://localhost:3044/api/topics/$TOPIC_ID/activate"
```

This opens/focuses the appropriate app window and activates the topic tab. It
does not edit the topic.

## Update the description

The description is Markdown. Use **append** for genuinely new context that
doesn't replace existing text:

```bash
curl -s -X POST "http://localhost:3044/api/topics/$TOPIC_ID/description/append" \
  -H 'content-type: application/json' \
  -d '{"text":"The token refresh path is in `AuthService.refresh`."}'
```

Use **section upsert** when updating existing status, findings, or next steps.
First fetch the topic and copy the heading exactly:

```bash
curl -s -X POST "http://localhost:3044/api/topics/$TOPIC_ID/description/section" \
  -H 'content-type: application/json' \
  -d '{
    "heading": "Status",
    "body": "**Done** — completed X. Remaining: Y."
  }'
```

- One exact heading match: its section body is replaced in place.
- No match: a new section is appended (`level` defaults to 2; use 3 for `###`).
- Multiple matches: the route returns `409`; resolve the duplicate headings
  before retrying.

Match heading text exactly (leading `#` characters are optional). Replacing a
section also replaces its nested deeper headings through the next heading at
the same level.

## File references and validation

Use a PURL-qualified link for a file in a repository:

```markdown
[AuthService.refresh](pkg:github/owner/repo#src/main/auth/AuthService.ts)
```

The repository must be included in the topic's `repos` scope. After editing
references, validate them:

```bash
curl -s -X POST "http://localhost:3044/api/topics/$TOPIC_ID/validate-links" \
  | jq
```

The validator reports malformed or non-PURL repository links, missing or
out-of-scope files, and suggestions for path-like inline code. It checks local
clones first and may use the repository's remote default branch as a fallback.
Apply corrections with section upsert, then validate again.

## Common errors

| Response | Meaning |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | The app/bridge is not running. Ask the user to launch the app. |
| `404 unknown topic id` | The id is not in the local topic registry. |
| `400 text (non-empty string) is required` | Append body is missing `text`. |
| `400 heading (non-empty string) is required` | Section-upsert body is missing `heading`. |
| `400 body (non-empty string) is required` | Section-upsert body is missing `body`. |
| `409 heading "X" matches N sections` | Duplicate headings make the target ambiguous; resolve them first. |

## Author discipline

- Fetch the topic before editing; don't overwrite context you haven't read.
- Upsert status sections instead of appending a contradictory second status.
- Keep updates concrete and actionable.
- Use PURL-qualified links for files and only reference repositories in the
  topic's declared scope.
- Keep the description focused on connective context; the topic already carries
  its title, repository scope, status metadata, assets, and subsystem-model
  associations.

## References

- HTTP routes: `electron-app/src/main/topics/topicRoutes.ts`
- Topic registry: `electron-app/src/main/stores/TopicRegistryService.ts`
- Link validation: `electron-app/src/main/topics/validateTopicLinks.ts`
- Topic type: `DraftTopic` in `@principal-ai/subsystems-core/node`
- Brief that points agents here:
  `electron-app/src/renderer/components/Titlebar/BriefAgentButton.tsx`
- Sibling skill: `create-topic`
