---
name: author-informative-trail
description: Author a fresh informative trail — the durable, canonical version of a flow or insight — and POST it to the local Principal MCP Bridge so it lands in the running electron-app's File City panel. Informative trails state what is true about the code now; they are not a record of how the answer was found. Use when the user says "make an informative trail", "author a canonical trail", "lay a durable trail through X", "write a sign-off trail for this flow", or invokes /author-informative-trail. NOT for forking an existing investigation — use promote-investigation. NOT for runtime exploration trails or PR diff walkthroughs — use file-city-trail or file-city-trail-review.
---

# Author Informative Trail

Author a fresh **informative** trail from scratch — the durable, canonical version of a flow, behavior, or invariant. Informative trails are the version that gets shared and signed off on. They state *what is true* about the code; they are not a log of how the answer was discovered.

This skill is for **authoring from scratch**. If you already have an investigation trail and want to mint its canonical sibling, use `promote-investigation` instead — that path forks + links, this path doesn't.

The payload shape and POST route are the same as `file-city-trail` — what changes is the discipline: tighter markers, a summary that states the answer, and `purpose: 'informative'` on the payload.

## When to fire

Fire on phrases like:

- "make an informative trail for this flow"
- "author a canonical trail through the auth path"
- "write a durable trail for this lifecycle"
- "lay a sign-off trail for X"
- "informative trail of how Y works"
- explicit `/author-informative-trail` invocation

Don't fire when:

- The user wants to **explore / investigate** a flow as they figure it out — that's `file-city-trail` (investigation flavor) or `local-trails`.
- The user has an existing investigation trail and wants to **promote** it — that's `promote-investigation`.
- The user wants a **PR diff walkthrough** — that's `file-city-trail-review`.
- The user wants to **publish to web-ade** as a shareable link — that's `publish-trail`. (Authoring locally first and publishing later is a fine workflow; this skill handles the authoring half.)

## Prerequisites

The electron-app must be running. Endpoints live on the **production** Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, ask the user to launch the app rather than guessing other ports.

The File City panel must be open on the repo whose paths your `sourcePath` values reference; otherwise the trail renders without building highlights (no error, just no cyan fill).

## Inputs

The user supplies the **subject** of the trail — the flow, behavior, or invariant the trail will describe ("how the titlebar selection opens the project-info tab", "the WorkOS callback flow", "what happens when a dashboard metric refreshes"). Everything else you resolve from the codebase.

If the user hands you an investigation trail id and says "make this informative", stop and route to `promote-investigation` — that's the fork+link path, and bypassing it loses the `derivedFrom` link.

## Workflow

### 1. Trace the flow in the codebase

Before writing a payload, get the steps straight by reading source. A good informative trail has:

- A clear **entry point** — the place a reader would start if you handed them a whiteboard marker.
- A small set of **lanes** (2–6) that bundle related markers by participant or subsystem.
- An **ordered chain** that walks the reader from entry → answer without detours.

Do the trace first. If you can't name the lanes and the order without looking at code, you're not ready to author.

### 2. Identify the headline

The informative trail has a *headline* — the load-bearing fact the reader walks away with. The trail's title should be a statement of that fact (not a question, not an exploration). The summary should restate it concisely. The first marker the reader hits should either *be* the headline or walk them to it without detours.

If you can't articulate the headline in one sentence, you don't have an informative trail yet — you have an investigation. Either complete the investigation first, or use `file-city-trail` to author the investigative version.

### 3. Build the payload

Build a `TrailPayload`. The full schema lives in `file-city-trail`; the informative-specific guidance:

- `id` — fresh, unique. Generate with `crypto.randomUUID()` or a stable kebab id like `<headline-slug>-informative`.
- `title` — **a statement of what's true**, not an exploration. Good: *"Titlebar selection opens project-info tab via repository-selected event."* Bad: *"Looking into titlebar selection behavior"* or *"Titlebar selection"*. The title is the headline.
- `summary` — concise statement of the durable insight, not the path to it. **3 sentences max.** Drop "I was investigating…", "we figured out that…", "it turns out…" phrasing. Follow `file-city-trail`'s "Formatting rules" section in full: paragraph breaks between sentences (`\n\n` in JSON, not single `\n`), optional bolded lede labels (`**Default.**`, `**On click.**`, `**Trigger.**`), inline-code every real identifier, at most one emphasis-bold term in the body, no headers/lists/links/code blocks, active-voice present tense, name real identifiers (not paraphrases — write `selectedTrailId`, not "the current selection"), no first person, no hedging.
- `purpose: 'informative'` — set it explicitly. Marks the trail as durable / canonical for any tooling that distinguishes the two flavors.
- `markers` — tight. 5–8 is typical; 12 is already long for an informative trail. Each marker is one decision, one branch, one handler — not "the area around this region." **Do not emit `kind: 'subject'` on any marker** — that's an investigation-only concept.
- `views` — ship `[{ kind: 'sequence', markers, edges, layout? }]`. See `file-city-trail` for the view-ref schema (`markerId`, `name`, `participant`, edges).
- `authoredAt: { sha, ref }` — capture the producer-machine commit you authored against. Recommended for single-repo trails.
- `repos[]` — only for multi-repo trails; otherwise omit and use `authoredAt`.
- `createdAt` / `updatedAt` — ISO 8601 now; the route will fill if you omit.
- `notes` — never emit. Notes are renderer-authored only and stripped by the route.
- `share` — leave undefined. Sharing is a separate user action via the Share button.

Match the marker / view / snippet schemas documented in `file-city-trail` — every field has the same meaning here.

### 4. POST it

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

### 5. Handle the response — loop on validation errors

Success looks like:

```json
{ "success": true, "id": "<id>", "broadcastTo": 1, "evictedIds": [], "windowOpened": "created" }
```

Report the `id` and `windowOpened` to the user and stop.

- `windowOpened: 'focused'` — existing dev-workspace window for the repo was brought to front.
- `windowOpened: 'created'` — new window opened with this trail auto-loaded via `?openTrailId=`.
- `windowOpened: 'none'` — repo isn't registered in Alexandria. Tell the user to add it from the launcher, then activate the saved trail from the Trails sidebar.

Failure: read `error`, fix the field in your payload, POST again. Common cases (mirrors `file-city-trail` and `promote-investigation`):

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
- **Has snippets on the right line.** `focusLine` points at the *call*, *decision*, or *emission* the marker is about — not the function signature or surrounding boilerplate. Windows are tight (5–25 lines) so the focal line isn't buried.
- **Has descriptions that answer "why this exists" or "what's surprising here"** — the cookie that's checked, the retry that's silent, the race that almost bit you, the branch most readers don't notice. 2–6 sentences. Active voice, present tense. Same formatting discipline as the summary (inline-code identifiers, no headers/lists, no first person).
- **Reads in the order a reader should learn it.** The first marker is where you'd start explaining the flow on a whiteboard. Edge labels (`"then"`, `"on success"`, `"on 401"`, `"async"`) carry the causal structure between markers.

## Checklist before POSTing

Run through this list. If any answer is "no" or "not sure," fix it before posting:

- [ ] `title` reads as a statement of what's true (not a question, not "Looking into…").
- [ ] `summary` is ≤3 sentences, paragraph-broken, and states the answer (not the journey).
- [ ] `purpose: 'informative'` is set explicitly.
- [ ] Every marker is on the path from entry to answer; no dead ends.
- [ ] No marker carries `kind: 'subject'`.
- [ ] Marker descriptions are factual — none of the banned editorial phrasings ("the magic happens", "critically", "TL;DR", "the key insight", "importantly", "crucially", "note that").
- [ ] Identifiers in summary and descriptions are inline-coded and real (not paraphrases).
- [ ] Lanes are stable and named by participant/subsystem; no `step1`/`handler`/`do-thing` placeholders.
- [ ] `sourcePath` values are repo-relative; `repositoryPath` (top-level, beside the payload) is the absolute path on this machine.
- [ ] `authoredAt: { sha, ref }` is set for single-repo trails; or `repos[]` + per-marker `repo` for multi-repo.

## What you're not doing

- **Not investigating.** Informative trails describe what's true now. If you're discovering it as you go, switch to `file-city-trail` (investigation flavor) and promote later with `promote-investigation`.
- **Not narrating the discovery.** No *"I noticed…"*, no *"after tracing through…"*, no *"the bug was…"*. The trail does not reference itself or its author.
- **Not capturing every related file.** Density matters more than coverage. A 6-marker trail that hits the load-bearing decisions beats a 14-marker trail that includes every file the flow touches.
- **Not setting `kind: 'subject'` on any marker.** That's an investigation-only concept for marking the destination during exploration. Informative trails lead with the answer instead.
- **Not publishing.** This skill writes to the local library and broadcasts to the running app. To share publicly, follow up with `publish-trail` once the trail is good.
- **Not emitting `notes`.** Notes are renderer-authored only and the route strips them on POST.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `400` with validation errors | See the error table above. Pre-check against the validation rules in `file-city-trail`'s "Validation rules to obey" section. |
| `broadcastTo: 0` | No renderer is listening. App may still be starting up. |
| Trail renders, no highlights | `sourcePath` values don't match buildings — check they're repo-relative and the panel is open on the right repo. |
| `windowOpened: 'none'` despite `repositoryPath` being set | Repo isn't registered in Alexandria. Tell the user to add it from the launcher. |
| Snippet shows wrong lines | `startLine`/`endLine` are 1-based, inclusive. Off-by-one usually means you typed them 0-based. |
| Trail feels like an investigation despite the `purpose` field | Re-read the Quality bar. The field is metadata; the *content* is what makes a trail informative. Tighten markers and rewrite the summary to state the answer. |

## Reference

- Schema, full marker/view/snippet field guidance, validation rules, lane ordering, multi-repo shape: `file-city-trail`
- Forking an existing investigation into an informative trail with a `derivedFrom` link: `promote-investigation`
- Publishing a finished informative trail to web-ade: `publish-trail`
- Local-only authoring loop (no electron-app required): `local-trails`
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
