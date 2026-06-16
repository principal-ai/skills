---
name: file-city-trail-review
description: Walk a user through a PR or branch diff visually inside the running electron-app's File City panel. Build a trail where each marker is a change region with an inline before/after diff snippet, and POST it to the local Principal MCP Bridge so the changes light up the matching buildings with a leader line to a Pierre diff drawer. Use when the user says "review this PR/branch in File City", "walk me through these changes", "show this diff as a trail", "lay a review trail", or invokes /file-city-trail-review. NOT for runtime traces or architecture diagrams without diffs — use author-investigation-trail or author-informative-trail for those.
---

# File City Trail Review (PR walkthrough)

Turn a set of changes (a branch, a PR, a working tree) into a clickable, ordered **trail** inside the running electron-app's File City panel. Each marker = one change region; clicking it highlights the corresponding building in 3D and opens a Pierre diff snippet showing old → new for just that region.

This skill covers the **review** flavor (`snippet.kind: 'diff'`). For runtime/trace trails, use the sister `author-investigation-trail` (exploratory) or `author-informative-trail` (canonical) skills.

## When to fire

Fire on phrases like:

- "review this PR / branch / diff in File City"
- "walk me through these changes"
- "show me this commit as a trail"
- "lay a review trail through the auth refactor"
- "step through this PR"
- explicit `/file-city-trail-review` invocation

Don't fire when the user is asking for a generic flow trail with no diff — that's `author-investigation-trail` (exploratory) or `author-informative-trail` (canonical).

## Prerequisites

The electron-app must be running. Endpoints live on the Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, tell the user to launch the app rather than guessing other ports.

The user must have the File City panel open on the repo whose paths your `sourcePath` values reference; otherwise the buildings won't resolve and the trail renders without highlights (no error, just no cyan fill).

## The trail schema in 60 seconds

Trails split content from layout:

- **Markers** carry the *content* — `id`, `sourcePath`, `snippet`, `description`. View-agnostic.
- **Views** carry the *structure* — for v1 always ship `views: [{ kind: 'sequence', markers: [...], edges: [...] }]`. The sequence view block names which markers participate and how they're laid out (lanes, edges).

Why split? The trail medium is designed to grow sibling renderers (linear, graph, tree) without changing marker content. v1 only registers the sequence renderer; ship a single sequence view.

Repos: trails carry portable identity, not filesystem paths. For the common single-repo case, ship `authoredAt: { sha, ref }` and omit `repos[]`. The producer machine's filesystem path is sent on the request body as a top-level `repositoryPath` field — separate from the payload — so the host can bucket the broadcast and open the right window.

## Authoring workflow

### 1. Decide the scope

Ask (or infer from the request) what the review covers:

- A branch vs. main: `git diff main...HEAD`
- A specific PR: `gh pr diff <num>` or check out the head and diff against base
- The working tree only: `git diff` (uncommitted) or `git diff HEAD` (incl. staged)
- A single commit: `git show <sha>`

Capture both the **base ref** (for `oldContents`) and the **head ref or working tree** (for `newContents`).

### 2. Enumerate change regions

You want one marker per *meaningful* change region, not one per hunk. Group hunks that belong to the same logical change in a file. A typical 200-line PR yields 5–15 markers.

Get the file list:

```bash
git diff --name-only <base>...<head>
```

For each file, read the per-hunk regions:

```bash
git diff --unified=0 <base>...<head> -- <file>
```

Use `--unified=0` to get tight hunk boundaries, then expand back to readable windows when you set `startLine`/`endLine` on the snippet (typically include 3–10 lines of surrounding context per side).

### 3. Pull old and new contents per marker

For each region, you need string content for the diff:

- `oldContents` — required. Pre-change file or pre-sliced window.
  - From a base ref: `git show <base>:<path>`
  - The renderer applies `startLine`/`endLine` to both sides, so you can pass the **full** old file and let the renderer window it.
- `newContents` — optional.
  - If omitted, the renderer reads `marker.sourcePath` via the host's `readFile` action (which v1 resolves against the working tree). Use this when the head ref is the user's current checkout.
  - If you're reviewing a remote PR or a different branch, pass `newContents` explicitly: `git show <head>:<path>`.

When in doubt, pass both — it makes the payload self-contained and removes any "what version is on disk?" ambiguity. It also makes the trail durable when shared to web-ade (the bake step is a no-op if `newContents` is already set).

### 4. Build the payload

```ts
interface TrailPayload {
  id: string;                     // REQUIRED — producer-supplied (e.g. crypto.randomUUID())
  title: string;                  // REQUIRED — shown in the drawer header
  summary?: string;               // markdown — appears in the left-edge panel until a marker is picked
  kind?: string;                  // free-form tag, e.g. 'pr-walkthrough', 'review'
  authoredAt?: { sha: string; ref?: string };  // single-repo provenance shorthand — RECOMMENDED
  repos?: TrailRepo[];            // multi-repo registry; omit for single-repo trails
  markers: TrailMarker[];         // REQUIRED, non-empty
  views: TrailView[];             // REQUIRED, non-empty — ship `[{ kind: 'sequence', ... }]` in v1
  // notes?: TrailNote[]          // renderer-only — stripped from HTTP POSTs. See "User notes".
  createdAt: string;              // REQUIRED — ISO 8601; route fills if you omit
  updatedAt: string;              // REQUIRED — ISO 8601; route fills if you omit
}
```

A typical review marker:

```json
{
  "id": "auth-1",
  "label": "useAuth.login() — error handling",
  "sourcePath": "src/hooks/useAuth.ts",
  "description": "Adds an `if (!res.ok)` guard so callers see a real error instead of a silent resolve. Was the cause of the silent-failure bug in INC-4421.",
  "snippet": {
    "kind": "diff",
    "oldContents": "...full pre-change file or pre-sliced window...",
    "newContents": "...full post-change file or pre-sliced window...",
    "startLine": 12,
    "endLine": 28,
    "focusLine": 18,
    "diffStyle": "unified",
    "language": "typescript"
  }
}
```

The matching `views[0]` entry pins the marker into a swimlane:

```json
{
  "kind": "sequence",
  "markers": [
    {
      "markerId": "auth-1",
      "name": "review.useAuth.login.errorHandling",
      "participant": "auth"
    }
  ],
  "edges": [
    { "id": "e1", "fromEvent": "auth-1", "toEvent": "auth-2", "label": "then" }
  ],
  "layout": { "laneOrder": ["auth", "api", "ui"] }
}
```

Field guidance — marker:

- `id` — short, stable, unique. Referenced by edges, notes, and view blocks.
- `label` — short human title shown in the snippet drawer header.
- `sourcePath` — **repo-relative**, not absolute. Required when `snippet` is set. Use exactly the path `git diff` printed.
- `description` — markdown. Surfaced in the left-edge floating panel when the marker is selected. This is where the *why* lives. Keep it 2–6 sentences and write it for someone who hasn't read the rest of the PR.
- `snippet.kind` — `'diff'` for review walkthroughs (this skill).
- `snippet.oldContents` — required, non-empty string.
- `snippet.newContents` — optional. Omit only when the head ref is the user's working tree.
- `snippet.startLine`/`endLine` — 1-based, inclusive. Apply to both old and new sides.
- `snippet.focusLine` — the single line you want centered/highlighted. Defaults to `startLine`.
- `snippet.contextLines` — defaults to 2; bump for dense code.
- `snippet.diffStyle` — `'unified'` (default) or `'split'`.
- `snippet.language` — syntax-highlight hint (e.g. `'typescript'`, `'python'`).
- `repo` — only set when `payload.repos.length > 1`. References a `TrailRepo.id`.

Field guidance — sequence view ref (`views[0].markers[]`):

- `markerId` — foreign key into `payload.markers[].id`.
- `name` — namespaced dot-path that reads as "what kind of change". The renderer derives lanes from the first dotted segment. Examples: `review.middleware.cors.tighten`, `review.schema.users.add-deleted-at`. Use stable namespacing across the walkthrough so related changes stack.
- `participant` — explicit lane bucket override. Set this when you want a marker to land in a lane that doesn't match its `name`'s namespace.
- `moveEvent` — `true` when this marker crosses participant boundaries.
- `type` — optional event-type string for styling.

Edges (`views[0].edges[]`) step the reviewer through the tour:

```json
{ "id": "e1", "fromEvent": "auth-1", "toEvent": "auth-2", "label": "then" }
```

Use a single linear chain unless the review legitimately branches (e.g. "frontend path" vs. "backend path" sharing a common origin marker). The field names `fromEvent`/`toEvent` are the upstream renderer's edge type, reused unchanged — they reference marker ids despite the historic name.

### 5. Write the summary

Set `payload.summary` to a markdown overview of the whole review — what the PR does, the headline risks, the suggested reading order. This is the first thing the user sees in the left panel before picking a marker. Aim for 4–10 lines.

**Offer the user title options before POSTing.** Present **2–4 candidate titles** for the walkthrough via `AskUserQuestion` and let the user pick, edit, or supply their own; keep each to **~140 characters or fewer**. Skip the prompt only when the user already handed you an explicit title.

### 6. POST it

Send the payload with the producer-machine `repositoryPath` as a top-level field beside it (the host uses it to bucket and open a window — it's not part of the portable payload):

```bash
curl -s -X POST http://localhost:3044/api/file-city/trail \
  -H 'content-type: application/json' \
  -d @payload.json
```

Where `payload.json` looks like:

```json
{
  "id": "useauth-refactor-review",
  "title": "useAuth refactor",
  "kind": "pr-walkthrough",
  "summary": "...",
  "authoredAt": { "sha": "abc1234", "ref": "feature/useauth-refactor" },
  "markers": [...],
  "views": [{ "kind": "sequence", "markers": [...], "edges": [...] }],
  "createdAt": "2026-05-06T18:00:00.000Z",
  "updatedAt": "2026-05-06T18:00:00.000Z",
  "repositoryPath": "/Users/you/code/auth-server"
}
```

Successful response:

```json
{ "success": true, "id": "<id>", "broadcastTo": 1, "evictedIds": [], "windowOpened": "created" }
```

- `id` — echoed back. Re-POSTing with the same `id` updates in place; `notes` already on disk are preserved.
- `broadcastTo` — renderer windows that received it. `0` is benign right after the route opens a fresh window — the renderer rehydrates via `getCurrent` on mount.
- `evictedIds` — entries dropped by the library's retention cap. Surface only if relevant.
- `windowOpened` — `'focused'` (existing dev-workspace window for the repo brought to front), `'created'` (new window opened), `'none'` (no `repositoryPath`, repo not registered in Alexandria, or `activate: false` was passed).

Optional body fields:

- `activate` — defaults to `true`. Pass `false` to persist without broadcasting/opening a window — useful when staging a review the user will activate from the Trails sidebar later.

Write the payload to a temp file rather than inlining a large JSON blob into the curl command — the bridge accepts up to a 10MB body but argv has its own limits, and review payloads with full file `oldContents`/`newContents` get big fast.

### 7. Tell the user what to do

The dev-workspace window opens itself for registered repos, so guidance is short:

1. If `windowOpened` was `'created'` or `'focused'`, the File City panel is already in front. If it was `'none'`, the repo isn't registered in Alexandria — tell the user to add it from the launcher, then activate the saved trail from the Trails sidebar.
2. Click a marker to step through. Selected marker: matching building paints cyan; non-marker buildings dim grey so the touched files stand out; a leader line connects the building to a Pierre diff drawer on the right; the description shows in the left markdown overlay.
3. Navigation lives in the **left** overlay (Start → prev/next/position bar). The **right** side, before any marker is selected, lists each marker-with-source in order so the user can scan the changed files and jump to a stop.
4. To clear without deleting the saved review: `curl -X DELETE 'http://localhost:3044/api/file-city/trail?repositoryPath=<path>'` or close the drawer. To delete the saved review: `DELETE /api/file-city/trail/:id` (see "Persistence and library").

## Persistence and library

Every accepted POST is persisted to disk under `userData/file-city-trails/<bucket>/<id>.json`, keyed by `id`. The "Trails" sidebar in the dev-workspace lists what's saved. The HTTP surface mirrors what the panel does:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/file-city/trail` | Create or update (when `id` matches). Activates and opens a window unless `activate: false`. Strips `notes` from the body. |
| `GET` | `/api/file-city/trail/library?repositoryPath=<abs>` | List saved entries for the repo (plus repo-agnostic ones). Returns `{ entries, activeId }`. |
| `GET` | `/api/file-city/trail/:id` | Load a saved payload by id. **Returns `notes` inline** — read this when an agent needs the user's review commentary. |
| `POST` | `/api/file-city/trail/activate` | Body `{ "id": "<id>" }`. Mark the saved entry active and broadcast. |
| `DELETE` | `/api/file-city/trail/:id` | Permanently delete the saved entry. |
| `GET` | `/api/file-city/trail?repositoryPath=<abs>` | Read the currently-active payload for the repo. |
| `DELETE` | `/api/file-city/trail?repositoryPath=<abs>` | Clear the active payload without deleting the saved entry. |

When the user wants to revise a review, prefer re-POSTing with the same `id` over delete-and-recreate — `notes` already on the entry are preserved.

## User notes

`TrailPayload.notes` carries user-authored snippet annotations (with slice or diff anchors) and markdown comments anchored to marker descriptions or the payload summary. They are **renderer-authored only**:

- HTTP `POST` bodies have `notes` stripped. Don't try to push notes from a script.
- `GET /api/file-city/trail/:id` returns notes inline. If a teammate has annotated the review, read this endpoint to pick up their comments.
- Re-POSTing a payload by `id` keeps existing notes intact.

## Sharing a review to web-ade

The dev-workspace Trails panel can publish a saved review to web-ade so other contributors with GitHub read access to the same repo can pull it. This is a renderer-driven feature (Share button + "Shared with this repo" rows) — there is no HTTP entry point. Worth knowing because:

- Sharing **bakes** every diff snippet's `newContents` from the working tree before uploading. If you authored the review with `newContents` omitted (relying on the renderer reading disk), the bake step still walks `sourcePath` for each marker. If a path no longer exists, the share fails with `MISSING_FILES_NEEDS_CONFIRM` until the user retries with "share anyway".
- Pre-baking `newContents` on the producer side (recommended for review trails) makes the bake step a no-op and the trail durable across renames/moves.
- The bake cap is 10MB on web-ade's side. Tight `startLine`/`endLine` windows keep shared reviews under that — a 200-line PR with full-file `oldContents`/`newContents` on every marker can blow the cap; pre-slice when in doubt.

## Validation rules to obey

The route rejects payloads that violate these — pre-check before posting:

- `id` is a non-empty string.
- `title` is a non-empty string.
- `markers` is a non-empty array; every marker has a string `id`; marker ids are unique within the payload.
- Every marker with `snippet` also has `sourcePath`.
- For `kind: 'diff'`: `oldContents` is a string; if `newContents` is provided it's a string; `startLine`/`endLine` are positive integers with `endLine >= startLine`; `diffStyle` (when present) is `'unified'` or `'split'`.
- `views` is a non-empty array; every view has a string `kind`. v1 only fully validates `kind: 'sequence'` view blocks.
- For `kind: 'sequence'` views: `markers` is an array where every `markerId` references an existing marker and every entry has a string `name`; `edges` is an array where every edge has string `id`, `fromEvent`, `toEvent`.
- Body must stay under 10MB.

If `sourcePath` doesn't resolve to a building in the city, the trail still renders — the highlight is just skipped. Don't treat that as an error.

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `src/hooks/useAuth.ts`), not absolute, not prefixed with the repo name.
- `repositoryPath` is the **absolute filesystem path on the producer machine**. It travels on the request body **alongside** the payload, not inside it. Always set it for review walkthroughs — it's how the host opens the right dev-workspace window. The portable payload itself never carries filesystem paths.
- For multi-repo PRs (rare but possible), ship `repos: [...]` and set `marker.repo` on every marker. Single-repo reviews should use the `authoredAt` shorthand.

## Lane ordering

The sequence renderer resolves each marker's lane by reading its dotted `name` (`review.useAuth.login.errorHandling` → lane `review`, drilled deeper if the renderer's `openedNamespaces` matches). Left-to-right lane order is controlled two ways:

- **Default — first-marker order.** The lane whose first marker appears earliest in `views[0].markers[]` becomes leftmost. For most review walkthroughs this is enough: order markers the way the user should read them.
- **Explicit — `views[0].layout.laneOrder`.** Pass an array of resolved namespaces left-to-right.

```json
{
  "kind": "sequence",
  "markers": [...],
  "edges": [...],
  "layout": { "laneOrder": ["client", "auth", "workos", "database"] }
}
```

Use explicit `laneOrder` when the read order doesn't match the spatial order you want.

## Common shapes

### Reviewing the current branch vs. main, working tree as "new"

```bash
BASE=$(git merge-base HEAD main)
REPO=$(git rev-parse --show-toplevel)
SHA=$(git rev-parse HEAD)
git diff --name-only "$BASE"...HEAD
# For each file: git show "$BASE:$file" → oldContents; omit newContents (renderer reads working tree)
# Build payload with authoredAt={ sha: "$SHA", ref: "<branch-name>" } and request body repositoryPath="$REPO"
```

### Reviewing a remote PR you've fetched

```bash
gh pr checkout <num>            # or fetch refs/pull/<num>/head
BASE=$(gh pr view <num> --json baseRefName -q .baseRefName)
HEAD_SHA=$(git rev-parse HEAD)
# oldContents from "$BASE:$file"; newContents from "HEAD:$file"
# authoredAt={ sha: "$HEAD_SHA", ref: "pr-<num>" }
```

### Reviewing a single commit

```bash
SHA=<sha>
# oldContents from "$SHA^:$file"; newContents from "$SHA:$file"
# authoredAt={ sha: "$SHA" }
```

## Authoring quality bar

Reviews that read well share these traits:

- **Order tells a story.** Sort markers so the reader builds intuition as they go — start with the entry point or the highest-risk change, end with cleanup.
- **Descriptions answer "why".** The diff already shows "what". Use `description` to surface the bug being fixed, the constraint that forced the change, the alternative that was rejected.
- **Snippets are tight.** 10–30 lines per side. If a region is bigger, split it into multiple markers.
- **Skip noise.** Generated files, lockfile bumps, pure formatting — leave them out unless they're load-bearing for the review.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `400` with `markers must be a non-empty array` | Forgot to populate `markers` or sent `events` (sequence-diagram terminology — wrong field). |
| `400` with `views must be a non-empty array` | Forgot the view block. v1 trails always need `views: [{ kind: 'sequence', ... }]`. |
| `400` with `sequence view: unknown markerId` | A `views[0].markers[].markerId` doesn't appear in `payload.markers[].id`. Common after renaming ids late. |
| `400` with `diff snippet requires an oldContents string` | `oldContents` was omitted, undefined, or non-string. It's required for diff snippets. |
| `400` with `id must be a non-empty string` / `title must be a non-empty string` | Producer didn't generate `id` (use `crypto.randomUUID()`) or set `title`. Both are required. |
| `400` with `marker X: snippet requires a sourcePath` | Set a `snippet` block on a marker that has no `sourcePath`. Either drop the snippet or add the path. |
| `broadcastTo: 0` | No renderer is listening. App may be starting up or no window is open. |
| Trail renders, no highlights | `sourcePath` values don't match any building. Check they're repo-relative and the panel is open on the right repo. |
| Diff shows but looks wrong | `startLine`/`endLine` window is approximate when lines are added or removed. Pre-slice both `oldContents` and `newContents` into matching windows and shrink the line-window props for exact correspondence. |
| `windowOpened: 'none'` even though `repositoryPath` was set | Repo isn't registered in Alexandria. Auto-open only works for known repos; ask the user to add it from the launcher. |
| Notes posted via curl don't appear | Expected — the route strips `notes` on POST. Notes can only be authored from the renderer. |
| Re-POST creates a duplicate library entry | Trail `id` is required on every POST. If you're seeing duplicates, you're generating a fresh id each time instead of reusing the one you intend to update. |
| Share fails with `MISSING_FILES_NEEDS_CONFIRM` | Bake step found a `sourcePath` that no longer exists on disk. User must retry with "share anyway", or you should re-author with `newContents` baked in. |

## Reference

- Sister skills for non-diff trails: `author-investigation-trail` (exploratory), `author-informative-trail` (canonical)
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
- Design doc: `industry-themed-file-city-panels/docs/TRAIL_DESIGN.md`
