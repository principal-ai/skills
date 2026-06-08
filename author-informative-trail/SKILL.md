---
name: author-informative-trail
description: Author a fresh informative trail — the durable, canonical version of a flow or insight — and POST it to the local Principal MCP Bridge so it lands in the running electron-app's File City panel. Informative trails state what is true about the code now; they are not a record of how the answer was found. Use when the user says "make an informative trail", "author a canonical trail", "lay a durable trail through X", "write a sign-off trail for this flow", or invokes /author-informative-trail. NOT for the exploratory version you lay as you figure something out — use author-investigation-trail. NOT for forking an existing investigation — use convert-investigation. NOT for PR diff walkthroughs — use file-city-trail-review.
---

# Author Informative Trail

Author a fresh **informative** trail from scratch — the durable, canonical version of a flow, behavior, or invariant. Informative trails are the version that gets shared and signed off on. They state *what is true* about the code; they are not a log of how the answer was discovered.

This skill is for **authoring from scratch**. If you already have an investigation trail and want to mint its canonical sibling, use `convert-investigation` instead — that path forks + links, this path doesn't. If you're still exploring as you go, use `author-investigation-trail` and convert later.

## When to fire

Fire on phrases like:

- "make an informative trail for this flow"
- "author a canonical trail through the auth path"
- "write a durable trail for this lifecycle"
- "lay a sign-off trail for X"
- "informative trail of how Y works"
- explicit `/author-informative-trail` invocation

Don't fire when:

- The user wants to **explore / investigate** a flow as they figure it out — that's `author-investigation-trail`.
- The user wants to author + view the trail **locally** in the standalone trail viewer (no electron-app) — that's `author-local-investigation-trail` (local authoring is investigation-flavor only).
- The user has an existing investigation trail and wants to **convert** it — that's `convert-investigation`.
- The user wants a **PR diff walkthrough** — that's `file-city-trail-review`.
- The user wants to **publish to web-ade** as a shareable link — publish from the Principal app UI. Authoring first and publishing later is fine; this skill handles the authoring half.

## Prerequisites

The electron-app must be running. Endpoints live on the **production** Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, ask the user to launch the app rather than guessing other ports.

The File City panel must be open on the repo whose paths your `sourcePath` values reference; otherwise the trail renders without building highlights (no error, just no cyan fill).

## Inputs

The user supplies the **subject** of the trail — the flow, behavior, or invariant the trail will describe ("how the titlebar selection opens the project-info tab", "the WorkOS callback flow", "what happens when a dashboard metric refreshes"). Everything else you resolve from the codebase.

If the user hands you an investigation trail id and says "make this informative", stop and route to `convert-investigation` — that's the fork+link path, and bypassing it loses the `derivedFrom` link.

## The trail schema in 60 seconds

Trails split content from layout:

- **Markers** carry the *content* — `id`, `sourcePath`, `snippet`, `description`. View-agnostic.
- **Views** carry the *structure* — for v1 always ship `views: [{ kind: 'sequence', markers: [...], edges: [...] }]`. The sequence view block names which markers participate and how they're laid out (lanes, edges).

Why split? The trail medium is designed to grow sibling renderers (linear, graph, tree) without changing marker content. v1 only registers the sequence renderer; ship a single sequence view.

Repos: trails carry portable identity, not filesystem paths. For the common single-repo case, ship `authoredAt: { sha, ref }` and omit `repos[]`. The producer machine's filesystem path is sent on the request body as a top-level `repositoryPath` field — separate from the payload — so the host can bucket the broadcast and open the right window.

## Authoring workflow

### 1. Trace the flow in the codebase

Before writing a payload, get the steps straight by reading source. A good informative trail has:

- A clear **entry point** — the place a reader would start if you handed them a whiteboard marker.
- A small set of **lanes** (2–6) that bundle related markers by participant or subsystem.
- An **ordered chain** that walks the reader from entry → answer without detours.

Do the trace first. If you can't name the lanes and the order without looking at code, you're not ready to author — switch to `author-investigation-trail`, figure it out, then convert.

### 2. Identify the headline

The informative trail has a *headline* — the load-bearing fact the reader walks away with. The trail's title should be a statement of that fact (not a question, not an exploration). The summary should restate it concisely. The first marker the reader hits should either *be* the headline or walk them to it without detours.

If you can't articulate the headline in one sentence, you don't have an informative trail yet — you have an investigation. Either complete the investigation first, or use `author-investigation-trail` to author the investigative version.

### 3. Build the payload

```ts
interface TrailPayload {
  id: string;                     // REQUIRED — producer-supplied (e.g. crypto.randomUUID())
  title: string;                  // REQUIRED — shown in the drawer header
  summary?: string;               // markdown — appears in the left-edge panel until a marker is picked
  purpose?: 'investigation' | 'informative';  // set 'informative' here
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

Field guidance — informative-specific:

- `id` — fresh, unique. Generate with `crypto.randomUUID()` or a stable kebab id like `<headline-slug>-informative`.
- `title` — **a statement of what's true**, not an exploration. Good: *"Titlebar selection opens project-info tab via repository-selected event."* Bad: *"Looking into titlebar selection behavior"* or *"Titlebar selection"*. The title is the headline.
- `summary` — concise statement of the durable insight, not the path to it. **3 sentences max.** Drop "I was investigating…", "we figured out that…", "it turns out…" phrasing. See "Write the summary" below for full formatting rules.
- `purpose: 'informative'` — set it explicitly. Marks the trail as durable / canonical for any tooling that distinguishes the two flavors.
- `markers` — tight. 5–8 is typical; 12 is already long. Each marker is one decision, one branch, one handler — not "the area around this region." **Do not emit `kind: 'subject'` on any marker** — that's an investigation-only concept.
- `views` — ship `[{ kind: 'sequence', markers, edges, layout? }]`.
- `authoredAt: { sha, ref }` — capture the producer-machine commit you authored against. Recommended for single-repo trails.
- `repos[]` — only for multi-repo trails; otherwise omit and use `authoredAt`.
- `createdAt` / `updatedAt` — ISO 8601 now; the route will fill if you omit.
- `notes` — never emit. Notes are renderer-authored only and stripped by the route.

A typical marker:

```json
{
  "id": "repository-selected-event",
  "label": "Titlebar emits repository-selected",
  "sourcePath": "principal-ade-desktop/src/components/Titlebar.tsx",
  "description": "Click handler on the repo chip emits `repository-selected` with the new `repositoryPath`. The dev-workspace listens on the same bus and routes to `useProjectInfoTab`.",
  "snippet": {
    "kind": "slice",
    "startLine": 84,
    "endLine": 102,
    "focusLine": 91,
    "contextLines": 2
  }
}
```

Marker field guidance:

- `id` — short, stable, unique. Referenced by edges, notes, and view blocks. Lowercase-kebab reads well (`repository-selected-event`, `project-info-tab-open`).
- `label` — short human title for the snippet drawer header. When omitted, the renderer falls back to the sequence view's `name`.
- `sourcePath` — **repo-relative** when set. Required if `snippet` is set; optional otherwise.
- `description` — markdown, 2–6 sentences. Factual, present-tense, identifier-anchored. See the Quality bar's "no editorializing" rule.
- `snippet.kind` — `'slice'` for trace/flow trails (this skill). Always set it explicitly.
- `snippet.startLine`/`endLine` — 1-based, inclusive. Keep windows tight (5–25 lines).
- `snippet.focusLine` — the single line you most want the reader to look at. Defaults to `startLine`.
- `snippet.contextLines` — extra buffer above/below the window. Default 2.
- `repo` — only set when `payload.repos.length > 1`. References a `TrailRepo.id`.

Sequence view ref (`views[0].markers[]`):

```json
{
  "kind": "sequence",
  "markers": [
    {
      "markerId": "repository-selected-event",
      "name": "ui.titlebar.repository-selected",
      "participant": "ui-titlebar"
    }
  ],
  "edges": [
    { "id": "e1", "fromEvent": "repository-selected-event", "toEvent": "project-info-tab-open", "label": "then" }
  ],
  "layout": { "laneOrder": ["ui-titlebar", "state", "ui-project-info"] }
}
```

- `markerId` — foreign key into `payload.markers[].id`.
- `name` — namespaced dot-path. The renderer derives lanes from the first dotted segment (`ui.titlebar.repository-selected` → lane `ui`). Use stable prefixes per flow.
- `participant` — explicit lane override. Set when a marker should land in a lane that doesn't match its `name` namespace.
- `moveEvent` — `true` when this marker crosses participant boundaries (styled differently).
- `type` — optional event-type string for styling.

Edges define order. Field names mirror the upstream renderer (`fromEvent`/`toEvent` even though the trail noun is "marker"). Use `label` to convey edge semantics: `"then"`, `"on success"`, `"on 401"`, `"async"`.

### 4. Write the summary

Set `payload.summary` to a markdown overview of the flow — what triggers it, what completes it, the headline gotchas. This is the first thing the user sees in the left panel before picking a marker. **3 sentences max.** If you can't fit the whole flow in three sentences, the surplus belongs in marker descriptions, not in the summary.

Informative summaries read as **statements**, not questions or hypotheses. Drop "I was investigating…", "the question was…", "it turns out…" — state what's true.

**Formatting rules** (the summary renders inside a narrow floating overlay; structural markdown breaks the layout):

- **Paragraph breaks between sentences.** A 3-sentence wall is still a wall in a narrow overlay; three short blocks are scannable. Put each sentence on its own paragraph separated by a blank line (`\n\n` in JSON). Don't use single trailing newlines (`\n`) — markdown collapses them into spaces.
- **Lede labels are encouraged.** Start each paragraph with a 1–3-word bolded anchor that ends in a period: `**Default.**`, `**On click.**`, `**Clean swap.**`, `**Steady state.**`, `**Trigger.**`, `**Surprise.**`. These act as visual scan anchors. Lede labels are a **separate budget** from the emphasis-bold rule — they don't count against it.
- **Inline-code every real identifier.** Component names, state variables, props, file paths, function names — wrap in `` ` ` ``. *"the card's `highlightLayers` memo"* beats *"the card's highlightLayers memo"*.
- **Emphasis bold sparingly — at most one term per summary**, reserved for the load-bearing concept the reader should walk away with. Lede labels don't count toward this budget.
- **No headers, no lists, no links, no code blocks.** A summary that wants any of these has broken the 3-sentence cap.
- **Active voice, present tense.** *"The card paints the layer"*, not *"the layer is painted"* and not *"we made the card paint"*. Describes how the code works **now**, not how it was built or investigated.
- **Name real identifiers, not paraphrases.** Write `selectedTrailId`, not *"the current selection"*. Generic nouns (*"the state"*, *"the handler"*) signal prose too abstract to be useful.
- **No first person, no hedging.** Drop *"we"*, *"I"*, *"basically"*, *"essentially"*, *"seems to"*.
- **Semicolons and em-dashes are fine** for compressing two related clauses into one sentence.

A useful default shape (not a hard rule): sentence 1 = the **default state** / what the thing does at rest, sentence 2 = the **trigger and the path** it travels when something changes, sentence 3 = the **invariant or detail** that would surprise a reader who stopped after sentence 2.

### 5. POST it

Write the body to a temp file (don't inline a large JSON blob into the curl command):

```bash
curl -s -X POST http://localhost:3044/api/file-city/trail \
  -H 'content-type: application/json' \
  -d @payload.json
```

Where `payload.json` is the `TrailPayload` plus a top-level `repositoryPath`:

```json
{
  "id": "titlebar-selection-opens-project-info-informative",
  "title": "Titlebar selection opens project-info tab via repository-selected event",
  "summary": "...",
  "purpose": "informative",
  "authoredAt": { "sha": "abc1234", "ref": "main" },
  "markers": [ ... ],
  "views": [{ "kind": "sequence", "markers": [...], "edges": [...] }],
  "createdAt": "2026-05-13T12:00:00.000Z",
  "updatedAt": "2026-05-13T12:00:00.000Z",
  "repositoryPath": "/Users/you/code/principal-ade-desktop"
}
```

`repositoryPath` is the **absolute filesystem path on this machine**. It travels on the request body alongside the payload, not inside it — the host uses it to bucket the broadcast and open the right dev-workspace window. The portable payload never carries filesystem paths.

### 6. Handle the response — loop on validation errors

Success looks like:

```json
{ "success": true, "id": "<id>", "broadcastTo": 1, "evictedIds": [], "windowOpened": "created" }
```

Report the `id` and `windowOpened` to the user and stop.

- `windowOpened: 'focused'` — existing dev-workspace window for the repo was brought to front.
- `windowOpened: 'created'` — new window opened with this trail auto-loaded via `?openTrailId=`.
- `windowOpened: 'none'` — repo isn't registered in Alexandria. Tell the user to add it from the launcher, then activate the saved trail from the Trails sidebar.

Failure: read `error`, fix the field in your payload, POST again. Common cases:

| `error` substring | Fix |
|---|---|
| `markers must be a non-empty array` | You stripped too much. Keep at least one marker. |
| `every marker must have a string id` | A marker is missing `id`. |
| `duplicate marker id` | You reused an id. Make them unique within the payload. |
| `unknown markerId` | A `views[0].markers[].markerId` doesn't match any `payload.markers[].id`. Usually after renaming ids late. |
| `marker X: snippet requires a sourcePath` | A marker has `snippet` but no `sourcePath`. Either drop the snippet or add the path. |
| `id must be a non-empty string` | You forgot to set `payload.id`. |
| `title must be a non-empty string` | You forgot `payload.title`. |
| `views must be a non-empty array` | Forgot the view block. Always ship `views: [{ kind: 'sequence', ... }]`. |

**Stop after 5 attempts** if you keep hitting validation errors. Report the last error to the user — there's likely something they need to clarify.

## Validation rules to obey

The route rejects payloads that violate these — pre-check before posting:

- `id` is a non-empty string.
- `title` is a non-empty string.
- `markers` is a non-empty array; every marker has a string `id`; marker ids are unique within the payload.
- Every marker with `snippet` also has `sourcePath`.
- For `kind: 'slice'`: `startLine`/`endLine` are positive integers with `endLine >= startLine`; `focusLine` (when present) is a positive integer; `contextLines` (when present) is non-negative.
- `views` is a non-empty array; every view has a string `kind`.
- For `kind: 'sequence'` views: every `markerId` references an existing marker and every entry has a string `name`; `edges` is an array where every edge has string `id`, `fromEvent`, `toEvent`.
- Body must stay under 10MB.

## Quality bar

This is the heart of the skill. The mechanics above are the same as any trail POST; what makes a trail *informative* is the discipline below. A good informative trail:

- **Reads as a statement, not a question.** Title, summary, and first marker all assert what's true. If the title is *"Looking into…"* or *"Why does X…"*, you're authoring an investigation, not an informative trail.
- **Leads with the answer.** The reader should not have to walk the whole chain to learn the headline. Either the first marker *is* the answer (then the rest is supporting structure), or the markers are ordered so a reader who stops after marker 1–2 has the gist.
- **Is tight.** 5–8 markers is the common range. 12 is already long. Every marker earns its place by carrying a decision, branch, handler, side-effect, or invariant — not "this region is involved." If you find yourself writing *"and this file also matters"*, that's not a marker.
- **States facts, never editorializes.** Descriptions are factual statements about what the code does. Banned phrasings (non-exhaustive): *"The whole feature."*, *"This is where the magic happens."*, *"Everything hinges on this."*, *"Critically important step."*, *"TL;DR: …"*, *"The key insight is …"*, *"Note that …"*, *"Importantly, …"*, *"Crucially, …"*. If a marker is load-bearing, **demonstrate** it through specific detail (the branch taken, the value checked, the event emitted) — never **tell** the reader it matters. Show the cookie, don't announce that there is a cookie worth showing.
- **Names real identifiers.** Write `selectedTrailId`, `repository-selected`, `useProjectInfoTab` — not *"the state"*, *"the event"*, *"the hook"*. A reader can grep for a real identifier; they can't grep for *"the helper"*. Inline-code every identifier in both summary and descriptions.
- **Has no investigation residue.** Strip *"I was checking…"*, *"It turns out…"*, *"After some digging…"*, *"My first guess was…"*, *"This took a while to find…"*. The informative trail does not narrate the discovery.
- **Has no dead ends.** Investigations are allowed to wander; informative trails are not. Every marker is on the path from entry to answer. If a marker is a "we considered this but it's not the cause" stop, delete it.
- **Has stable lanes.** 3 lanes reused across 8 markers reads clean; 8 different lanes reads as confetti. Pick lanes by participant or subsystem (`ui.titlebar.*`, `state.repository.*`, `ui.project-info.*`) and reuse them.
- **Has snippets on the right line.** `focusLine` points at the *call*, *decision*, or *emission* the marker is about — not the function signature or surrounding boilerplate. Windows are tight (5–25 lines).
- **Has descriptions that answer "why this exists" or "what's surprising here"** — the cookie that's checked, the retry that's silent, the race that almost bit you. 2–6 sentences. Active voice, present tense. Same formatting discipline as the summary (inline-code identifiers, no headers/lists, no first person).
- **Reads in the order a reader should learn it.** The first marker is where you'd start explaining the flow on a whiteboard. Edge labels (`"then"`, `"on success"`, `"on 401"`, `"async"`) carry the causal structure.

## Checklist before POSTing

Run through this list. If any answer is "no" or "not sure," fix it before posting:

- [ ] `title` reads as a statement of what's true (not a question, not "Looking into…").
- [ ] `summary` is ≤3 sentences, paragraph-broken, and states the answer (not the journey).
- [ ] `purpose: 'informative'` is set explicitly.
- [ ] Every marker is on the path from entry to answer; no dead ends.
- [ ] No marker carries `kind: 'subject'`.
- [ ] Marker descriptions are factual — none of the banned editorial phrasings.
- [ ] Identifiers in summary and descriptions are inline-coded and real (not paraphrases).
- [ ] Lanes are stable and named by participant/subsystem; no `step1`/`handler`/`do-thing` placeholders.
- [ ] `sourcePath` values are repo-relative; `repositoryPath` (top-level, beside the payload) is the absolute path on this machine.
- [ ] `authoredAt: { sha, ref }` is set for single-repo trails; or `repos[]` + per-marker `repo` for multi-repo.

## What you're not doing

- **Not investigating.** Informative trails describe what's true now. If you're discovering it as you go, switch to `author-investigation-trail` and convert later with `convert-investigation`.
- **Not narrating the discovery.** No *"I noticed…"*, no *"after tracing through…"*, no *"the bug was…"*. The trail does not reference itself or its author.
- **Not capturing every related file.** Density matters more than coverage. A 6-marker trail that hits the load-bearing decisions beats a 14-marker trail that includes every file the flow touches.
- **Not setting `kind: 'subject'` on any marker.** That's an investigation-only concept for marking the destination during exploration. Informative trails lead with the answer instead.
- **Not publishing.** This skill writes to the local library and broadcasts to the running app. To share publicly, publish from the app UI once the trail is good.
- **Not emitting `notes`.** Notes are renderer-authored only and the route strips them on POST.

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `auth-server/src/routes/workos.ts`), not absolute, not prefixed with the repo name.
- `repositoryPath` is the **absolute filesystem path on the producer machine**. It travels on the request body **alongside** the payload, not inside it.
- For multi-repo trails, ship `repos: [...]` and set `marker.repo` on every marker. Single-repo trails use the `authoredAt` shorthand and leave `repos[]` and `marker.repo` unset.

## Lane ordering

The sequence renderer resolves each marker's lane by reading its dotted `name` (`ui.titlebar.repository-selected` → lane `ui`). Left-to-right order is controlled two ways:

- **Default — first-marker order.** The lane whose first marker appears earliest in `views[0].markers[]` becomes leftmost. For most flows this is enough: order markers the way the user should read them and the lanes fall out naturally.
- **Explicit — `views[0].layout.laneOrder`.** Pass an array of resolved namespaces left-to-right. Listed lanes are placed first in the given order; unlisted lanes fall back to first-marker order behind them. Unknown entries are ignored.

```json
{
  "kind": "sequence",
  "markers": [...],
  "edges": [...],
  "layout": { "laneOrder": ["client", "auth", "workos", "database"] }
}
```

Use explicit `laneOrder` when the temporal/causal order doesn't match the spatial layout you want.

### First-class actor lanes

When the flow includes participants that aren't markers — `User`, `Browser`, `Database` lanes for context — declare them as actors on the sequence view:

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

Actors render as lanes without buildings/snippets.

## Persistence and library

Every accepted POST is persisted to disk under `~/.principal/trails/<bucket>/<id>.json`, keyed by `id`. The HTTP surface:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/file-city/trail` | Create or update. Persists, broadcasts, and opens/focuses the dev-workspace window. Strips any `notes` from the body. |
| `GET` | `/api/file-city/trail/library?repositoryPath=<abs>` | List saved entries for the given repo (plus repo-agnostic ones); `repositoryPath` optional. |
| `GET` | `/api/file-city/trail/:id` | Load a saved payload by id. Returns the payload with `notes` inline. |
| `POST` | `/api/file-city/trail/activate` | Body `{ "id": "<id>" }`. Loads the saved entry, broadcasts `PAYLOAD_SET`, and opens/focuses the window. |
| `DELETE` | `/api/file-city/trail/:id` | Permanently delete the saved entry. |

When updating an existing trail, prefer re-POSTing with the same `id` over deleting and re-creating — `notes` already on the entry are preserved across the update.

## User notes

`TrailPayload.notes` carries user-authored snippet annotations and markdown comments. They are **renderer-authored only**:

- HTTP `POST` bodies have `notes` stripped during validation.
- `GET /api/file-city/trail/:id` returns notes inline.
- Re-POSTing a payload by `id` keeps existing notes.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `400` with validation errors | See the error table above. |
| `broadcastTo: 0` | No renderer is listening. App may still be starting up. |
| Trail renders, no highlights | `sourcePath` values don't match buildings — check they're repo-relative and the panel is open on the right repo. |
| `windowOpened: 'none'` despite `repositoryPath` being set | Repo isn't registered in Alexandria. Tell the user to add it from the launcher. |
| Snippet shows wrong lines | `startLine`/`endLine` are 1-based, inclusive. Off-by-one usually means you typed them 0-based. |
| Trail feels like an investigation despite the `purpose` field | Re-read the Quality bar. The field is metadata; the *content* is what makes a trail informative. Tighten markers and rewrite the summary to state the answer. |

## Reference

- Sister skill for the exploratory version (lay markers as you figure things out): `author-investigation-trail`
- Forking an existing investigation into an informative trail with a `derivedFrom` link: `convert-investigation`
- Publishing a finished informative trail to web-ade: from the Principal app UI
- Local-only authoring + viewer loop (no electron-app required): `author-local-investigation-trail`
- PR diff walkthroughs: `file-city-trail-review`
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
