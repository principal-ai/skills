---
name: author-local-investigation-trail
description: Author an investigation trail — the exploratory version of a flow, request path, callback chain, or architecture sequence — for the user's own codebase and open it in the standalone trail viewer locally. Investigation trails are the version you lay as you figure something out; they're allowed to carry exploratory titles, a "subject" marker pointing at the answer, and a record of how the answer was found. Writes a TrailPayload JSON to disk and runs `principal-ai trail view --file <path>` — fully local and offline: no publish step, no network, no GitHub token. Use when the user says "investigate this flow locally", "trace this request locally", "lay a local investigation trail through X", "visualize how A calls B locally", or invokes /author-local-investigation-trail.
---

# Author Local Investigation Trail

Turn a flow — a request path, a startup sequence, a callback chain, a cross-service handshake — into a clickable, ordered **investigation trail** in the user's own codebase and open it in the standalone `principal-ai/trail-viewer` locally. Each marker = one stop on the trail; clicking it loads the matching slice snippet showing the relevant source lines.

Investigation trails are the **exploratory** flavor: you lay markers as you trace through the codebase, allowed to wander, allowed to leave dead ends and "let me check…" descriptions, allowed to mark a destination as the **subject**.

This skill writes the payload to a JSON file and opens it in the standalone `principal-ai/trail-viewer` via the `principal-ai` CLI. Everything stays on the local machine — there is no publish, upload, or server step.

**The only delivery channel is the `principal-ai` CLI.** `principal-ai trail view --file <path>` hands the trail off to a running viewer over a Unix socket, or spawns a new viewer locally. The trail stays on disk and in the local viewer — never send the payload anywhere else.

## When to fire

Fire on phrases like:

- "investigate this flow locally"
- "diagram this flow / sequence locally"
- "show the sequence of how X works, view it locally"
- "trace this request / callback / lifecycle locally"
- "visualize how A calls B locally"
- "lay a local trail / investigation through the auth flow"
- "make a local trail for this startup sequence"
- explicit `/author-local-investigation-trail` invocation

Don't fire when:

- The user wants to **publish or share** the trail beyond their own machine — this skill has no upload or publish step; it only writes a local file and opens it in the local viewer.
- The user explicitly asks for a destination other than the local standalone viewer.

## Prerequisites

The CLI runs via npx — nothing has to be installed up front:

```bash
npx -y @principal-ai/principal-view-cli@latest trail view --help
```

Platform support: the bundled trail-viewer ships prebuilt for **macOS arm64** today. Other platforms can pass `--viewer-dir <path>` to a source checkout if they have one; otherwise the CLI exits 2 with a clear message.

What the user does *not* need for this flow:

- Any companion app or background service. The CLI launches the standalone viewer as its own self-contained process.
- A GitHub token. Local mode reads slices straight from the working tree.
- An `<owner>/<repo>` argument. The trail isn't going anywhere; identity doesn't matter.
- Network access. Fully offline.

## The trail schema in 60 seconds

Trails split content from layout:

- **Markers** carry the *content* — `id`, `sourcePath`, `snippet`, `description`. View-agnostic in *shape*, but **the array order is itself meaningful** (see below).
- **Views** carry the *structure* — for v1 always ship `views: [{ kind: 'sequence', markers: [...], edges: [...] }]`. The sequence view block names which markers participate and how they're laid out (lanes, edges).

Why split? The trail medium is designed to grow sibling renderers (linear, graph, tree) without changing marker content. v1 only registers the sequence renderer; ship a single sequence view.

> **Order lives in TWO places — keep them in sync.** The viewer renders a trail
> two ways at once, each ordered by a *different* array:
> - The **sequence diagram** (swimlanes) orders by `views[0].markers` + `edges` +
>   `layout.laneOrder`.
> - The **brief stepper / carousel** (the numbered stops a reader clicks through)
>   orders by the **top-level `payload.markers[]` array**, in array order.
>
> So `payload.markers` order is *not* cosmetic — it is the linear reading order.
> When you reorder a trail, reorder **both** `payload.markers` and
> `views[0].markers`/`edges`, or the stepper and the diagram disagree. Default:
> list `payload.markers` in the exact order you want them read, and mirror that
> order in the view.

Repos: trails carry portable identity, not filesystem paths. For the common single-repo case, ship `authoredAt: { sha, ref }` and omit `repos[]`. The producer machine's filesystem path is passed to the viewer as the `--repo-root` flag at launch — separate from the payload — so slice fetches resolve against the right working tree.

## Authoring workflow

### 1. Map out the flow

Before writing a payload, get the steps straight in plain English. A useful flow has:

- A clear **entry point** (the thing that kicks the trail off).
- A small set of **lanes** — stable names that bundle related markers. Pick 2–6.
- An **ordered chain** of markers, with branches only where the flow genuinely forks.

If you can't articulate the lanes and order without looking at code, do the trace first (read the entry-point file, follow the calls) and only then build the payload.

### 2. Identify source locations per marker

For each step, decide whether it warrants a code snippet:

- **Yes** — the step lives at a specific call site or handler the reader will want to read. Capture `sourcePath` (repo-relative) and a tight line window (`startLine`/`endLine`, ~5–25 lines).
- **No** — the step is conceptual ("user clicks button", "browser issues request", "OS schedules timer"). Leave `sourcePath` and `snippet` off; the marker still appears in the trail, it just won't open a snippet drawer.

Mix freely: a 12-marker trail might have 8 markers with snippets and 4 conceptual ones.

### 3. Pick the subject marker

Investigation trails are allowed (and encouraged) to mark a single marker as the **subject** — the destination the trail directs the reader's focus toward. The cause, the bug location, the answer to the question that triggered the investigation. Set `kind: 'subject'` on that marker.

If the investigation has no clear destination yet — it's still wandering — leave `kind: 'subject'` off. You can add it later by editing the JSON and re-opening the file in the viewer (close the tab with Cmd+W first to force a re-load).

The subject concept is **investigation-only** — it marks where the exploration was headed. If you later refine the trail into a polished, dead-end-free version, drop the subject marker.

### 4. Build the payload

```ts
interface TrailPayload {
  id: string;                     // REQUIRED — producer-supplied (e.g. crypto.randomUUID())
  title: string;                  // REQUIRED — shown in the drawer header
  summary?: string;               // markdown — appears in the left-edge panel until a marker is picked
  purpose?: 'investigation' | 'informative';  // set 'investigation' here
  kind?: string;                  // free-form tag, e.g. 'flow', 'trace', 'lifecycle'
  authoredAt?: { sha: string; ref?: string };  // single-repo provenance shorthand — RECOMMENDED
  repos?: TrailRepo[];            // multi-repo registry; omit for single-repo trails
  markers: TrailMarker[];         // REQUIRED, non-empty
  views: TrailView[];             // REQUIRED, non-empty — ship `[{ kind: 'sequence', ... }]` in v1
  // notes?: TrailNote[]          // renderer-authored only; see "User notes".
  createdAt: string;              // REQUIRED — ISO 8601
  updatedAt: string;              // REQUIRED — ISO 8601
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

The marker that answers the investigation's question:

```json
{
  "id": "csrf-state-mismatch",
  "label": "State mismatch rejects the callback",
  "kind": "subject",
  "sourcePath": "auth-server/src/routes/workos.ts",
  "description": "The `state` cookie comparison fails when the cookie was issued under a different SameSite policy. This is the source of the silent redirect-loop seen in #4291.",
  "snippet": { "kind": "slice", "startLine": 51, "endLine": 58, "focusLine": 53 }
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
  ],
  "layout": { "laneOrder": ["client", "auth-server", "workos"] }
}
```

Field guidance — marker:

- `id` — short, stable, unique. Referenced by edges, notes, and view blocks. Lowercase-kebab reads well (`callback-received`, `token-exchange`).
- `label` — short human title for the snippet drawer header. When omitted, the renderer falls back to the sequence view's `name`.
- `sourcePath` — **repo-relative** when set. Required if `snippet` is set; optional otherwise.
- `kind` — `'subject'` marks the investigation's destination. Investigation-only; drop it when you refine the trail into a polished version.
- `description` — markdown. Surfaced in the left-edge floating panel when the marker is selected. This is where the *why* and the surrounding context live. 2–6 sentences. **Factual, not editorial** — see the quality bar's "No editorializing" rule before writing.
- `snippet.kind` — `'slice'` for runtime traces (this skill). Always set it explicitly.
- `snippet.startLine`/`endLine` — 1-based, inclusive. Keep windows tight; long snippets bury the focal line.
- `snippet.focusLine` — the single line you most want the reader to look at. Defaults to `startLine`.
- `snippet.contextLines` — extra buffer above/below the window. Default 2; bump to 4–6 for dense code.
- `repo` — only set when `payload.repos.length > 1`. References a `TrailRepo.id`.

Field guidance — sequence view ref (`views[0].markers[]`):

- `markerId` — foreign key into `payload.markers[].id`.
- `name` — namespaced dot-path that reads as a fully-qualified marker identity. The renderer derives lanes from the first dotted segment (`auth.workos.callback.received` → lane `auth`). Use stable prefixes per flow: `auth.workos.*`, `ingest.batch.*`, `ui.panel.boot.*`.
- `participant` — explicit lane bucket override. Set this when you want a marker to land in a lane that doesn't match its `name`'s namespace (cross-lane move events, intentional regrouping).
- `moveEvent` — `true` when this marker crosses participant boundaries (the renderer styles it differently).
- `type` — optional event-type string for styling.

Edges (`views[0].edges[]`) define order. The field names mirror the upstream renderer:

```json
{ "id": "e1", "fromEvent": "request-received", "toEvent": "auth-checked", "label": "then" }
```

Use `label` to convey edge semantics: `"then"`, `"on success"`, `"on 401"`, `"async"`. For straightforward sequences a single linear chain is enough. (Yes, the field is `fromEvent`/`toEvent` even though the trail noun is "marker" — this is the upstream renderer's edge type, reused unchanged.)

### 5. Write the summary

Set `payload.summary` to a markdown overview of the flow — what triggers it, what completes it, the headline gotchas. This is the first thing the user sees in the left panel before picking a marker. **3 sentences max.** If you can't fit the whole flow in three sentences, the surplus belongs in marker descriptions, not in the summary.

Investigation summaries are allowed to read as a question or hypothesis ("Why does the WorkOS callback silently redirect-loop on Safari?") rather than a statement. When the investigation reaches an answer, the title and summary can be tightened in place, or the trail can be cloned into an informative one.

**Offer the user title options before opening the viewer.** The title is the most visible part of the trail, and the right framing often depends on audience context only the user has. Once you have a working title, present **2–4 candidate titles** via `AskUserQuestion` and let the user pick, edit, or supply their own — investigation titles may stay exploratory (a question or hypothesis is fine), but keep each to **~140 characters or fewer**; detail beyond that belongs in the summary. Skip the prompt only when the user already handed you an explicit title; when they later ask to change it, treat that as a fresh round of options unless they dictate the exact wording.

**Formatting rules** (the summary renders inside a narrow floating overlay; structural markdown breaks the layout):

- **Paragraph breaks between sentences.** A 3-sentence wall is still a wall in a narrow overlay; three short blocks are scannable. Put each sentence (or each logical beat) on its own paragraph separated by a blank line (`\n\n` in JSON). Don't use single trailing newlines (`\n`) — markdown collapses them into spaces and you get the wall back.
- **Lede labels are encouraged.** Start each paragraph with a 1–3-word bolded anchor that ends in a period: `**Default.**`, `**On click.**`, `**Clean swap.**`, `**Steady state.**`, `**Trigger.**`, `**Surprise.**`, etc. These act as visual scan anchors and label what each paragraph is *about*. Lede labels are a **separate budget** from the emphasis-bold rule below — they don't count against it. Skip them when prose alone already telegraphs the structure (rare for a 3-sentence summary).
- **Inline-code every real identifier.** Component names, state variables, props, file paths, function names — wrap in `` ` ` ``. Anchors the prose to grep-able symbols and visually separates names from prose. *"the card's `highlightLayers` memo"* beats *"the card's highlightLayers memo"*.
- **Emphasis bold sparingly — at most one term per summary**, reserved for the load-bearing concept the reader should walk away with (e.g. `**union**` when contrasting the default state with the narrowed one). Lede labels don't count toward this budget; this rule is about the *body* of a paragraph. More than one emphasis bold dilutes the emphasis to none.
- **No headers, no lists, no links, no code blocks.** A summary that wants any of these has broken the 3-sentence cap. Lists in particular signal that you're enumerating instead of summarizing — push them into marker descriptions.
- **Active voice, present tense.** *"The card paints the layer"*, not *"the layer is painted by the card"* and not *"we made the card paint the layer"*. The summary describes how the code works **now**, not how it was built or investigated.
- **Name real identifiers, not paraphrases.** Write `selectedTrailId`, not *"the current selection"*. Generic nouns (*"the state"*, *"the handler"*, *"the helper"*) are a sign the prose is too abstract to be useful — a reader can't grep for *"the state"*.
- **No first person, no hedging.** Drop *"we"*, *"I"*, *"basically"*, *"essentially"*, *"roughly"*, *"seems to"*, *"more or less"*. The summary states facts, not impressions.
- **Semicolons and em-dashes are fine** for compressing two related clauses into one sentence — better than padding the 3-sentence budget with a short connective sentence.

A useful default shape (not a hard rule): sentence 1 = the **default state** / what the thing does at rest, sentence 2 = the **trigger and the path** it travels when something changes, sentence 3 = the **invariant or detail** that would surprise a reader who stopped after sentence 2.

### 6. Write the payload to a local file

Pick a stable location the user can re-open later. Conventions that work well:

- `<repo-root>/.trails/<flow-name>.json` — keeps the trail next to the code it documents (add to `.gitignore` if it's WIP).
- `~/.principal/local/<flow-name>.json` — user-scoped, survives across repos.
- `/tmp/<flow-name>-trail.json` — throwaway / iteration.

```json
{
  "id": "auth-workos-callback-investigation",
  "title": "Why does the WorkOS callback silently redirect-loop on Safari?",
  "summary": "...",
  "purpose": "investigation",
  "authoredAt": { "sha": "abc1234", "ref": "main" },
  "markers": [...],
  "views": [{ "kind": "sequence", "markers": [...], "edges": [...] }],
  "createdAt": "2026-05-13T18:00:00.000Z",
  "updatedAt": "2026-05-13T18:00:00.000Z"
}
```

For `authoredAt.sha`, use the current commit if you want pinned provenance:

```bash
git rev-parse HEAD
```

For local-only workflows, the sha is informational — slices come from the working tree, not from GitHub. You can leave `authoredAt` off entirely if there's no surrounding context to record.

### 7. Open in the viewer

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

### 8. Iterate

The viewer is single-instance and dedupes by `trailFilePath`:

- Re-running the same `--file <path>` focuses the existing tab (does **not** open a duplicate).
- Editing the JSON and re-running picks up changes — but **only if the tab was closed first** (Cmd+W). Open tabs hold the parsed payload in memory; close the tab and re-open to force a re-load.
- Different `--file` paths each get their own tab.
- The Library tab lists every cached trail under `~/.principal/trails/...` and routes clicks to local mode automatically when the slug decodes to a working tree on disk.

Investigations are explicitly allowed to iterate — that's the point. Lay markers as you trace, re-open to see the chain, add or rename markers, close + re-open to refresh.

To force-quit and start fresh: Cmd+Q the viewer window; the next CLI invocation spawns a new instance.

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

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `auth-server/src/routes/workos.ts`), not absolute, not prefixed with the repo name.
- `--repo-root` is the **absolute filesystem path** the viewer resolves slices against. Pass it explicitly when the user runs the CLI from a different cwd; otherwise the CLI defaults to `process.cwd()`.
- For multi-repo trails, ship `repos: [...]` and set `marker.repo` on every marker. Single-repo trails use the `authoredAt` shorthand and leave `repos[]` and `marker.repo` unset.

## Lane ordering

The sequence renderer resolves each marker's lane by reading its dotted `name` (`auth.workos.callback.received` → lane `auth`). Left-to-right lane order is controlled two ways:

- **Default — first-marker order.** The lane whose first marker appears earliest in `views[0].markers[]` becomes leftmost. For most flows this is enough: order markers the way the user should read them and the lanes fall out naturally.
- **Explicit — `views[0].layout.laneOrder`.** Pass an array of resolved namespaces left-to-right. Listed lanes are placed first in the given order; any unlisted lanes fall back to first-marker order behind them. Unknown entries are ignored, so it is safe to list namespaces that may not be present in every dataset.

```json
{
  "kind": "sequence",
  "markers": [...],
  "edges": [...],
  "layout": { "laneOrder": ["client", "auth", "workos", "database"] }
}
```

Use explicit `laneOrder` when the temporal/causal order doesn't match the spatial layout you want — e.g. `client` always leftmost regardless of which marker happens first.

### First-class actor lanes

When the flow includes participants that aren't markers — `User`, `Browser`, `Database` lanes that exist purely for context — declare them as actors on the sequence view:

```json
{
  "kind": "sequence",
  "actors": [
    { "name": "client", "label": "Browser" },
    { "name": "database", "label": "Postgres" }
  ],
  "markers": [...],
  "edges": [...]
}
```

Actors render as lanes without buildings/snippets — useful for "User → API → DB" sketches where the User and DB lanes are conceptual.

## Naming conventions

Good `name` values (on the sequence view ref) are namespaced and read as a sentence on their own:

| Flow | Examples |
|---|---|
| OAuth | `auth.workos.callback.received`, `auth.workos.code.exchanged`, `auth.session.created` |
| Ingest pipeline | `ingest.batch.received`, `ingest.batch.validated`, `ingest.batch.persisted` |
| UI boot | `ui.app.bootstrap`, `ui.workspace.layout.loaded`, `ui.panel.first-render` |
| Cross-process IPC | `ipc.invoke.fileSystem.readFile`, `ipc.handler.fileSystem.readFile`, `ipc.return` |

Stick with one prefix per flow. Avoid generic names like `step1`, `handler`, `do-thing`.

## Authoring quality bar

Investigation trails are allowed to wander — that's the point. But they still read better when they obey these traits:

- **One thing per marker.** If a step contains an `if/else` whose branches diverge, model them as two markers with separate edges.
- **Stable lanes.** Reusing 3 lanes across 12 markers makes the trail clean; using 12 different lanes makes it confetti.
- **Snippets land on the right line.** `focusLine` should point at the *call* or *decision* the marker is about, not at the surrounding boilerplate.
- **Descriptions answer "why this exists" or "what's surprising here".** Surface the cookie that's checked, the retry that's silent, the race that almost bit you.
- **Subject marker (when you have one) is unambiguous.** Exactly one marker carries `kind: 'subject'`. It's the answer the trail points at, not a stop along the way.
- **No editorializing.** Descriptions are factual statements about behavior, not commentary about the trail. Banned phrasings (non-exhaustive): *"The whole feature."*, *"This is where the magic happens."*, *"Everything hinges on this."*, *"Critically important step."*, *"TL;DR: …"*, *"The key insight is …"*, *"Note that …"*, *"Importantly, …"*. If a marker is load-bearing, **demonstrate** it through specific detail (the branch taken, the value checked, the side-effect emitted) — never **tell** the reader it matters. Show the cookie, don't announce that there is a cookie worth showing. State *what* the code does and *why* a reader would care; do not editorialize *that* they should care. (Investigation trails are allowed *exploratory* language — "let me check…", "first guess was…" — but not editorial flourishes about importance. The two are different rules.)
- **Order matches reading order.** The first marker is where you'd start explaining the flow on a whiteboard.

## Common shapes

### Trace a request through a multi-package monorepo

Capture each hop as its own marker with the appropriate lane via `name`:

```
ui  →  api-gateway  →  auth-service  →  user-service  →  db
```

Each marker's `sourcePath` points at the entry function in that hop's package. All markers live in `payload.markers`; `views[0].markers` references them with the dotted lane names.

### Document a startup/lifecycle sequence

Useful when onboarding contributors. Markers run in temporal order: bootstrap → config → service init → first render. Snippets land on the line where each phase begins.

### Bare trail with no code

If the user wants a conceptual flow ("user clicks 'export', browser downloads file, server logs it"), skip `sourcePath` and `snippet` on most markers. The trail still renders cleanly; you just won't get snippet drawers on those markers.

### Multi-repo trail

Ship `repos: [...]` with one entry per repo, set `marker.repo` on every marker, and leave `authoredAt` off (per-repo `authoredAtSha` lives on each `TrailRepo`):

```json
{
  "repos": [
    { "id": "auth-server",   "name": "auth-server",   "remote": { "host": "github", "owner": "principal-ai", "name": "auth-server"   }, "authoredAtSha": "abc1234" },
    { "id": "api-gateway",   "name": "api-gateway",   "remote": { "host": "github", "owner": "principal-ai", "name": "api-gateway"   }, "authoredAtSha": "def5678" }
  ],
  "markers": [
    { "id": "ingress",   "repo": "api-gateway", "sourcePath": "src/server.ts", ... },
    { "id": "auth-call", "repo": "auth-server", "sourcePath": "src/routes/verify.ts", ... }
  ],
  ...
}
```

Multi-repo local trails are best viewed with the working tree for at least the primary repo available at `--repo-root`; markers anchored to other repos surface read errors on snippet open unless those checkouts are sandboxed under the same root.

## Persistence and library

Trail files live wherever you write them (see step 6). The viewer also reads its **cache library** at `~/.principal/trails/`:

- `~/.principal/trails/local/<path-slug>/<id>.json` — trails authored against a local working tree. The slug is `encodePathForPurl(repoRoot)` (slashes become dashes).
- `~/.principal/trails/by-id/<id>.json` — trails without repo identity.
- `~/.principal/trails/<purl-namespace>/<purl-name>/<id>.json` — hierarchical layout for trails with `repos[0].id` (github/gitlab/etc).

The library tab in the viewer walks this tree and shows every entry. To make a trail show up in the library, either author it directly into the cache layout above, or open it once with `principal-ai trail view --file …` and re-open from the library after.

## User notes

`TrailPayload.notes` carries user-authored snippet annotations and markdown comments. They are **renderer-authored only**:

- Don't hand-author `notes` in the JSON. The viewer's note composer is the entry point.
- The bun host writes mutations back into the JSON file in place: `createTrailNote` appends, `updateTrailNote` mutates by id + bumps `updatedAt`, `deleteTrailNote` filters out. The `{ entry, payload }` wrapper, when present, is preserved on rewrite.
- Re-opening the trail (or closing + reopening the tab) re-reads `notes` from disk.

Investigation notes are especially useful — leave a snippet annotation when a dead end turns up, or a markdown note on the summary when you reach the subject. Notes survive the iteration loop because the bun host re-reads the file on every open.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| Viewer opens with empty markers | `markers` array is empty or every marker is missing `id`. |
| Viewer renders but every marker errors on click | `--repo-root` doesn't match where `sourcePath` values were authored against. |
| `Path escapes repo root: <path>` in snippet drawer | A marker's `sourcePath` resolves outside `--repo-root` (e.g. `../other-repo/x.ts`). Trails are single-repo by design; for multi-repo, ship `repos: [...]` and `marker.repo`. |
| `Render error: undefined is not an object (evaluating 'trail2.views[0]')` | `views` array is missing or empty. v1 trails always need `views: [{ kind: 'sequence', ... }]`. |
| Library-tab click loads in remote mode and snippets fail with "no repo identity" | The decoded path slug doesn't point at a directory on disk. Open via `--file` with explicit `--repo-root`. |
| Viewer doesn't open on second invocation | Already running and the new trail handed off via socket — check the existing window for a new tab. |
| Edit-and-re-open doesn't pick up changes | Tab caches the payload. Close the tab (Cmd+W), re-fire the CLI invocation. |
| `principal-trail-viewer: no prebuilt bundle for <os>-<arch>` | Currently only macOS arm64 is shipped. Pass `--viewer-dir <path>` to a source checkout, or wait for the per-arch fan-out. |
| Lanes appear in unexpected order | First-marker order in `views[0].markers[]` drives default lane layout. Either reorder, or set `views[0].layout.laneOrder` explicitly. |
| Snippet shows wrong lines | `startLine`/`endLine` were 0-based by mistake — they're 1-based, inclusive. |

## Reference

- CLI reference: `principal-ai trail view --help`
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
- Viewer modes design: `principal-view-core-library/docs/TRAIL_VIEWER_MODES.md`
