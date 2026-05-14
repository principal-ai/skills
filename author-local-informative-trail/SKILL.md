---
name: author-local-informative-trail
description: Author a fresh informative trail — the durable, canonical version of a flow or insight — for the user's own codebase and open it in the standalone trail viewer locally. Informative trails state what is true about the code now; they are not a record of how the answer was found. Writes a TrailPayload JSON to disk and runs `principal-ai trail view --file <path>` — no web-ade publish, no electron-app, no GitHub token. Use when the user says "make a local informative trail", "author a canonical trail locally", "lay a durable local trail through X", "write a local sign-off trail for this flow", or invokes /author-local-informative-trail. NOT for the exploratory version you lay as you figure something out — use author-local-investigation-trail. NOT for publishing to web-ade — use publish-trail. NOT for the electron-app File City panel — use author-informative-trail.
---

# Author Local Informative Trail

Author a fresh **informative** trail from scratch — the durable, canonical version of a flow, behavior, or invariant — for the user's own codebase and open it in the standalone `principal-ai/trail-viewer` locally. Informative trails are the version that gets shared and signed off on. They state *what is true* about the code; they are not a log of how the answer was discovered.

This skill is the **local-viewer sibling** of `author-informative-trail`. Same authoring discipline, same payload shape — different destination. Instead of POSTing to the electron-app's MCP Bridge, this skill writes the payload to a JSON file and opens it in the standalone trail-viewer via the `principal-ai` CLI.

For the **exploratory** flavor (lay markers as you figure something out, allowed to wander, allowed to mark a subject), use `author-local-investigation-trail` and tighten later. For the electron-app File City panel destination, use `author-informative-trail`.

## When to fire

Fire on phrases like:

- "make a local informative trail for this flow"
- "author a canonical trail locally through the auth path"
- "write a durable local trail for this lifecycle"
- "lay a sign-off trail for X locally"
- "informative trail of how Y works, view it locally"
- explicit `/author-local-informative-trail` invocation

Don't fire when:

- The user wants to **explore / investigate** locally as they figure it out — that's `author-local-investigation-trail`.
- The user wants the trail to land in the **electron-app's File City panel** — that's `author-informative-trail`.
- The user wants to **publish to web-ade** as a shareable link — that's `publish-trail`. Authoring locally first and publishing later is fine; this skill handles the authoring + local-viewing half.
- The user wants a **PR diff walkthrough** — that's `file-city-trail-review`.

## Prerequisites

The CLI runs via npx — nothing has to be installed up front:

```bash
npx -y @principal-ai/principal-view-cli@latest trail view --help
```

Platform support: the bundled trail-viewer ships prebuilt for **macOS arm64** today. Other platforms can pass `--viewer-dir <path>` to a source checkout if they have one; otherwise the CLI exits 2 with a clear message.

What the user does *not* need for this flow:

- The electron-app running. The standalone viewer is its own process.
- A GitHub token. Local mode reads slices straight from the working tree.
- An `<owner>/<repo>` argument. The trail isn't going anywhere; identity doesn't matter.
- Network access. Fully offline.

## Inputs

The user supplies the **subject** of the trail — the flow, behavior, or invariant the trail will describe ("how the titlebar selection opens the project-info tab", "the WorkOS callback flow", "what happens when a dashboard metric refreshes"). Everything else you resolve from the codebase.

If the user already has an exploratory trail file authored via `author-local-investigation-trail` and wants its canonical sibling, copy the markers you want to keep, strip the subject marker (`kind: 'subject'`), and rewrite the title/summary as statements of what's true. The local pair doesn't have a `convert-investigation` equivalent today — the conversion is mechanical: drop the subject, tighten markers to the answer path, restate the title.

## The trail schema in 60 seconds

Trails split content from layout:

- **Markers** carry the *content* — `id`, `sourcePath`, `snippet`, `description`. View-agnostic.
- **Views** carry the *structure* — for v1 always ship `views: [{ kind: 'sequence', markers: [...], edges: [...] }]`. The sequence view block names which markers participate and how they're laid out (lanes, edges).

Why split? The trail medium is designed to grow sibling renderers (linear, graph, tree) without changing marker content. v1 only registers the sequence renderer; ship a single sequence view.

Repos: trails carry portable identity, not filesystem paths. For the common single-repo case, ship `authoredAt: { sha, ref }` and omit `repos[]`. The producer machine's filesystem path is passed to the viewer as the `--repo-root` flag at launch — separate from the payload — so slice fetches resolve against the right working tree.

## Authoring workflow

### 1. Trace the flow in the codebase

Before writing a payload, get the steps straight by reading source. A good informative trail has:

- A clear **entry point** — the place a reader would start if you handed them a whiteboard marker.
- A small set of **lanes** (2–6) that bundle related markers by participant or subsystem.
- An **ordered chain** that walks the reader from entry → answer without detours.

Do the trace first. If you can't name the lanes and the order without looking at code, you're not ready to author — switch to `author-local-investigation-trail`, figure it out, then tighten.

### 2. Identify the headline

The informative trail has a *headline* — the load-bearing fact the reader walks away with. The trail's title should be a statement of that fact (not a question, not an exploration). The summary should restate it concisely. The first marker the reader hits should either *be* the headline or walk them to it without detours.

If you can't articulate the headline in one sentence, you don't have an informative trail yet — you have an investigation. Either complete the investigation first, or use `author-local-investigation-trail` to author the investigative version.

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
  // notes?: TrailNote[]          // renderer-authored only; see "User notes"
  createdAt: string;              // REQUIRED — ISO 8601
  updatedAt: string;              // REQUIRED — ISO 8601
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
- `createdAt` / `updatedAt` — ISO 8601 now.
- `notes` — never author by hand. Notes are renderer-authored only and are added to the file via the viewer's note composer.

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

### 5. Write the payload to a local file

Pick a stable location the user can re-open later. Conventions that work well:

- `<repo-root>/.trails/<flow-name>.json` — keeps the trail next to the code it documents (add to `.gitignore` if it's WIP).
- `~/.principal/local/<flow-name>.json` — user-scoped, survives across repos.
- `/tmp/<flow-name>-trail.json` — throwaway / iteration.

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
  "updatedAt": "2026-05-13T12:00:00.000Z"
}
```

For `authoredAt.sha`, use the current commit if you want pinned provenance:

```bash
git rev-parse HEAD
```

For local-only workflows, the sha is informational — slices come from the working tree, not from GitHub. You can leave `authoredAt` off entirely if there's no surrounding context to record.

### 6. Open in the viewer

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

### 7. Iterate

The viewer is single-instance and dedupes by `trailFilePath`:

- Re-running the same `--file <path>` focuses the existing tab (does **not** open a duplicate).
- Editing the JSON and re-running picks up changes — but **only if the tab was closed first** (Cmd+W). Open tabs hold the parsed payload in memory; close the tab and re-open to force a re-load.
- Different `--file` paths each get their own tab.
- The Library tab lists every cached trail under `~/.principal/trails/...` and routes clicks to local mode automatically when the slug decodes to a working tree on disk.

To force-quit and start fresh: Cmd+Q the viewer window; the next CLI invocation spawns a new instance.

### 8. (Optional) Publish

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

## Quality bar

This is the heart of the skill. The mechanics above are the same as any trail JSON; what makes a trail *informative* is the discipline below. A good informative trail:

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

## Checklist before opening the viewer

Run through this list. If any answer is "no" or "not sure," fix it before opening:

- [ ] `title` reads as a statement of what's true (not a question, not "Looking into…").
- [ ] `summary` is ≤3 sentences, paragraph-broken, and states the answer (not the journey).
- [ ] `purpose: 'informative'` is set explicitly.
- [ ] Every marker is on the path from entry to answer; no dead ends.
- [ ] No marker carries `kind: 'subject'`.
- [ ] Marker descriptions are factual — none of the banned editorial phrasings.
- [ ] Identifiers in summary and descriptions are inline-coded and real (not paraphrases).
- [ ] Lanes are stable and named by participant/subsystem; no `step1`/`handler`/`do-thing` placeholders.
- [ ] `sourcePath` values are repo-relative; pass `--repo-root` matching the working tree they were authored against.
- [ ] `authoredAt: { sha, ref }` is set for single-repo trails; or `repos[]` + per-marker `repo` for multi-repo.

## What you're not doing

- **Not investigating.** Informative trails describe what's true now. If you're discovering it as you go, switch to `author-local-investigation-trail`.
- **Not narrating the discovery.** No *"I noticed…"*, no *"after tracing through…"*, no *"the bug was…"*. The trail does not reference itself or its author.
- **Not capturing every related file.** Density matters more than coverage. A 6-marker trail that hits the load-bearing decisions beats a 14-marker trail that includes every file the flow touches.
- **Not setting `kind: 'subject'` on any marker.** That's an investigation-only concept for marking the destination during exploration. Informative trails lead with the answer instead.
- **Not publishing.** This skill writes a local JSON and opens it in the standalone viewer. To share publicly, follow up with `publish-trail` once the trail is good.
- **Not POSTing to the electron-app.** For the File City panel destination, use `author-informative-trail`.
- **Not authoring `notes`.** Notes are renderer-authored only — added inside the viewer via its note composer, persisted back into the JSON by the bun host.

## Path discipline

- All `sourcePath` values are **repo-relative** (e.g. `auth-server/src/routes/workos.ts`), not absolute, not prefixed with the repo name.
- `--repo-root` is the **absolute filesystem path** the viewer resolves slices against. Pass it explicitly when the user runs the CLI from a different cwd; otherwise the CLI defaults to `process.cwd()`.
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

Trail files live wherever you write them (see step 5). The viewer also reads its **cache library** at `~/.principal/trails/`:

- `~/.principal/trails/local/<path-slug>/<id>.json` — trails authored against a local working tree. The slug is `encodePathForPurl(repoRoot)` (slashes become dashes).
- `~/.principal/trails/by-id/<id>.json` — trails without repo identity.
- `~/.principal/trails/<purl-namespace>/<purl-name>/<id>.json` — hierarchical layout for trails with `repos[0].id` (github/gitlab/etc).

The library tab in the viewer walks this tree and shows every entry. To make a trail show up in the library, either author it directly into the cache layout above, or open it once with `principal-ai trail view --file …` and re-open from the library after.

## User notes

`TrailPayload.notes` carries user-authored snippet annotations and markdown comments. They are **renderer-authored only**:

- Don't hand-author `notes` in the JSON. The viewer's note composer is the entry point.
- The bun host writes mutations back into the JSON file in place: `createTrailNote` appends, `updateTrailNote` mutates by id + bumps `updatedAt`, `deleteTrailNote` filters out. The `{ entry, payload }` wrapper used by web-ade fetches is preserved on rewrite.
- Re-opening the trail (or closing + reopening the tab) re-reads `notes` from disk.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| Viewer opens with empty markers | `markers` array is empty or every marker is missing `id`. |
| Viewer renders but every marker errors on click | `--repo-root` doesn't match where `sourcePath` values were authored against. |
| `Path escapes repo root: <path>` in snippet drawer | A marker's `sourcePath` resolves outside `--repo-root` (e.g. `../other-repo/x.ts`). Trails are single-repo by design; for multi-repo, ship `repos: [...]` and `marker.repo`. |
| `Render error: undefined is not an object (evaluating 'trail2.views[0]')` | `views` array is missing or empty. v1 trails always need `views: [{ kind: 'sequence', ... }]`. |
| Library-tab click loads in remote mode and snippets fail with "no repo identity" | The decoded path slug doesn't point at a directory on disk. Open via `--file` with explicit `--repo-root`. |
| Viewer doesn't open on second invocation | Already running and the new trail handed off via socket — check the existing window for a new tab. |
| New tab not appearing for the same trail | Dedupe focused the existing tab. Close (Cmd+W) and reopen if you need a fresh load. |
| `principal-trail-viewer: no prebuilt bundle for <os>-<arch>` | Currently only macOS arm64 is shipped. Pass `--viewer-dir <path>` to a source checkout, or wait for the per-arch fan-out. |
| Trail feels like an investigation despite the `purpose` field | Re-read the Quality bar. The field is metadata; the *content* is what makes a trail informative. Tighten markers and rewrite the summary to state the answer. |

## Reference

- Sister skill for the exploratory version (lay markers as you figure things out): `author-local-investigation-trail`
- Sister skill for the electron-app File City panel destination: `author-informative-trail`
- Publishing a finished informative trail to web-ade: `publish-trail`
- PR diff walkthroughs: `file-city-trail-review`
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
- Viewer modes design: `principal-view-core-library/docs/TRAIL_VIEWER_MODES.md`
