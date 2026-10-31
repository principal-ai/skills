---
name: topic-context
description: Read and update local topic briefs through the Principal desktop app bridge. Use when briefed on a topic, asked to update its title, Markdown, repository scope or references, maintain findings and remaining work, reorganize sections, find topics, validate links, or open a topic. Supports revision-checked edits when available in the running build. NOT for creating topics (use create-topic), publishing, structured status metadata, or assets.
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
- The user asks to rename a topic, change its repository scope or references,
  or delete, move, rename, or reconcile sections of its brief.

Use `create-topic` to create a topic. This skill does not publish topics, browse
published web-ade topics, or change structured status metadata or assets.

## Prerequisites

The intended electron-app build must be running. Choose it explicitly:
production uses `3044`, development uses `3054`. Do not silently switch builds.

```bash
BASE=http://localhost:3044
curl -fsS "$BASE/health"
```

If the health check fails, ask the user to launch the app. The bridge is local
and unauthenticated; no GitHub token is needed for these routes.

Use `principal-ai-desktop-app-tools` for discovery. Before editing, fetch the
running contracts rather than assuming the installed build supports PATCH:

```bash
curl -fsS "$BASE/routes" -H 'content-type: application/json' \
  -d '{"routes":[{"path":"/api/topics/:id"},{"method":"POST","path":"/api/topics/:id/description/section"}]}'
```

If PATCH is absent, use the documented append/section operations for edits they
can express. For a rename or larger restructure, ask for an updated/restarted
build. Never edit the on-disk JSON, import TopicStore to bypass the bridge, or
delete and recreate a topic to update it. Those paths bypass notifications or
change the ID and break existing references and session links.

## Routes

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/topics` | List/search local topics. Accepts optional `?q=` substring filter. Returns `{ success, topics, count }`. |
| `GET` | `/api/topics/:id` | Fetch a topic. Current builds return `{ success, topic, revision }`. |
| `PATCH` | `/api/topics/:id` | Update fields in place. Requires `expectedRevision` from GET; returns `{ success, topic, revision }`. Discover support first. |
| `POST` | `/api/topics/:id/description/append` | Append text to the description. Body: `{ text }`. |
| `POST` | `/api/topics/:id/description/section` | Replace one `##` or `###` section, or append it if absent. Body: `{ heading, body, level? }`. |
| `POST` | `/api/topics/:id/activate` | Open the topic as a tab in the app. Returns `{ success, delivered, windowOpened }`. |
| `POST` | `/api/topics/:id/validate-links` | Validate PURL-qualified file references in the description. Read-only. |
| `POST` | `/api/topics/:id/subsystem-models` | Associate a subsystem model. Body: `{ modelId, trailIds? }`. |
| `DELETE` | `/api/topics/:id/subsystem-models/:modelId` | Remove a subsystem-model association. |

The description-write routes return `{ success, topic }`.
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
  references?: Array<
    { kind: "topic"; topicId: string } |
    { kind: "c4-sketch"; sketchId: string } |
    { kind: "drawing"; drawingId: string }
  >;
}
```

## Find a topic

If the user gave you an id, fetch it directly. Otherwise, list or search topics:

```bash
curl -fsS "$BASE/api/topics" \
  | jq '.topics[] | {id, title, descriptionPreview, state, updatedAt}'

curl -fsS "$BASE/api/topics?q=storage" \
  | jq '.topics[] | {id, title, descriptionPreview}'
```

The list is a lightweight directory, sorted by most recently updated. Each
entry has `id`, `href`, `title`, `descriptionPreview`, `state`, `createdAt`,
and `updatedAt`; it does not include the full description.

Fetch the topic before making any update:

```bash
curl -fsS "$BASE/api/topics/$TOPIC_ID" | jq
```

Read the `description` as the working brief. A topic can declare its repository
scope in `repos`; use that scope when interpreting file references.

## Update a topic in place

Prefer PATCH for general editing: title changes, a cleaned-up full brief,
repository scope, attachment changes, or several section changes in one write.
It also lets you remove duplicate headings, delete stale material, and put new
sections in the right order without custom section parsing or multiple writes.

1. GET the topic and read the entire description. Retain its `revision`.
2. Construct only the fields you intend to change, plus `expectedRevision`.
3. PATCH the same topic URL using JSON serialization, not shell interpolation.
4. On HTTP `409`, GET again and reconcile your changes with the latest brief.
   Do not merely replace the revision and resend a stale description.
5. Read back and verify the changed fields before claiming success.

Example, after reviewing the topic:

```bash
curl -fsS "$BASE/api/topics/$TOPIC_ID" > /tmp/topic-before.json
jq '{expectedRevision: .revision, title: "Updated topic title"}' \
  /tmp/topic-before.json > /tmp/topic-patch.json
curl -fsS -X PATCH "$BASE/api/topics/$TOPIC_ID" \
  -H 'content-type: application/json' --data-binary @/tmp/topic-patch.json
curl -fsS "$BASE/api/topics/$TOPIC_ID"
```

Editable fields: `title`, `description`, `repos`, `subsystemModels`, `references`.
Omitted fields are preserved. An empty `description` clears the brief; `[]`
clears an association array. Arrays replace the whole field, so preserve
existing associations unless their removal was requested. References use the
same shapes as creation; IDs are stable local artifact identities, not URLs.
Publishing, assets, structured status, timestamps, and identity are not editable.

Revision checks coordinate updates in the running app, not arbitrary file
writes or another app process.

## Open a topic in the app

```bash
curl -fsS -X POST "$BASE/api/topics/$TOPIC_ID/activate"
```

This opens/focuses the appropriate app window and activates the topic tab. It
does not edit the topic.

## Update the description

The description is Markdown. Use **append** for genuinely new context that
doesn't replace existing text:

```bash
curl -fsS -X POST "$BASE/api/topics/$TOPIC_ID/description/append" \
  -H 'content-type: application/json' \
  -d '{"text":"The token refresh path is in `AuthService.refresh`."}'
```

Use **section upsert** when updating existing status, findings, or next steps.
First fetch the topic and copy the heading exactly:

```bash
curl -fsS -X POST "$BASE/api/topics/$TOPIC_ID/description/section" \
  -H 'content-type: application/json' \
  -d '{
    "heading": "Status",
    "body": "**Done** — completed X. Remaining: Y."
  }'
```

- One exact heading match: its section body is replaced in place.
- No match: a new section is appended (`level` defaults to 2; use 3 for `###`).
- Multiple matches: the route returns `409`; resolve the duplicate headings
  with a revision-checked full-description PATCH before retrying. If PATCH is
  unavailable, ask for an updated build instead of bypassing the API.

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
curl -fsS -X POST "$BASE/api/topics/$TOPIC_ID/validate-links" \
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
| `400 expectedRevision ... is required` | Read the topic and use its returned revision; do not invent one. |
| `409 Topic changed` | Reread and reconcile with the latest topic before retrying. |
| `400 text (non-empty string) is required` | Append body is missing `text`. |
| `400 heading (non-empty string) is required` | Section-upsert body is missing `heading`. |
| `400 body (non-empty string) is required` | Section-upsert body is missing `body`. |
| `409 heading "X" matches N sections` | Duplicate headings make the target ambiguous; resolve them first. |

## Author discipline

- Fetch the topic before editing; don't overwrite context you haven't read.
- Upsert status sections instead of appending a contradictory second status.
- Keep updates concrete and actionable.
- Before reporting completed work, update the brief with the outcome,
  validation, and remaining work. Replace stale claims instead of only adding
  another status paragraph. Read it back to verify persistence.
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
