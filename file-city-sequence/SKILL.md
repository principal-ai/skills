---
name: file-city-sequence
description: Visualize a runtime trace, request flow, or architecture sequence inside the running electron-app's File City panel. Build a sequence diagram where each event names a step in the flow and (optionally) points at a specific file + line range, then POST it to the local Principal MCP Bridge so clicking a node highlights the matching building in 3D and opens the source snippet in a Pierre drawer. Use when the user says "diagram this flow", "show the sequence of X", "trace this request", "visualize how A calls B", or invokes /file-city-sequence. NOT for reviewing PR diffs — use file-city-review for those.
---

# File City Sequence Diagram

Turn a flow — a request path, a startup sequence, a callback chain, a cross-service handshake — into a clickable, ordered diagram inside the running electron-app's File City panel. Each event = one step; clicking it highlights the corresponding building in 3D and (optionally) opens a Pierre snippet showing the relevant source lines.

This skill covers the **trace/flow** flavor (`snippet.kind: 'slice'` or no snippet). For PR diff walkthroughs, use the sister `file-city-review` skill.

## When to fire

Fire on phrases like:

- "diagram this flow / sequence"
- "show the sequence of how X works"
- "trace this request / callback / lifecycle"
- "visualize how A calls B"
- "draw the auth flow in File City"
- explicit `/file-city-sequence` invocation

Don't fire when the user wants a review of *changes* — that's `file-city-review`.

## Prerequisites

The electron-app must be running. Endpoints live on the **production** Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, tell the user to launch the app rather than guessing other ports.

The user must have the File City panel open on the repo whose paths your `sourcePath` values reference; otherwise the buildings won't resolve and the diagram renders without highlights (no error, just no cyan fill).

## Authoring workflow

### 1. Map out the flow

Before writing a payload, get the steps straight in plain English. A useful flow has:

- A clear **entry point** (the thing that kicks the sequence off).
- A small set of **participants** — stable lane names that bundle related steps. Pick 2–6.
- An **ordered chain** of events, with branches only where the flow genuinely forks.

If you can't articulate participants and order without looking at code, do the trace first (read the entry-point file, follow the calls) and only then build the payload.

### 2. Identify source locations per event

For each step, decide whether it warrants a code snippet:

- **Yes** — the step lives at a specific call site or handler the reader will want to read. Capture `sourcePath` (repo-relative) and a tight line window (`startLine`/`endLine`, ~5–25 lines).
- **No** — the step is conceptual ("user clicks button", "browser issues request", "OS schedules timer"). Leave `sourcePath` and `snippet` off; the event still appears in the diagram, it just won't highlight a building or open a snippet drawer.

Mix freely: a 12-event flow might have 8 events with snippets and 4 conceptual ones.

### 3. Build the payload

```ts
interface SequenceDiagramPayload {
  id?: string;                // stable id; assigned server-side when omitted. Re-POSTing with the same id updates in place. See "Persistence and library".
  title?: string;             // shown in the bottom drawer header
  repositoryPath?: string;    // pass this — let the agent fill in via `git rev-parse --show-toplevel`
  summary?: string;           // markdown — appears in the left-edge panel until an event is picked
  events: FileCitySequenceEventDef[];
  edges: SequenceEdge[];
  layoutOptions?: {
    laneOrder?: string[];     // explicit left-to-right swimlane order; see "Lane ordering" below
  };
  // notes?: SequenceNote[]   // renderer-only — stripped from HTTP POSTs. See "User notes".
}
```

A typical event:

```json
{
  "id": "callback-received",
  "name": "auth.workos.callback.received",
  "label": "WorkOS callback received",
  "participant": "auth-server",
  "sourcePath": "auth-server/src/routes/workos.ts",
  "description": "Express handler for `/auth/workos/callback`. Validates the `state` cookie before exchanging the code — this is where CSRF is enforced.",
  "snippet": {
    "kind": "slice",
    "startLine": 42,
    "endLine": 67,
    "focusLine": 51,
    "contextLines": 2
  }
}
```

Field guidance:

- `id` — short, stable, unique. Used by edges and selection state. Lowercase-kebab reads well (`callback-received`, `token-exchange`).
- `name` — namespaced dot-path that reads as a fully-qualified event identity. Use stable prefixes per flow: `auth.workos.*`, `ingest.batch.*`, `ui.panel.boot.*`. The diagram displays this prominently.
- `label` — short human title for the snippet drawer header. If omitted, the diagram falls back to `name`.
- `participant` — swimlane bucket. Stable across the flow. 2–6 participants reads well; more than ~8 makes the diagram noisy.
- `sourcePath` — **repo-relative** when set. Required if `snippet` is set; optional otherwise.
- `description` — markdown. Surfaced in the left-edge floating panel when the event is selected. This is where the *why* and the surrounding context live. 2–6 sentences.
- `snippet.kind` — `'slice'` (default; can be omitted) for runtime traces. Reads `sourcePath` from disk.
- `snippet.startLine`/`endLine` — 1-based, inclusive. Keep windows tight; long snippets bury the focal line.
- `snippet.focusLine` — the single line you most want the reader to look at. Defaults to `startLine`.
- `snippet.contextLines` — extra buffer above/below the window. Default 2; bump to 4–6 for dense code.

Edges define order:

```json
{ "id": "e1", "fromEvent": "request-received", "toEvent": "auth-checked", "label": "then" }
```

Use `label` to convey edge semantics: `"then"`, `"on success"`, `"on 401"`, `"async"`. For straightforward sequences a single linear chain is enough.

### 4. Write the summary

Set `payload.summary` to a markdown overview of the flow — what triggers it, what completes it, the headline gotchas. This is the first thing the user sees in the left panel before picking an event. 4–10 lines.

### 5. POST it

```bash
curl -s -X POST http://localhost:3044/api/file-city/sequence \
  -H 'content-type: application/json' \
  -d @payload.json
```

Successful response:

```json
{ "success": true, "id": "<assigned-or-echoed-id>", "broadcastTo": 1, "evictedIds": [], "windowOpened": "created" }
```

- `id` — the stable id of the persisted payload. Capture this if you intend to update the same diagram later (re-POST with the same `id` and the entry is replaced in place; existing `notes` on disk are preserved).
- `broadcastTo` — how many renderer windows received the payload. `0` means no panel is listening yet; usually fine because the route may have just opened a window (see `windowOpened`) and the renderer rehydrates via `getCurrent` on mount.
- `evictedIds` — ids dropped by the library's retention policy when this push exceeded the cap. Surface them to the user only if relevant.
- `windowOpened` — `'focused'` (existing dev-workspace window for the repo brought to front), `'created'` (new window opened for it), or `'none'` (no `repositoryPath`, repo not registered in Alexandria, or `activate: false` was passed).

Optional body fields:

- `activate` — defaults to `true`. Pass `false` to persist the payload without broadcasting it or opening a window — useful when prepping a library entry the user will activate later from the sidebar.

Write the payload to a temp file rather than inlining a large JSON blob into the curl command.

### 6. Tell the user what to do

The dev-workspace window opens itself for registered repos, so guidance is short:

1. If `windowOpened` was `'created'` or `'focused'`, the File City panel is already in front. If it was `'none'`, the repo isn't registered in Alexandria yet — tell the user to add it, then activate from the sidebar.
2. Click an event to step through. Selected event: matching building paints cyan; non-event buildings dim to grey so the involved files stand out; a leader line connects the building to a Pierre snippet drawer on the right; the description shows in the left markdown overlay.
3. Navigation lives in the **left** overlay (Start → prev/next/position bar). The **right** side, before any event is selected, lists each event-with-source in order so the user can scan files and jump to a step.
4. To clear without deleting the saved entry: `curl -X DELETE 'http://localhost:3044/api/file-city/sequence?repositoryPath=<path>'` or close the drawer. To delete the saved entry: `DELETE /api/file-city/sequence/:id` (see "Persistence and library").

## Persistence and library

Every accepted POST is persisted to disk under `userData/.../sequence-diagrams`, keyed by `id`. A sidebar "Diagrams" panel in the dev-workspace lists what's saved. The HTTP surface mirrors what the panel does:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/file-city/sequence` | Create or update (when `id` matches an existing entry). Activates and opens the window unless `activate: false`. Strips any `notes` from the body. |
| `GET` | `/api/file-city/sequence/library?repositoryPath=<abs>` | List saved entries for the given repo (plus repo-agnostic ones); `repositoryPath` optional — omitted returns every entry. Returns `{ entries, activeId }`. |
| `GET` | `/api/file-city/sequence/:id` | Load a saved payload by id. Returns the payload **with `notes` inline** — use this when an agent needs to read user-authored context. |
| `POST` | `/api/file-city/sequence/activate` | Body `{ "id": "<id>" }`. Mark the saved entry active for its repo and broadcast it. |
| `DELETE` | `/api/file-city/sequence/:id` | Permanently delete the saved entry. If it was active, fires `PAYLOAD_CLEARED` for its repo. |
| `GET` | `/api/file-city/sequence?repositoryPath=<abs>` | Read the currently-active payload for the repo (or `?repositoryPath=` omitted to get all active payloads). |
| `DELETE` | `/api/file-city/sequence?repositoryPath=<abs>` | Clear the active payload for the repo without deleting the saved entry. |

When updating an existing flow, prefer re-POSTing with the same `id` over deleting and re-creating — `notes` already on the entry are preserved across the update.

## User notes

`SequenceDiagramPayload.notes` carries user-authored snippet annotations and markdown comments anchored to events. They are **renderer-authored only**:

- HTTP `POST` bodies have `notes` stripped during validation. Don't try to push notes from a script — they won't land. (Notes are created via the renderer's note composers and persisted through IPC.)
- `GET /api/file-city/sequence/:id` returns notes inline. When summarizing or diffing a flow, read this endpoint to pick up the user's commentary.
- Re-POSTing a payload by `id` keeps existing notes; ids and content are preserved across updates.

## Validation rules to obey

The route rejects payloads that violate these — pre-check before posting:

- `events` is a non-empty array; each event has `id` and `name`; event ids are unique within the payload.
- Every `edge.fromEvent` and `edge.toEvent` exists in `events`.
- Every event with `snippet` also has `sourcePath`.
- For `kind: 'slice'` (or `kind` omitted): `startLine`/`endLine` are positive integers with `endLine >= startLine`; `focusLine` (when present) is a positive integer; `contextLines` (when present) is non-negative. (`kind: 'diff'` is also accepted by the route but belongs to the `file-city-review` skill.)
- If `layoutOptions` is set it must be an object; `layoutOptions.laneOrder`, if set, must be an array of strings (unknown entries are tolerated by the renderer).
- If `id` is set it must be a non-empty string.
- Body must stay under 10MB.

If `sourcePath` doesn't resolve to a building, the diagram still renders — the highlight is just skipped. Don't treat that as an error.

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `auth-server/src/routes/workos.ts`), not absolute, not prefixed with the repo name.
- `repositoryPath` is absolute and identifies which File City panel should receive the broadcast. If you omit it, every open panel reacts. Always set it.

## Lane ordering

The diagram resolves each event to a swimlane by reading its dotted `name` (`auth.workos.callback.received` → lane `auth`, drilled deeper if the renderer's `openedNamespaces` matches). Left-to-right lane order is controlled two ways:

- **Default — first-event order.** The lane whose first event appears earliest in your `events` array becomes leftmost. For most flows this is enough: order events the way the user should read them and the lanes fall out naturally.
- **Explicit — `layoutOptions.laneOrder`.** Pass an array of resolved namespaces left-to-right. Listed lanes are placed first in the given order; any unlisted lanes (including ones that materialize after a drill-down) fall back to first-event order behind them. Unknown entries are ignored, so it is safe to list namespaces that may not be present in every dataset.

```json
{
  "title": "WorkOS callback flow",
  "events": [...],
  "edges": [...],
  "layoutOptions": {
    "laneOrder": ["client", "auth", "workos", "database"]
  }
}
```

Use explicit `laneOrder` when the temporal/causal order of events doesn't match the spatial layout you want — e.g. you want `client` always leftmost regardless of which event happens first, or you're rendering a stable reference flow where layout shouldn't drift as you reorder events.

## Naming conventions

Good `name` values are namespaced and read as a sentence on their own:

| Flow | Examples |
|---|---|
| OAuth | `auth.workos.callback.received`, `auth.workos.code.exchanged`, `auth.session.created` |
| Ingest pipeline | `ingest.batch.received`, `ingest.batch.validated`, `ingest.batch.persisted` |
| UI boot | `ui.app.bootstrap`, `ui.workspace.layout.loaded`, `ui.panel.first-render` |
| Cross-process IPC | `ipc.invoke.fileSystem.readFile`, `ipc.handler.fileSystem.readFile`, `ipc.return` |

Stick with one prefix per flow. Avoid generic names like `step1`, `handler`, `do-thing`.

## Authoring quality bar

Diagrams that read well share these traits:

- **One thing per event.** If a step contains an `if/else` whose branches diverge, model them as two events with separate edges.
- **Stable participants.** Reusing 3 participants across 12 events makes the diagram clean; using 12 different participants makes it confetti.
- **Snippets land on the right line.** `focusLine` should point at the *call* or *decision* the event is about, not at the surrounding boilerplate.
- **Descriptions answer "why this exists" or "what's surprising here".** Surface the cookie that's checked, the retry that's silent, the race that almost bit you.
- **Order matches reading order.** The first event is where you'd start explaining the flow on a whiteboard.

## Common shapes

### Trace a request through a multi-package monorepo

Capture each hop as its own event with the appropriate `participant`:

```
ui  →  api-gateway  →  auth-service  →  user-service  →  db
```

Each event's `sourcePath` points at the entry function in that hop's package.

### Document a startup/lifecycle sequence

Useful when onboarding contributors. Events run in temporal order: bootstrap → config → service init → first render. Snippets land on the line where each phase begins.

### Bare diagram with no code

If the user wants a conceptual flow ("user clicks 'export', browser downloads file, server logs it"), skip `sourcePath` and `snippet` on most events. The diagram still renders cleanly; you just won't get building highlights.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running, or it's running on dev port 3054. This skill targets prod only — confirm with the user. |
| `400` with `events must be non-empty` | Forgot to populate `events` or sent the wrong field. |
| `400` with `unknown event id in edge` | Edge references an event id that isn't in the array. Common after renaming ids late. |
| `400` with `snippet requires sourcePath` | Set a `snippet` block on an event that has no `sourcePath`. Either drop the snippet or add the path. |
| `broadcastTo: 0` | No renderer is listening. App may be starting up or no window is open. |
| Diagram renders, no highlights | `sourcePath` values don't match any building. Check they're repo-relative and the panel is open on the right repo. |
| Snippet shows wrong lines | `startLine`/`endLine` were 0-based by mistake — they're 1-based, inclusive. |
| `windowOpened: 'none'` even though `repositoryPath` was set | Repo isn't registered in Alexandria. Auto-open only works for known repos; tell the user to add it from the launcher. |
| Notes posted via curl don't appear | Expected — the route strips `notes` on POST. Use the renderer's note composer or the IPC API. |
| Re-POST creates a duplicate library entry | You didn't pass `id`. Without it the server assigns a fresh id; capture the `id` from the first response and pass it on subsequent updates. |

## Reference

- Sister skill for PR diff walkthroughs: `file-city-review`
