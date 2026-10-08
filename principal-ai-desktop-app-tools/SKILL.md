---
name: principal-ai-desktop-app-tools
description: Canonical reference for the local HTTP surface the running Principal desktop app (electron-app) exposes to agents through the Principal MCP Bridge at http://localhost:3044. Documents topic, repository, document-note, and document-open routes plus bridge conventions. Use when you need to know what the bridge supports or diagnose a local request. The bridge is local and unauthenticated. NOT for the topic workflows themselves — use create-topic, topic-context, or search-local-topics — or for authoring File City tours.
---

# Principal Desktop App Tools

The running electron-app exposes the **Principal MCP Bridge**, a local HTTP
server for working with local topics, registered repositories, document notes,
and opening documents in an already-running app window. This skill is the
reference for its supported HTTP surface; topic and document-note skills own
their workflows.

## Connect

The bridge listens on `http://localhost:3044`. Confirm that it is available
before calling its routes:

```bash
curl -s http://localhost:3044/health
```

If the health request fails, ask the user to launch the app; don't guess another
port. The bridge is local and unauthenticated. JSON request bodies are limited
to 10 MB.

`repositoryPath` is route-specific, not a field required on every request. It
is used by document-note routes to distinguish the same relative file path in
different repositories, and by document-open when resolving a relative file
path.

## Topics

Topics are local subject briefs with a title, Markdown description, optional
repository PURLs, workflow status, and optional assets. They are not bundles of
File City trails.

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/topics` | List/search local topics. Optional `?q=` filters title and description. Returns `{ success, topics, count }`. |
| `POST` | `/api/topics` | Create a topic from `title`, optional `description`, optional `repos` PURL array, and optional `visibility` intent. Returns `{ success, topic }`. |
| `GET` | `/api/topics/:id` | Read a topic. Returns `{ success, topic }`. |
| `POST` | `/api/topics/:id/description/append` | Append `{ text }` to the description. |
| `POST` | `/api/topics/:id/description/section` | Replace a `##`/`###` section or append it if absent. Body: `{ heading, body, level? }`; ambiguous duplicate headings return `409`. |
| `POST` | `/api/topics/:id/activate` | Open the topic as a tab in the focused/main app window. |
| `POST` | `/api/topics/:id/validate-links` | Validate PURL-qualified file references in the description. Read-only. |
| `DELETE` | `/api/topics/:id` | Delete a local topic and its sync metadata. |

The list route returns lightweight summaries with `id`, `href`, `title`,
`descriptionPreview`, `state`, `createdAt`, and `updatedAt`. The detail route
returns only the topic, not related trail payloads.

## Repositories

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/repos` | List registered local repositories with paths, remotes, and PURLs. |
| `POST` | `/api/repos` | Register an absolute local repository path. Re-registering is idempotent. |
| `DELETE` | `/api/repos?path=<abs>` | Remove a repository from the registry; does not delete its files. |

`POST /api/repos` requires `{ "path": "/absolute/path/to/repo" }`; a Git
repository must exist at that path. Its response includes a canonical PURL when
one can be derived. Use that PURL in a topic's `repos` field.

## Document notes

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/document-notes?repositoryPath=<abs>&relativeFilePath=<rel>` | List notes for one document; `relativeFilePath` is required. |
| `GET` | `/api/document-notes/library?repositoryPath=<abs>` | List documents with notes, optionally filtered by repository. |
| `POST` | `/api/document-notes` | Create a note. Body includes `relativeFilePath`, non-empty `body`, and `anchor: { exact, prefix?, suffix? }`; `repositoryPath` and `author` are optional. |
| `PATCH` | `/api/document-notes/:noteId` | Update a note's body. Body includes `relativeFilePath` and `body`; `repositoryPath` is optional. |
| `DELETE` | `/api/document-notes/:noteId?repositoryPath=<abs>&relativeFilePath=<rel>` | Delete a note; `relativeFilePath` is required. |

Notes persist under the app's `userData/document-notes/`, partitioned by
repository path and document path.

## Open a document

`POST /api/document/open` sends an open request to the already-focused
principal or dev-workspace window:

```json
{
  "filePath": "src/path/to/file.md",
  "repositoryPath": "/absolute/path/to/repository"
}
```

`filePath` is required. It may be absolute; otherwise, `repositoryPath` is
required to resolve it. The response includes `windowOpened`. The route does
not start or focus a new window; when there is no suitable focused window, it
returns `{ success: true, windowOpened: false }`.

## Persistence

- Topics are stored as one JSON file per topic under `~/.principal/topics/`.
- Document notes are stored under the app's `userData/document-notes/`.
- Registered repositories are maintained by the app's Alexandria registry.

## Troubleshooting

| Symptom | Meaning |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | The app/bridge is not running. Ask the user to launch it. |
| `404 unknown topic id` | The id is not in the local topic registry. |
| `404 unknown note id` | The note id does not exist for the supplied repository/document pair. |
| A document-open request returns `windowOpened: false` | No suitable already-focused app window could receive it. |

## Workflow skills

| Need | Skill |
|---|---|
| Create a local topic | `create-topic` |
| Read or update a topic description | `topic-context` |
| Find or open a local topic | `search-local-topics` |
| Read or edit document notes | `document-notes` |
| Author or publish a File City tour | `file-city-tours` |
