---
name: local-trails
description: Build a slice/flow trail for the user's own codebase and open it in the standalone trail viewer locally — no web-ade publish, no Principal ADE desktop app. Use when the user says "make a trail for this", "view this local trail", "open my trail file", "trace this flow locally", or invokes /local-trails. Authors a TrailPayload JSON pointing at file + line ranges in the working tree, then runs `principal-ai trail view --file <path> --repo-root <path>`. The viewer is single-instance with tabs and a library; subsequent invocations open new tabs in the running window. NOT for publishing to web-ade — use publish-trail. NOT for opening a teammate's shared trail link — that's `principal-ai trail view <id>` directly.
---

# Local Trails

Build a slice/flow **trail** — markers + sequence view, anchored to file + line ranges — for the user's own codebase and open it in the standalone `principal-ai/trail-viewer` locally. No web-ade round-trip, no GitHub token needed, no `<owner>/<repo>` to anchor.

This skill is for the **local authoring loop**: write a trail JSON, open it, refine, re-open, repeat. The same payload shape is publish-compatible — once the trail is good, run the `publish-trail` skill to share it.

## When to fire

Fire on phrases like:

- "make a trail for this flow"
- "view this local trail"
- "open my trail file"
- "trace this through the codebase locally"
- "build a trail JSON for this"
- explicit `/local-trails` invocation

Don't fire when:

- The user wants to **publish** a trail to web-ade — use `publish-trail`.
- The user is opening a **shared trail link** (`https://app.principal-ade.com/trail/<id>`) — they just need to run `principal-ai trail view <id>` directly; no JSON authoring required.
- The user wants a PR / branch diff walkthrough — that's a separate diff-aware flow.

## Prerequisites

The CLI runs via npx — nothing has to be installed up front:

```bash
npx -y @principal-ai/principal-view-cli@latest trail view --help
```

Platform support: the bundled trail-viewer ships prebuilt for **macOS arm64** today. Other platforms can pass `--viewer-dir <path>` to a source checkout if they have one; otherwise the CLI exits 2 with a clear message.

What the user does *not* need for this flow:

- A GitHub token. Local mode reads slices straight from the working tree.
- An `<owner>/<repo>` argument. The trail isn't going anywhere; identity doesn't matter.
- Network access. Fully offline.

## The trail schema in 60 seconds

A TrailPayload is markers + a sequence view + metadata. Same shape as `publish-trail` — see that skill for full field reference. The minimum:

```ts
interface TrailPayload {
  id: string;                                  // REQUIRED — kebab-case stable id
  title: string;                               // REQUIRED
  summary?: string;                            // markdown overview
  authoredAt?: { sha: string; ref?: string };  // single-repo provenance
  markers: TrailMarker[];                      // REQUIRED, non-empty
  views: TrailView[];                          // REQUIRED — ship `[{ kind: 'sequence', ... }]`
  createdAt: string;                           // ISO 8601
  updatedAt: string;                           // ISO 8601
}
```

A typical marker:

```json
{
  "id": "callback-received",
  "label": "WorkOS callback received",
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

The matching `views[0]` entry pins the marker into a swimlane:

```json
{
  "kind": "sequence",
  "markers": [
    {
      "markerId": "callback-received",
      "name": "auth.workos.callback.received",
      "participant": "auth-server"
    }
  ],
  "edges": [
    { "id": "e1", "fromEvent": "callback-received", "toEvent": "code-exchanged", "label": "then" }
  ]
}
```

`sourcePath` values are **repo-relative** to the `--repo-root` you'll pass to the viewer. If `--repo-root` is `/Users/me/proj` and the marker should resolve to `/Users/me/proj/src/auth.ts`, set `sourcePath: "src/auth.ts"`.

## Authoring workflow

### 1. Map out the flow

Before writing JSON, get the steps straight in plain English. A useful flow has:

- A clear entry point.
- 2–6 stable lanes that bundle related markers.
- An ordered chain, with branches only where the flow genuinely forks.

If you can't articulate the lanes and order without looking at code, do the trace first (read the entry-point file, follow the calls) and only then build the payload.

### 2. Identify source locations per marker

For each step, decide whether it warrants a code snippet:

- **Yes** — the step lives at a specific call site. Capture `sourcePath` (repo-relative) and a tight line window (5–25 lines).
- **No** — the step is conceptual ("user clicks button", "OS schedules timer"). Leave `sourcePath` and `snippet` off; the marker still appears, no snippet drawer.

Mix freely.

### 3. Write the payload to a local file

Pick a stable location the user can re-open later. Conventions that work well:

- `<repo-root>/.trails/<flow-name>.json` — keeps the trail next to the code it documents (add to `.gitignore` if it's WIP).
- `~/.principal/local/<flow-name>.json` — user-scoped, survives across repos.
- `/tmp/<flow-name>-trail.json` — throwaway / iteration.

```json
{
  "id": "auth-workos-callback-flow",
  "title": "WorkOS callback flow",
  "summary": "Walks the WorkOS OAuth callback from the redirect to a signed session.",
  "authoredAt": { "sha": "abc1234", "ref": "main" },
  "markers": [ ... ],
  "views": [
    {
      "kind": "sequence",
      "markers": [ ... ],
      "edges": [ ... ],
      "layout": { "laneOrder": ["client", "auth-server", "workos"] }
    }
  ],
  "createdAt": "2026-05-09T12:00:00.000Z",
  "updatedAt": "2026-05-09T12:00:00.000Z"
}
```

For `authoredAt.sha`, use the current commit if you want pinned provenance:

```bash
git rev-parse HEAD
```

For local-only workflows, the sha is informational — slices come from the working tree, not from GitHub. You can leave `authoredAt` off entirely if there's no surrounding context to record.

### 4. Open in the viewer

```bash
npx -y @principal-ai/principal-view-cli@latest trail view \
  --file /path/to/trail.json \
  --repo-root /path/to/repo
```

What happens:

1. The CLI reads the JSON, decides mode (local because `--file` defaults to local), and assembles env for the viewer.
2. If a viewer is **already running**, the CLI hands the trail off via Unix socket and exits 0. Existing window comes to front, new tab appears.
3. If no viewer is running, the CLI spawns one. First launch self-extracts the bundle (~5s); subsequent launches are near-instant.
4. The viewer opens with two tabs: **Library** (always present, lists cached trails) + the trail you just opened.

`--repo-root` defaults to `cwd` when omitted, so:

```bash
cd /path/to/repo
npx -y @principal-ai/principal-view-cli@latest trail view --file ./trail.json
```

works as long as the user is inside the repo.

### 5. Iterate

The viewer is single-instance and dedupes by `trailFilePath`:

- Re-running the same `--file <path>` focuses the existing tab (does **not** open a duplicate).
- Editing the JSON and re-running picks up changes — but **only if the tab was closed first**. Open tabs hold the parsed payload in memory; close the tab and re-open to force a re-load. (This is a known v1 limitation — there's no in-tab refresh button yet.)
- Different `--file` paths each get their own tab.
- The Library tab lists every cached trail (`~/.principal/trails/...`) plus, indirectly, any local file that's been opened in this session — clicking refocuses the right tab.

To force-quit and start fresh: close the viewer window (or `pkill -f trail-viewer-canary`); the next CLI invocation spawns a new instance.

### 6. (Optional) Publish

When the trail is ready to share, fire the **`publish-trail`** skill. It uses the same payload shape — no schema rewrite, just an `--owner` + `--repo` flag at publish time and a different CLI subcommand (`trail publish`).

## How the viewer resolves slices

In **local mode** (default for `--file`), `readFile` requests from the renderer go to:

```
<repo-root>/<sourcePath>
```

Sandboxed: paths starting with `/` have the leading slash stripped, paths starting with `GitHub/` have that prefix stripped (renderer compat), and any path that escapes `repo-root` is refused with "Path escapes repo root."

If a marker's slice doesn't resolve, the snippet drawer surfaces the read error. Common causes:

- `sourcePath` is absolute when it should be repo-relative.
- `--repo-root` points at the wrong directory.
- The file was renamed / deleted since the trail was authored.
- `startLine`/`endLine` are 0-indexed by mistake (they're 1-based, inclusive).

## Validation rules to obey

The viewer is permissive — a trail with broken sourcePaths still renders, you just see read errors when you click those markers. The hard validation rules to obey:

- `id` is a non-empty string.
- `title` is a non-empty string.
- `markers` is a non-empty array; every marker has a string `id`; marker ids are unique within the payload.
- Every marker with `snippet` also has `sourcePath`.
- For `snippet.kind: 'slice'`: `startLine`/`endLine` are positive integers with `endLine >= startLine`; `focusLine` (when present) is a positive integer; `contextLines` (when present) is non-negative.
- `views` is a non-empty array; ship `[{ kind: 'sequence', markers, edges, layout? }]`.
- For sequence views: every `markerId` references an existing marker; every entry has a string `name`; every edge has string `id`, `fromEvent`, `toEvent` (referencing valid marker ids).

If you plan to publish later, also obey: body under 10MB; `id` is not the reserved word `publish`.

## Authoring quality bar

Same as `publish-trail`:

- **One thing per marker.**
- **Stable lanes** (3–6 across the trail).
- **Snippets land on the right line** — `focusLine` points at the call/decision, not the surrounding boilerplate.
- **Descriptions answer "why this exists" or "what's surprising here"** — surface the cookie that's checked, the retry that's silent, the race that almost bit you.
- **Order matches reading order** — the first marker is where you'd start explaining the flow on a whiteboard.

## Common shapes

### Tracing a request through your repo

Each hop is a marker. `sourcePath` points at the entry function. Lanes group by package/service. Edges go top-to-bottom in temporal order.

```
client → api-route → service-layer → db-call
```

### Documenting a startup or lifecycle sequence

Useful for onboarding contributors. Markers run in temporal order: bootstrap → config load → service init → first render. Snippets land on the line where each phase begins.

### Bare conceptual flow (no code)

If some steps are conceptual ("user clicks button"), skip `sourcePath` and `snippet` on those markers. The trail still renders cleanly; those markers just don't have a snippet drawer.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| Viewer opens with empty markers | `markers` array is empty or every marker is missing `id`. |
| Viewer renders but every marker errors on click | `--repo-root` doesn't match where `sourcePath` values were authored against. |
| `Path escapes repo root: <path>` error in snippet drawer | A marker's `sourcePath` resolves outside `--repo-root` (e.g. `../other-repo/x.ts`). Trails are single-repo by design; for multi-repo, ship `repos: [...]` and `marker.repo`. |
| `Render error: undefined is not an object (evaluating 'trail2.views[0]')` | `views` array is missing or empty. v1 trails always need `views: [{ kind: 'sequence', ... }]`. |
| Viewer doesn't open on second invocation | Already running and the new trail handed off via socket — check the existing window for a new tab. |
| New tab not appearing for the same trail | Dedupe focused the existing tab. Close and reopen if you need a fresh load. |
| `principal-trail-viewer: no prebuilt bundle for <os>-<arch>` | Currently only macOS arm64 is shipped. Pass `--viewer-dir <path>` to a source checkout, or wait for the per-arch fan-out. |
| Edit-and-re-open doesn't pick up changes | Tab caches the payload. Close the tab, re-fire the CLI invocation. |

## Reference

- Publish flow (when ready to share): `publish-trail` skill.
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`.
- Viewer modes design: `principal-view-core-library/docs/TRAIL_VIEWER_MODES.md`.
- Deployment: `principal-view-core-library/docs/TRAIL_VIEWER_DEPLOYMENT.md`.
