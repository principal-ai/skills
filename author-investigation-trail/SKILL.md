---
name: author-investigation-trail
description: Author an investigation trail — the exploratory version of a flow, request path, callback chain, or architecture sequence — and POST it to the local Principal MCP Bridge so it lands in the running electron-app's File City panel. Investigation trails are the version you lay as you figure something out; they're allowed to carry exploratory titles, a "subject" marker pointing at the answer, and a record of how the answer was found. Use when the user says "investigate this flow", "trace this request", "diagram this sequence", "lay an investigation trail through X", "visualize how A calls B", or invokes /author-investigation-trail. NOT for the durable canonical version — use author-informative-trail (or convert-investigation to fork this trail into one). NOT for PR diff walkthroughs — use file-city-trail-review.
---

# Author Investigation Trail

Turn a flow — a request path, a startup sequence, a callback chain, a cross-service handshake — into a clickable, ordered **investigation trail** inside the running electron-app's File City panel. Each marker = one stop on the trail; clicking it highlights the corresponding building in 3D and (optionally) opens a Pierre slice snippet showing the relevant source lines.

Investigation trails are the **exploratory** flavor: you lay markers as you trace through the codebase, allowed to wander, allowed to leave dead ends and "let me check…" descriptions, allowed to mark a destination as the **subject**. Once you have your answer, you can fork the investigation into a clean informative trail with `convert-investigation` — the original stays untouched as the record of how the answer was found.

This skill covers the **trace/flow** flavor (`snippet.kind: 'slice'` or no snippet). For PR diff walkthroughs, use the sister `file-city-trail-review` skill. For authoring a fresh canonical/durable trail from scratch (no investigation source to fork), use `author-informative-trail`.

## When to fire

Fire on phrases like:

- "investigate this flow"
- "diagram this flow / sequence"
- "show the sequence of how X works"
- "trace this request / callback / lifecycle"
- "visualize how A calls B"
- "lay a trail / investigation through the auth flow"
- "make a trail for this startup sequence"
- explicit `/author-investigation-trail` invocation

Don't fire when:

- The user wants a review of *changes* — that's `file-city-trail-review`.
- The user already has an investigation and wants the canonical version — that's `convert-investigation`.
- The user wants the durable / canonical statement of how something works from scratch (no exploration needed) — that's `author-informative-trail`.

## Prerequisites

The electron-app must be running. Endpoints live on the Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, tell the user to launch the app rather than guessing other ports.

The user must have the File City panel open on the repo whose paths your `sourcePath` values reference; otherwise the buildings won't resolve and the trail renders without highlights (no error, just no cyan fill).

## The trail schema in 60 seconds

Trails split content from layout:

- **Markers** carry the *content* — `id`, `sourcePath`, `snippet`, `description`. View-agnostic in *shape*, but **the array order is itself meaningful** (see below).
- **Views** carry the *structure* — for v1 always ship `views: [{ kind: 'sequence', markers: [...], edges: [...] }]`. The sequence view block names which markers participate and how they're laid out (lanes, edges).

Why split? The trail medium is designed to grow sibling renderers (linear, graph, tree) without changing marker content. v1 only registers the sequence renderer; ship a single sequence view.

> **Order lives in TWO places — keep them in sync.** The panel renders a trail
> two ways at once, each ordered by a *different* array:
> - The **sequence diagram** (swimlanes) orders by `views[0].markers` + `edges` +
>   `layout.laneOrder`.
> - The **brief stepper / carousel** (the numbered stops a reader clicks through)
>   orders by the **top-level `payload.markers[]` array**, in array order.
>
> So `payload.markers` order is *not* cosmetic — it is the linear reading order.
> When you reorder a trail, reorder **both** `payload.markers` and
> `views[0].markers`/`edges`, or the stepper and the diagram disagree and a
> re-POST "looks like it didn't update." Default: list `payload.markers` in the
> exact order you want them read, and mirror that order in the view.

Repos: trails carry portable identity, not filesystem paths. For the common single-repo case, ship `authoredAt: { sha, ref }` and omit `repos[]`. The producer machine's filesystem path is sent on the request body as a top-level `repositoryPath` field — separate from the payload — so the host can bucket the broadcast and open the right window.

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
- **No** — the step is conceptual ("user clicks button", "browser issues request", "OS schedules timer"). Leave `sourcePath` and `snippet` off; the marker still appears in the trail, it just won't highlight a building or open a snippet drawer.

Mix freely: a 12-marker trail might have 8 markers with snippets and 4 conceptual ones.

### 3. Pick the subject marker

Investigation trails are allowed (and encouraged) to mark a single marker as the **subject** — the destination the trail directs the reader's focus toward. The cause, the bug location, the answer to the question that triggered the investigation. Set `kind: 'subject'` on that marker.

If the investigation has no clear destination yet — it's still wandering — leave `kind: 'subject'` off. You can add it later by re-POSTing.

The subject concept is **investigation-only**. When this trail is converted to informative via `convert-investigation`, the subject marker is stripped (the server enforces this).

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
  // notes?: TrailNote[]          // renderer-only — stripped from HTTP POSTs. See "User notes".
  createdAt: string;              // REQUIRED — ISO 8601; route fills if you omit
  updatedAt: string;              // REQUIRED — ISO 8601; route fills if you omit
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
- `kind` — `'subject'` marks the investigation's destination. Investigation-only; stripped on convert.
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

Investigation summaries are allowed to read as a question or hypothesis ("Why does the WorkOS callback silently redirect-loop on Safari?") rather than a statement. When the investigation reaches an answer, the title and summary can be tightened in place, or the trail can be converted to informative for the canonical statement.

**Offer the user title options before POSTing.** The title is the most visible part of the trail, and the right framing often depends on audience context only the user has. Once you have a working title, present **2–4 candidate titles** via `AskUserQuestion` and let the user pick, edit, or supply their own — investigation titles may stay exploratory (a question or hypothesis is fine), but keep each to **~140 characters or fewer**; detail beyond that belongs in the summary. Skip the prompt only when the user already handed you an explicit title; when they later ask to change it, treat that as a fresh round of options unless they dictate the exact wording.

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
  "id": "auth-workos-callback-investigation",
  "title": "Why does the WorkOS callback silently redirect-loop on Safari?",
  "summary": "...",
  "purpose": "investigation",
  "authoredAt": { "sha": "abc1234", "ref": "main" },
  "markers": [...],
  "views": [{ "kind": "sequence", "markers": [...], "edges": [...] }],
  "createdAt": "2026-05-13T18:00:00.000Z",
  "updatedAt": "2026-05-13T18:00:00.000Z",
  "repositoryPath": "/Users/you/code/auth-server"
}
```

Successful response:

```json
{ "success": true, "id": "<id>", "broadcastTo": 1, "evictedIds": [], "windowOpened": "created" }
```

- `id` — echoed back. Re-POSTing with the same `id` updates in place; `notes` already on disk are preserved.
- `broadcastTo` — how many renderer windows received the payload. `0` is benign right after the route opens a fresh window — the new window picks up the trail from the `?openTrailId=` arg baked into its URL by `openDevWorkspaceWindow`.
- `evictedIds` — ids dropped by the library's retention policy when this push exceeded the cap. Surface them to the user only if relevant.
- `windowOpened` — `'focused'` (existing dev-workspace window for the repo brought to front), `'created'` (new window opened for it with this trail already wired in via `?openTrailId=`), `'none'` (no `repositoryPath` or the path isn't a git repo and Alexandria couldn't auto-register it).

Every POST broadcasts and opens a window — there's no "persist quietly without showing it" flag. To bake a trail into the library without activating it, POST it once, then close the trail tab; nothing references "active" anywhere.

Write the payload to a temp file rather than inlining a large JSON blob into the curl command.

### 7. Tell the user what to do

The dev-workspace window opens itself for registered repos, so guidance is short:

1. If `windowOpened` was `'created'` or `'focused'`, the File City panel is already in front. If it was `'none'`, the repo isn't registered in Alexandria yet — tell the user to add it from the launcher, then activate the saved trail from the Trails sidebar.
2. Click a marker to step through. Selected marker: matching building paints cyan; non-marker buildings dim to grey so the involved files stand out; a leader line connects the building to a Pierre snippet drawer on the right; the description shows in the left markdown overlay.
3. Navigation lives in the **left** overlay (Start → prev/next/position bar). The **right** side, before any marker is selected, lists each marker-with-source in order so the user can scan files and jump to a stop.
4. To close the trail tab in the renderer, just close it — there is no "active" pointer to clear. To delete a saved entry from disk: `DELETE /api/file-city/trail/:id` (see "Persistence and library").

## Persistence and library

Every accepted POST is persisted to disk under `~/.principal/trails/<bucket>/<id>.json`, keyed by `id`. The host never tracks a "currently-active" trail on disk — which trail a window is showing is per-window state driven by the `?openTrailId=` URL arg at window creation and IPC `PAYLOAD_SET` broadcasts after that. A "Trails" panel in the dev-workspace sidebar lists what's saved. The HTTP surface mirrors what the panel does:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/file-city/trail` | Create or update (when `id` matches an existing entry). Persists, broadcasts, and opens/focuses the dev-workspace window with this trail's id baked into its URL so the trail tab auto-opens. Strips any `notes` from the body. |
| `GET` | `/api/file-city/trail/library?repositoryPath=<abs>` | List saved entries for the given repo (plus repo-agnostic ones); `repositoryPath` optional — omitted returns every entry. Returns `{ entries }`. |
| `GET` | `/api/file-city/trail/:id` | Load a saved payload by id. Returns the payload **with `notes` inline** — use this when an agent needs to read user-authored context. |
| `POST` | `/api/file-city/trail/activate` | Body `{ "id": "<id>" }`. Loads the saved entry, broadcasts `PAYLOAD_SET`, and opens/focuses the dev-workspace window with this trail's id baked in. Same UX as POSTing the trail again, without re-sending the body. |
| `DELETE` | `/api/file-city/trail/:id` | Permanently delete the saved entry. Windows currently showing this trail clear their tab; saved entries unrelated to the deleted id are unaffected. |

When updating an existing trail, prefer re-POSTing with the same `id` over deleting and re-creating — `notes` already on the entry are preserved across the update.

## User notes

`TrailPayload.notes` carries user-authored snippet annotations (with slice or diff anchors) and markdown comments anchored to marker descriptions or the payload summary. They are **renderer-authored only**:

- HTTP `POST` bodies have `notes` stripped during validation. Don't try to push notes from a script — they won't land. Notes are created via the renderer's note composers and persisted through IPC.
- `GET /api/file-city/trail/:id` returns notes inline. When summarizing or revising a trail, read this endpoint to pick up the user's commentary.
- Re-POSTing a payload by `id` keeps existing notes; ids and content are preserved across updates.

## Sharing a trail to web-ade

The dev-workspace Trails panel can publish a saved trail to web-ade so other contributors with GitHub read access to the same repo can pull it. This is a renderer-driven feature (Share button + "Shared with this repo" rows) — there is no HTTP entry point.

Worth knowing because:

- Sharing **bakes** every diff snippet's `newContents` from the working tree before uploading. Slice snippets stay slim — they're resolved from the reader's checkout — so trace/flow trails (this skill) bake very little.
- The bake cap is 10MB on web-ade's side. Tight `startLine`/`endLine` windows keep shared trails under that.
- If a `sourcePath` no longer exists on disk at share time, the bake step fails with `MISSING_FILES_NEEDS_CONFIRM` until the user retries with "share anyway".

## Validation rules to obey

The route rejects payloads that violate these — pre-check before posting:

- `id` is a non-empty string.
- `title` is a non-empty string.
- `markers` is a non-empty array; every marker has a string `id`; marker ids are unique within the payload.
- Every marker with `snippet` also has `sourcePath`.
- For `kind: 'slice'`: `startLine`/`endLine` are positive integers with `endLine >= startLine`; `focusLine` (when present) is a positive integer; `contextLines` (when present) is non-negative.
- `views` is a non-empty array; every view has a string `kind`. v1 only fully validates `kind: 'sequence'` view blocks; other kinds are accepted but not strictly checked.
- For `kind: 'sequence'` views: `markers` is an array where every `markerId` references an existing marker and every entry has a string `name`; `edges` is an array where every edge has string `id`, `fromEvent`, `toEvent` (and `fromEvent`/`toEvent` should reference valid marker ids).
- Body must stay under 10MB.

If `sourcePath` doesn't resolve to a building in the city, the trail still renders — the highlight is just skipped. Don't treat that as an error.

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `auth-server/src/routes/workos.ts`), not absolute, not prefixed with the repo name.
- `repositoryPath` is the **absolute filesystem path on the producer machine**. It travels on the request body **alongside** the payload, not inside it. Always set it — it's how the host opens the right dev-workspace window. The portable payload itself never carries filesystem paths.
- For multi-repo trails, ship `repos: [...]` and set `marker.repo` on every marker. Single-repo trails should use the `authoredAt` shorthand and leave `repos[]` and `marker.repo` unset.

## Lane ordering

The sequence renderer resolves each marker's lane by reading its dotted `name` (`auth.workos.callback.received` → lane `auth`, drilled deeper if the renderer's `openedNamespaces` matches). Left-to-right lane order is controlled two ways:

- **Default — first-marker order.** The lane whose first marker appears earliest in `views[0].markers[]` becomes leftmost. For most flows this is enough: order markers the way the user should read them and the lanes fall out naturally.
- **Explicit — `views[0].layout.laneOrder`.** Pass an array of resolved namespaces left-to-right. Listed lanes are placed first in the given order; any unlisted lanes (including ones that materialize after a drill-down) fall back to first-marker order behind them. Unknown entries are ignored, so it is safe to list namespaces that may not be present in every dataset.

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

If the user wants a conceptual flow ("user clicks 'export', browser downloads file, server logs it"), skip `sourcePath` and `snippet` on most markers. The trail still renders cleanly; you just won't get building highlights.

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

Multi-repo trails activate one panel per registered repo when broadcast.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `400` with `markers must be a non-empty array` | Forgot to populate `markers` or sent `events` (sequence-diagram terminology — wrong field). |
| `400` with `views must be a non-empty array` | Forgot the view block. v1 trails always need `views: [{ kind: 'sequence', ... }]`. |
| `400` with `sequence view: unknown markerId` | A `views[0].markers[].markerId` doesn't appear in `payload.markers[].id`. Common after renaming ids late. |
| `400` with `every marker must have a string id` | A marker is missing `id` or has a non-string id. |
| `400` with `id must be a non-empty string` / `title must be a non-empty string` | Producer didn't generate `id` (use `crypto.randomUUID()`) or set `title`. Both are required. |
| `400` with `marker X: snippet requires a sourcePath` | Set a `snippet` block on a marker that has no `sourcePath`. Either drop the snippet or add the path. |
| `broadcastTo: 0` | No renderer is listening. App may be starting up or no window is open. |
| Trail renders, no highlights | `sourcePath` values don't match any building. Check they're repo-relative and the panel is open on the right repo. |
| Snippet shows wrong lines | `startLine`/`endLine` were 0-based by mistake — they're 1-based, inclusive. |
| `windowOpened: 'none'` even though `repositoryPath` was set | Repo isn't registered in Alexandria. Auto-open only works for known repos; tell the user to add it from the launcher. |
| Notes posted via curl don't appear | Expected — the route strips `notes` on POST. Use the renderer's note composer or the IPC API. |
| Re-POST creates a duplicate library entry | Trail `id` is required on every POST. If you're seeing duplicates, you're generating a fresh id each time instead of reusing the one you intend to update. |
| Lanes appear in unexpected order | First-marker order in `views[0].markers[]` drives default lane layout. Either reorder, or set `views[0].layout.laneOrder` explicitly. |

## Reference

- Sister skill for the canonical / durable version of a trail: `author-informative-trail`
- Fork this investigation into a clean informative trail with a `derivedFrom` link: `convert-investigation`
- Sister skill for PR diff walkthroughs: `file-city-trail-review`
- Publishing to web-ade: from the Principal app UI
- Sister skill for local-only authoring + viewer (no electron-app required): `author-local-investigation-trail`
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
- Design doc: `industry-themed-file-city-panels/docs/TRAIL_DESIGN.md`
