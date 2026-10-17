---
name: principal-ai-desktop-app-tools
description: Canonical reference for the local HTTP surface the running Principal desktop app (electron-app) exposes to agents through the Principal MCP Bridge (production port 3044, development port 3054). Documents topic, repository, document-note, document-open, localhost-browser, and managed-process routes plus bridge conventions. Use when you need to open a localhost tab, know what the bridge supports, or diagnose a local request. The bridge is local and unauthenticated. NOT for the topic workflows themselves — use create-topic, topic-context, or search-local-topics — or for authoring File City tours.
---

# Principal Desktop App Tools

The running electron-app exposes the **Principal MCP Bridge**, a local HTTP
server for working with local topics, registered repositories, document notes,
managed processes, and opening documents or localhost browser tabs in an
already-running app window. This skill is the reference for its supported HTTP
surface; topic and document-note skills own their workflows.

## Connect

The packaged/production bridge listens on `http://localhost:3044`; development
builds use `http://localhost:3054`. Choose the intended build explicitly,
especially when both apps are running. Examples below use production; replace
`3044` with `3054` when targeting development. Confirm that it is available
before calling its routes:

```bash
curl -s http://localhost:3044/health
```

If the health request fails, ask the user to launch the intended app; don't
guess another port or silently switch builds. `GET /routes` lists the running
build's routes; older builds may not expose newly added endpoints. The bridge
is local and unauthenticated. JSON request bodies are limited to 10 MB.

`repositoryPath` is route-specific, not a field required on every request. It
is used by document-note routes to distinguish the same relative file path in
different repositories, and by document-open when resolving a relative file
path.

## Topics

Topics are local subject briefs with a title, Markdown description, optional
repository PURLs, workflow status, optional assets, and optional subsystem-model
associations (local store ids). They are not bundles of File City trails.

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/topics` | List/search local topics. Optional `?q=` filters title and description. Returns `{ success, topics, count }`. |
| `POST` | `/api/topics` | Create a topic from `title`, optional `description`, optional `repos` PURL array, optional `subsystemModels` array of `{ modelId, trailIds? }`, and optional `visibility` intent. Returns `{ success, topic }`. |
| `GET` | `/api/topics/:id` | Read a topic. Returns `{ success, topic }`. |
| `POST` | `/api/topics/:id/description/append` | Append `{ text }` to the description. |
| `POST` | `/api/topics/:id/description/section` | Replace a `##`/`###` section or append it if absent. Body: `{ heading, body, level? }`; ambiguous duplicate headings return `409`. |
| `POST` | `/api/topics/:id/activate` | Open the topic as a tab in the focused/main app window. |
| `POST` | `/api/topics/:id/validate-links` | Validate PURL-qualified file references in the description. Read-only. |
| `POST` | `/api/topics/:id/subsystem-models` | Associate a subsystem model with the topic. Body: `{ modelId, trailIds? }`; upserts by `modelId`. Local-only (never travels on publish). |
| `DELETE` | `/api/topics/:id/subsystem-models/:modelId` | Remove a subsystem-model association. Idempotent. |
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

## Processes

The app owns long-lived **managed processes** (dev servers, watchers, tasks).
An agent can start, stop, restart, read logs, and remove them. The app owns the
process — closing its panel does not kill it; only an explicit stop (or app
quit) ends it. `GET /api/processes` returns **managed processes only**. The
Processes UI separately detects local servers the app did not start and merges
them into its display. Those detected entries are not returned by this HTTP
route or controlled through its managed-process endpoints.

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/processes` | List managed processes. Returns `{ success, processes }`. |
| `POST` | `/api/processes` | Start a managed process. |
| `GET` | `/api/processes/:id/logs` | Captured stdout/stderr for one process. Returns `{ success, id, logs }`. |
| `POST` | `/api/processes/:id/stop` | Stop (kill) a running process; escalates SIGTERM → SIGKILL. |
| `POST` | `/api/processes/:id/restart` | Restart by re-running the original command (same id). |
| `DELETE` | `/api/processes/:id` | Remove a finished entry from the list (only when not running). |

`POST /api/processes` body: `{ command, args?, cwd, env?, label?, kind? }`.
`command` is a **binary** (e.g. `npm`, `bun`) with `args` — not a shell string.
Agent starts are constrained: `cwd` must be inside a **registered repository**,
and the command binary must be on the server's allowlist (`npm`, `pnpm`, `yarn`,
`bun`, `npx`, `node`, `deno`, `vite`, `next`, `astro`, `nuxt`, `storybook`,
`http-server`, `vitest`, `jest`, `tsc`, `tsx`, `python`, `python3`, `uv`, `go`,
`cargo`, `make`, `ruby`, `bundle`, `php`, `composer`, `dotnet`, `docker`).
Returns `201 { success, process }`.

Each `process` entry: `{ id, label, command, cwd, repo?, kind, status, origin,
pid?, port?, source?, startedAt, endedAt?, exitCode?, lastError?, runCount? }`.
`id` is a durable app-generated id (never the pid). HTTP registry entries have
`origin: "managed"`; the UI's shared entry type also supports `detected`.
`repo` (`{ name, owner?, purl?, avatarUrl? }`) is the process's cwd
resolved to a registered repository; `port` is parsed from the process's output
when it exposes one.

## Open a localhost browser tab

`POST /api/browser/open` opens or reuses a tab in the already-focused
**Principal MAIN window** and reveals its workspace. It does not support
Dev Workspace windows, create or focus windows, or start servers.

Open an existing localhost server by URL:

```bash
curl -s http://localhost:3044/api/browser/open \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://localhost:5173/docs?mode=preview#intro","label":"Preview"}'
```

Prefer a managed process ID from `GET /api/processes` when the tab should
follow that process through restarts and port changes:

```bash
curl -s http://localhost:3044/api/browser/open \
  -H 'Content-Type: application/json' \
  -d '{"processId":"<managed-process-id>","path":"/docs","label":"Preview"}'
```

Supply exactly one of `url` and `processId`. `label` is optional. `path` is
only accepted with `processId`, defaults to `/`, and must start with a single
`/`. With `url`, put the path, query, and fragment in the URL itself.

URLs must use HTTP, the literal `localhost` hostname, and an explicit port
from 1 through 65535, without credentials. Whitespace, backslashes, and control
characters are rejected. HTTPS, `127.0.0.1`, and remote hosts are unsupported.
Requests carrying `Origin` or `Sec-Fetch-Site` headers return `403`; call this
local agent API from a tool, not a browser page. Opening validation does not
restrict subsequent redirects or navigation or make the webview a sandbox.

Success returns `{ success: true, opened: true, tabId, processId? }`.
`opened` means the renderer acknowledged a committed tab, **not** that the
page loaded or the server responded. The bridge waits up to three seconds.
No suitable focused window returns
`{ success: true, opened: false, reason: "no-focused-browser-host" }`;
no renderer acknowledgement returns `reason: "browser-host-unavailable"`.
Do not report a tab as opened when `opened` is false. Ask the user to focus
the Principal MAIN window and retry when the reason is `no-focused-browser-host`.

Invalid requests return `400`, unknown managed process IDs return `404`, and
processes that are not running or have no known port return `409`. Start or
restart the process separately, then wait for its port before retrying.

- Managed tabs use the process ID, not its PID or current port, as identity.
  A URL associates only when exactly one running managed process matches its
  port; otherwise, it uses a port-keyed tab without a process association.
  Detected external servers are never managed-process associations.
- Reopening updates the path and reloads the tab, even for the same URL,
  resetting browser navigation history.
- Linked tabs follow status, run count, and port changes even when the
  Processes view is hidden. Restart clears the old port until the new run
  reports one. Stopped, failed, restarting, or removed processes leave the tab
  open with an unavailable message rather than a stale preview.
- Logs, Stop, and Restart are explicit controls on linked tabs. Closing a tab
  never stops its process.

## Persistence

- Topics are stored as one JSON file per topic under `~/.principal/topics/`.
- Document notes are stored under the app's `userData/document-notes/`.
- Registered repositories are maintained by the app's Alexandria registry.
- Managed processes are owned by the main process and are not stored on disk.

## Troubleshooting

| Symptom | Meaning |
|---|---|
| Connection refused on bridge port `3044` or `3054` | The intended production or development app/bridge is not running. Ask the user to launch it; do not silently switch builds. |
| `404 unknown topic id` | The id is not in the local topic registry. |
| `404 unknown note id` | The note id does not exist for the supplied repository/document pair. |
| `400 command "x" is not on the agent allowlist` | The process `command` binary isn't on the allowlist. |
| `400 body.cwd is not inside a registered repository` | Register the repo first (`POST /api/repos`), then start with a `cwd` inside it. |
| A document-open request returns `windowOpened: false` | No suitable already-focused app window could receive it. |
| Browser-open returns `opened: false`, `reason: "no-focused-browser-host"` | Focus the Principal MAIN window, not a Dev Workspace window, and retry. |
| Browser-open returns `opened: false`, `reason: "browser-host-unavailable"` | The renderer did not acknowledge a committed tab within three seconds, or its host became unavailable. |
| Browser-open returns `409` | The managed process is not running with a known port. Start/restart it separately and wait for its port. |

## Workflow skills

| Need | Skill |
|---|---|
| Create a local topic | `create-topic` |
| Read or update a topic description | `topic-context` |
| Find or open a local topic | `search-local-topics` |
| Read or edit document notes | `document-notes` |
| Author or publish a File City tour | `file-city-tours` |
