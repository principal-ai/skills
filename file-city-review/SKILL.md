---
name: file-city-review
description: Walk a user through a PR or branch diff visually inside the running electron-app's File City panel. Build a sequence diagram where each event is a change region with an inline before/after diff snippet, and POST it to the local Principal MCP Bridge so the changes light up the matching buildings with a leader line to a Pierre diff drawer. Use when the user says "review this PR/branch in File City", "walk me through these changes", "show this diff as a flow", or invokes /file-city-review. NOT for runtime traces or architecture diagrams without diffs — use file-city-sequence for those.
---

# File City Review Walkthrough

Turn a set of changes (a branch, a PR, a working tree) into a clickable, ordered tour inside the running electron-app's File City panel. Each event = one change region; clicking it highlights the corresponding building in 3D and opens a Pierre diff snippet showing old → new for just that region.

This skill covers the **review** flavor (`snippet.kind: 'diff'`). For runtime/trace diagrams, use the sister `file-city-sequence` skill.

## When to fire

Fire on phrases like:

- "review this PR / branch / diff in File City"
- "walk me through these changes"
- "show me this commit as a sequence"
- "step through the auth refactor"
- explicit `/file-city-review` invocation

Don't fire when the user is asking for a generic flow diagram with no diff — that's `file-city-sequence`.

## Prerequisites

The electron-app must be running. Endpoints live on the **production** Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, tell the user to launch the app rather than guessing other ports.

The user must have the File City panel open on the repo whose paths your `sourcePath` values reference; otherwise the buildings won't resolve and the diagram renders without highlights (no error, just no cyan fill).

## Authoring workflow

### 1. Decide the scope

Ask (or infer from the request) what the review covers:

- A branch vs. main: `git diff main...HEAD`
- A specific PR: `gh pr diff <num>` or check out the head and diff against base
- The working tree only: `git diff` (uncommitted) or `git diff HEAD` (incl. staged)
- A single commit: `git show <sha>`

Capture both the **base ref** (for `oldContents`) and the **head ref or working tree** (for `newContents`).

### 2. Enumerate change regions

You want one event per *meaningful* change region, not one per hunk. Group hunks that belong to the same logical change in a file. A typical 200-line PR yields 5–15 events.

Get the file list:

```bash
git diff --name-only <base>...<head>
```

For each file, read the per-hunk regions:

```bash
git diff --unified=0 <base>...<head> -- <file>
```

Use `--unified=0` to get tight hunk boundaries, then expand back to readable windows when you set `startLine`/`endLine` on the snippet (typically include 3–10 lines of surrounding context per side).

### 3. Pull old and new contents per event

For each region, you need string content for the diff:

- `oldContents` — required. Pre-change file or pre-sliced window.
  - From a base ref: `git show <base>:<path>`
  - The renderer applies `startLine`/`endLine` to both sides, so you can pass the **full** old file and let the renderer window it.
- `newContents` — optional.
  - If omitted, the renderer reads `event.sourcePath` from disk (the working tree). Use this when the head ref is the user's current checkout.
  - If you're reviewing a remote PR or a different branch, pass `newContents` explicitly: `git show <head>:<path>`.

When in doubt, pass both — it makes the payload self-contained and removes any "what version is on disk?" ambiguity.

### 4. Build the payload

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

For review walkthroughs each event looks like:

```json
{
  "id": "auth-1",
  "name": "review.useAuth.login.errorHandling",
  "label": "useAuth.login() — error handling",
  "participant": "auth",
  "sourcePath": "src/hooks/useAuth.ts",
  "description": "Adds an `if (!res.ok)` guard so callers see a real error instead of a silent resolve. Was the cause of the silent-failure bug in INC-4421.",
  "snippet": {
    "kind": "diff",
    "oldContents": "...full pre-change file or pre-sliced window...",
    "newContents": "...full post-change file or pre-sliced window...",
    "startLine": 12,
    "endLine": 28,
    "focusLine": 18,
    "diffStyle": "unified"
  }
}
```

Field guidance:

- `id` — short, stable, unique. Used by edges and selection state.
- `name` — namespaced dot-path that reads as "what kind of change". Examples: `review.middleware.cors.tighten`, `review.schema.users.add-deleted-at`. The diagram displays this prominently.
- `label` — short human title shown in the snippet drawer header.
- `participant` — swimlane bucket. Use stable buckets across the walkthrough (`auth`, `db`, `api`, `ui`) so related changes stack together.
- `sourcePath` — **repo-relative**, not absolute. Required when `snippet` is set. The renderer maps this to a building in the city; use exactly the path `git diff` printed.
- `description` — markdown. Surfaced in the left-edge floating panel when the event is selected. This is where the *why* lives. Keep it 2–6 sentences and write it for someone who hasn't read the rest of the PR.
- `snippet.startLine`/`endLine` — 1-based, inclusive. Apply to both old and new sides.
- `snippet.focusLine` — the single line you want centered/highlighted. Defaults to `startLine`.
- `snippet.diffStyle` — `'unified'` (default) or `'split'`.

Edges step the reviewer through the tour:

```json
{ "id": "e1", "fromEvent": "auth-1", "toEvent": "auth-2", "label": "then" }
```

Use a single linear chain unless the review legitimately branches (e.g. "frontend path" vs. "backend path" sharing a common origin event).

### 5. Write the summary

Set `payload.summary` to a markdown overview of the whole review — what the PR does, the headline risks, the suggested reading order. This is the first thing the user sees in the left panel before picking an event. Aim for 4–10 lines.

### 6. POST it

```bash
curl -s -X POST http://localhost:3044/api/file-city/sequence \
  -H 'content-type: application/json' \
  -d @payload.json
```

Successful response:

```json
{ "success": true, "id": "<assigned-or-echoed-id>", "broadcastTo": 1, "evictedIds": [], "windowOpened": "created" }
```

- `id` — capture this if the user might iterate on the same review (re-POST with the same `id` to update in place; existing notes on disk are preserved).
- `broadcastTo` — renderer windows that received it. `0` is benign right after the route opens a fresh window — the renderer rehydrates via `getCurrent` on mount.
- `evictedIds` — entries dropped by the library's retention cap. Surface only if relevant.
- `windowOpened` — `'focused'` (existing dev-workspace window for the repo brought to front), `'created'` (new window opened), `'none'` (no `repositoryPath`, repo not registered in Alexandria, or `activate: false` was passed).

Optional body fields:

- `activate` — defaults to `true`. Pass `false` to persist without broadcasting/opening a window — useful when staging a review the user will activate from the sidebar later.

Write the payload to a temp file rather than inlining a large JSON blob into the curl command — the bridge accepts up to a 10MB body but argv has its own limits.

### 7. Tell the user what to do

The dev-workspace window opens itself for registered repos, so guidance is short:

1. If `windowOpened` was `'created'` or `'focused'`, the File City panel is already in front. If it was `'none'`, the repo isn't registered in Alexandria — tell the user to add it, then activate the saved entry from the sidebar.
2. Click an event to step through. Selected event: matching building paints cyan; non-event buildings dim grey so the touched files stand out; a leader line connects the building to a Pierre diff drawer on the right; the description shows in the left markdown overlay.
3. Navigation lives in the **left** overlay (Start → prev/next/position bar). The **right** side, before any event is selected, lists each event-with-source in order so the user can scan the changed files and jump to a step.
4. To clear without deleting the saved review: `curl -X DELETE 'http://localhost:3044/api/file-city/sequence?repositoryPath=<path>'` or close the drawer. To delete the saved review: `DELETE /api/file-city/sequence/:id` (see "Persistence and library").

## Persistence and library

Every accepted POST is persisted to disk, keyed by `id`, and surfaces in the dev-workspace "Diagrams" sidebar. The HTTP surface mirrors what the panel does:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/file-city/sequence` | Create or update (when `id` matches). Activates and opens a window unless `activate: false`. Strips `notes` from the body. |
| `GET` | `/api/file-city/sequence/library?repositoryPath=<abs>` | List saved entries for the repo (plus repo-agnostic ones). Returns `{ entries, activeId }`. |
| `GET` | `/api/file-city/sequence/:id` | Load a saved payload by id. **Returns `notes` inline** — read this when an agent needs the user's review commentary. |
| `POST` | `/api/file-city/sequence/activate` | Body `{ "id": "<id>" }`. Mark the saved entry active and broadcast. |
| `DELETE` | `/api/file-city/sequence/:id` | Permanently delete the saved entry. |
| `GET` | `/api/file-city/sequence?repositoryPath=<abs>` | Read the currently-active payload for the repo. |
| `DELETE` | `/api/file-city/sequence?repositoryPath=<abs>` | Clear the active payload without deleting the saved entry. |

When the user wants to revise a review, prefer re-POSTing with the same `id` over delete-and-recreate — `notes` already on the entry are preserved.

## User notes

`SequenceDiagramPayload.notes` carries user-authored snippet annotations and markdown comments anchored to events on a review. They are **renderer-authored only**:

- HTTP `POST` bodies have `notes` stripped. Don't try to push notes from a script.
- `GET /api/file-city/sequence/:id` returns notes inline. If a teammate has annotated the review you saved, read this endpoint to pick up their comments.
- Re-POSTing a payload by `id` keeps existing notes intact.

## Sharing a review to web-ade

The dev-workspace sidebar can publish a saved review to web-ade so other contributors with GitHub read access to the same repo can pull it. This is a renderer-driven feature (Share button + "Shared with this repo" rows) — there is no HTTP entry point. Worth knowing because:

- Sharing **bakes** every diff snippet's `newContents` from the working tree before uploading. If you authored the review with `newContents` omitted (relying on the renderer reading disk), the bake step still walks `sourcePath` for each event. If a path no longer exists, the share fails with `MISSING_FILES_NEEDS_CONFIRM` until the user retries with "allow missing".
- The bake cap is 10MB on web-ade's side. Tight `startLine`/`endLine` windows keep shared reviews under that.

## Validation rules to obey

The route rejects payloads that violate these — pre-check before posting:

- `events` is a non-empty array; each event has `id` and `name`; event ids are unique within the payload.
- Every `edge.fromEvent` and `edge.toEvent` exists in `events`.
- Every event with `snippet` also has `sourcePath`.
- For `kind: 'diff'`: `oldContents` is a non-empty string; if `newContents` is provided it's a string; `startLine`/`endLine` (when present) are positive integers with `endLine >= startLine`; `diffStyle` (when present) is `'unified'` or `'split'`.
- If `layoutOptions` is set it must be an object; `layoutOptions.laneOrder`, if set, must be an array of strings (unknown entries are tolerated by the renderer).
- If `id` is set it must be a non-empty string.
- Body must stay under 10MB.

If `sourcePath` doesn't resolve to a building in the city, the diagram still renders — the highlight is just skipped. Don't treat that as an error.

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `src/hooks/useAuth.ts`), not absolute, not prefixed with the repo name.
- `repositoryPath` is absolute and identifies which File City panel should receive the broadcast. If you omit it, every open panel reacts. Always set it for review walkthroughs.

## Lane ordering

The diagram resolves each event to a swimlane by reading its dotted `name` (`review.useAuth.login.errorHandling` → lane `review`, drilled deeper if the renderer's `openedNamespaces` matches). Left-to-right lane order is controlled two ways:

- **Default — first-event order.** The lane whose first event appears earliest in your `events` array becomes leftmost. This means the order you author events directly drives layout. For most review walkthroughs this is enough: order events the way the user should read them and the lanes fall out naturally.
- **Explicit — `layoutOptions.laneOrder`.** Pass an array of resolved namespaces left-to-right. Listed lanes are placed first in the given order; any unlisted lanes (including ones that materialize after a drill-down) fall back to first-event order behind them. Unknown entries are ignored, so it is safe to list namespaces that may not be present in every dataset.

```json
{
  "title": "useAuth refactor",
  "events": [...],
  "edges": [...],
  "layoutOptions": {
    "laneOrder": ["client", "auth", "workos", "database"]
  }
}
```

Use explicit `laneOrder` when the read order doesn't match the spatial order you want — e.g. you want `client` always leftmost regardless of which event happens first, or you're rendering a stable reference walkthrough where layout shouldn't drift as you reorder events.

## Common shapes

### Reviewing the current branch vs. main, working tree as "new"

```bash
BASE=$(git merge-base HEAD main)
REPO=$(git rev-parse --show-toplevel)
git diff --name-only "$BASE"...HEAD
# For each file: git show "$BASE:$file" → oldContents; omit newContents (renderer reads disk)
# Build payload with repositoryPath="$REPO"
```

### Reviewing a remote PR you've fetched

```bash
gh pr checkout <num>            # or fetch refs/pull/<num>/head
BASE=$(gh pr view <num> --json baseRefName -q .baseRefName)
# oldContents from "$BASE:$file"; newContents from "HEAD:$file"
```

### Reviewing a single commit

```bash
SHA=<sha>
# oldContents from "$SHA^:$file"; newContents from "$SHA:$file"
```

## Authoring quality bar

Reviews that read well share these traits:

- **Order tells a story.** Sort events so the reader builds intuition as they go — start with the entry point or the highest-risk change, end with cleanup.
- **Descriptions answer "why".** The diff already shows "what". Use `description` to surface the bug being fixed, the constraint that forced the change, the alternative that was rejected.
- **Snippets are tight.** 10–30 lines per side. If a region is bigger, split it into multiple events.
- **Skip noise.** Generated files, lockfile bumps, pure formatting — leave them out unless they're load-bearing for the review.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running, or it's running but bound to dev port 3054. Confirm with the user — this skill targets prod only. |
| `400` with `events must be non-empty` | Forgot to populate `events` or sent the wrong field. |
| `400` with `unknown event id in edge` | Edge references an event id that isn't in the array. Common after renaming ids late. |
| `broadcastTo: 0` | No renderer is listening. App may be starting up or no window is open. |
| Diagram renders, no highlights | `sourcePath` values don't match any building. Check they're repo-relative and the panel is open on the right repo. |
| Diff shows but looks wrong | `startLine`/`endLine` window is approximate when lines are added or removed. Pre-slice both sides into matching windows and drop the window props for exact correspondence. |
| `windowOpened: 'none'` even though `repositoryPath` was set | Repo isn't registered in Alexandria. Auto-open only works for known repos; ask the user to add it from the launcher. |
| Notes posted via curl don't appear | Expected — the route strips `notes` on POST. Notes can only be authored from the renderer. |
| Re-POST creates a duplicate library entry | You didn't pass `id`. Capture `id` from the first response and pass it on subsequent updates. |
| Share fails with `MISSING_FILES_NEEDS_CONFIRM` | Bake step found a `sourcePath` that no longer exists on disk. User must retry with "allow missing", or you should re-author with `newContents` baked in. |

## Reference

- Sister skill for non-diff diagrams: `file-city-sequence`
