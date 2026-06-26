---
name: principal-ai-desktop-app-tools
description: Canonical reference for the local HTTP surface the running Principal desktop app (electron-app) exposes to agents — the "Principal MCP Bridge" at http://localhost:3044. Covers how to reach it (the /health probe), the full route surface grouped by domain (File City trails, topics, document notes), and the cross-cutting conventions every call shares (the top-level repositoryPath bucketing field, the 10MB body limit, where data persists on disk, local-and-unauthenticated). Use when you need to know what the desktop app can do over the bridge, what routes exist, why a POST isn't landing in the app, or what the repositoryPath field is for — and as the source the action skills point at instead of re-explaining the bridge. The bridge is local and unauthenticated; web-ade (published, GitHub-gated) is a separate surface — use discover-trails for that. NOT for authoring/curating itself — use author-investigation-trail, author-informative-trail, create-topic, topic-context, file-city-trail-review, or document-notes, each of which owns its routes below.
---

# Principal Desktop App Tools (the local bridge)

The running Principal desktop app (electron-app) exposes a local HTTP server —
the **Principal MCP Bridge** — that agents talk to in order to drive the app:
push a File City trail, create a topic, leave a note on a document, open a file.
This skill is the **canonical reference** for that surface. The action skills
(`author-investigation-trail`, `author-informative-trail`, `create-topic`,
`topic-context`, `file-city-trail-review`, `document-notes`) each own a slice of
it; this is the one place that describes the bridge as a whole.

It is **read-only as a skill** — it explains the surface and routes you to the
action skill that performs the operation. It never POSTs anything itself.

## Local bridge vs web-ade

| | **Local bridge (this skill)** | **web-ade (published)** |
|---|---|---|
| Where | `http://localhost:3044` on this machine | `https://app.principal-ade.com` |
| Auth | **None** — local and unauthenticated | GitHub token (`gh auth token` / git credential helper) |
| Reach it via | `curl` against the running app | the `principal-ai` CLI |
| What's there | trails/topics/notes on *this* machine, local-only until published | trails/topics a user has published and shared |
| Read side | this skill | `discover-trails` |

A local trail/topic is promoted to web-ade by **publishing from the app UI** —
publishing is intentionally *not* a bridge operation.

## Connecting

The electron-app must be running. The bridge listens on:

```
http://localhost:3044
```

Confirm it's up **before any call** — don't guess other ports:

```bash
curl -s http://localhost:3044/health   # → {"status":"ok",...,"port":3044}
```

If that fails (`curl: (7) Failed to connect to localhost port 3044`), the app
isn't running. Ask the user to launch it rather than probing for an alternate
port. No GitHub token is needed: the bridge is local and unauthenticated.

## The route surface

Every route below is served from `http://localhost:3044`. Routes are grouped by
domain; the rightmost column is the skill that owns each domain and documents the
request/response shapes in full.

### File City trails — `author-investigation-trail` · `author-informative-trail` · `file-city-trail-review`

| Method | Route | Purpose |
| --- | --- | --- |
| `POST` | `/api/file-city/trail` | Create or update (when `id` matches an existing entry). Persists, broadcasts `PAYLOAD_SET`, opens/focuses the window with this trail baked into its URL. Strips any `notes` from the body. |
| `GET` | `/api/file-city/trail/library?repositoryPath=<abs>` | List saved entries for the repo (plus repo-agnostic ones); `repositoryPath` optional — omit to return every entry. Returns `{ entries, activeId }`. |
| `GET` | `/api/file-city/trail/:id` | Load a saved payload by id. Returns the payload **with `notes` inline** — use this when an agent needs the user's authored context. |
| `POST` | `/api/file-city/trail/activate` | Body `{ "id": "<id>" }`. Loads the saved entry, broadcasts, opens/focuses the window — same UX as re-POSTing, without re-sending the body. |
| `DELETE` | `/api/file-city/trail/:id` | Permanently delete the saved entry. Windows showing it clear their tab; other entries are unaffected. |
| `GET` | `/api/file-city/trail?repositoryPath=<abs>` | Read the currently-active payload for the repo. |
| `DELETE` | `/api/file-city/trail?repositoryPath=<abs>` | Clear the active payload **without** deleting the saved entry. |

### Topics — `create-topic` (create) · `topic-context` (read + keep-current)

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/topics` | List/search local topics (directory, no trails). Supports `?q=` substring filter. Returns `{ success, topics, count }`. |
| `POST` | `/api/topics` | Create a local topic. Returns `201 { success, topic, trails }`. |
| `GET` | `/api/topics/:id` | Read the topic + its trails. Returns `{ success, topic, trails }`. |
| `POST` | `/api/topics/:id/description/append` | Append a paragraph to the description. Body `{ text }`. |
| `POST` | `/api/topics/:id/description/section` | Replace one `##`/`###` section in place (or append if absent). Body `{ heading, body, level? }`. |
| `POST` | `/api/topics/:id/activate` | Open the topic as a tab in the focused/main window (opens/focuses the window as needed). No body. Returns `{ success, delivered, windowOpened }`. |

Editing a topic's **title or trail list** over the bridge isn't supported — do
that from the app UI. Publishing to web-ade is owner-authenticated and likewise
UI-only.

### Documents — `document-notes`

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/document-notes?repositoryPath=<abs>&relativeFilePath=<rel>` | List notes for a single document. Returns `{ success, repositoryPath, relativeFilePath, notes }`. |
| `GET` | `/api/document-notes/library?repositoryPath=<abs>` | List every file-with-notes (optionally repo-filtered). Returns `{ success, entries }`. |
| `POST` | `/api/document-notes` | Create a note. Body `{ repositoryPath?, relativeFilePath, anchor, body, author? }`. |
| `PATCH` | `/api/document-notes/:noteId` | Update a note's body. Body `{ repositoryPath?, relativeFilePath, body }`. |
| `DELETE` | `/api/document-notes/:noteId?repositoryPath=<abs>&relativeFilePath=<rel>` | Delete a note. |
| `POST` | `/api/document/open` | Open a markdown document into the focused window as a tab. Addresses a `{ repositoryPath, filePath }` doc. |

## Cross-cutting conventions

These hold across every domain above — they're the reason this reference exists
rather than being repeated per skill.

- **`repositoryPath` is a top-level field, not part of the payload.** On POSTs it
  rides on the request body **alongside** the portable payload, never inside it.
  The host uses it to bucket the broadcast and open the **right** dev-workspace
  window. It is the **absolute filesystem path on the producer machine**. The
  portable payload itself never carries filesystem paths (trails carry portable
  identity — `authoredAt: { sha, ref }` — instead).
- **Repo-agnostic bucket.** When `repositoryPath` is omitted or unknown, entries
  land in a repo-agnostic bucket keyed by absolute path. They still work locally
  but aren't portable across machines.
- **10MB body limit.** The bridge accepts up to a 10MB request body, but argv has
  its own limits — write large payloads (e.g. File City review trails with full
  `oldContents`/`newContents`) to a temp file and `curl --data @file` rather than
  inlining the JSON.
- **Persistence on disk.** Trails persist under `~/.principal/trails/<bucket>/<id>.json`,
  keyed by `id` (re-POSTing the same `id` updates in place). Document notes persist
  under the app's `userData/document-notes/`. There is no "currently-active" trail
  tracked on disk — which trail a window shows is per-window state driven by the
  `?openTrailId=` URL arg and `PAYLOAD_SET` broadcasts.
- **Local-only until published.** Everything created over the bridge lives on this
  machine until the user publishes it to web-ade from the app UI.

## Troubleshooting

| Symptom | Meaning |
| --- | --- |
| `curl: (7) Failed to connect to localhost port 3044` | The app isn't running. Ask the user to launch it; don't guess other ports. |
| `404` on a `GET .../:id` | The id doesn't resolve — wrong id, deleted, or (for notes) the `noteId` doesn't exist under the `(repositoryPath, relativeFilePath)` you passed. |
| POST succeeds but nothing appears in the app | Check `repositoryPath` — a wrong/omitted path buckets the broadcast away from the window you're watching. |
| `MISSING_FILES_NEEDS_CONFIRM` on share | A baked `sourcePath` no longer exists on disk; retry "share anyway" or re-author with contents baked in. (Share is a web-ade op, not a bridge op.) |

## Hand-off — pick the action skill

This skill explains the surface; the action skills perform the work.

| You want to… | Use |
|---|---|
| Lay an exploratory trail through a flow | `author-investigation-trail` |
| Author a durable, canonical trail | `author-informative-trail` |
| Walk a PR/branch diff visually | `file-city-trail-review` |
| Create a topic from existing trails | `create-topic` |
| Read / keep a topic's description current | `topic-context` |
| Read or write notes on a markdown doc, or open a doc | `document-notes` |
| Browse trails/topics published to web-ade | `discover-trails` |
