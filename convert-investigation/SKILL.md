---
name: convert-investigation
description: Convert an existing investigation trail into a new informative trail by forking it. Read the source investigation, identify the subject marker (the answer it points at), author a refined informative payload, and POST it to the local Principal MCP Bridge's fork route. The route stamps a derivedFrom link so the new informative trail is associated with the original investigation. Use when the user says "convert this trail", "make this an informative trail", "turn this investigation into a canonical trail", or invokes /convert-investigation. NOT for authoring a trail from scratch — use author-informative-trail (canonical from scratch) or author-investigation-trail (exploratory).
---

# Convert Investigation

Take an existing **investigation** trail and produce a sibling **informative** trail derived from it. The investigation is preserved as a record of how the answer was found; the informative trail is the durable, canonical version that gets shared and signed off on.

This skill is for **fork + link**. The original investigation is never mutated. A new informative trail is created with a fresh id; the host's index records `derivedFrom: <investigation-id>` so the link travels with the local library.

## When to fire

Fire on phrases like:

- "convert this trail to informative"
- "make a canonical version of this investigation"
- "fork this to an informative trail"
- "turn this investigation into a durable trail"
- explicit `/convert-investigation` invocation

Don't fire when:

- The user wants to **author** a trail from scratch — that's `author-informative-trail` (canonical) or `author-investigation-trail` (exploratory). For publishing or local-only authoring see `publish-trail` / `local-trails`.
- The user wants to **share** a trail to web-ade — that's the renderer-driven Share flow.
- The user wants to **mutate** the investigation in place — conversion is fork-only.

## Prerequisites

The electron-app must be running. Endpoints live on the **production** Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before posting. If it's not up, ask the user to launch the app rather than guessing other ports.

## Inputs

The user supplies (or the caller passes) a **sourceId** — the id of the investigation trail to convert. Resolve everything else from the source.

## Workflow

### 1. Fetch the source investigation

```bash
curl -s http://localhost:3044/api/file-city/trail/<sourceId>
```

This returns the full payload including `notes` (which are renderer-only and won't be in the new payload — you read them for context, but don't copy them across).

If you get a 404, stop and tell the user the source trail doesn't exist locally. Don't invent a payload.

### 2. Identify the subject

The **subject** is the marker the trail directs the reader's focus toward — the cause, the bug location, the answer. It's the destination, not a stop along the way.

If marker descriptions are clear, you can identify the subject from the payload alone. If they're ambiguous, use the read/grep tools to inspect the actual source code at each marker's `sourcePath` + line range to disambiguate. The working directory is set to the trail's repository.

The subject concept is **investigation-only**. The informative trail's markers will **not** carry `kind: 'subject'` — the server strips it defensively, but don't emit it.

### 3. Author the informative payload

Build a new `TrailPayload`:

- `id` — fresh, unique. Generate with `crypto.randomUUID()` or a stable kebab id like `<source-title-slug>-informative`. **Must differ from `sourceId`.**
- `title` — refined. Investigations often have exploratory titles ("looking into…"); informative trails should read as a statement of what's true ("Titlebar selection opens project-info tab via repository-selected event").
- `summary` — refined. Concise statement of the durable insight, not the path to it. **3 sentences max** (overflow belongs in marker descriptions, not the summary). Drop the "I was confused about…" phrasing. Follow `author-informative-trail`'s "Write the summary" / "Formatting rules" section in full: paragraph breaks between sentences, optional bolded lede labels (`**Default.**`, `**On click.**`, etc.), inline-code every real identifier, at most one emphasis-bold term in the body, no headers/lists/links/code blocks, active-voice present tense, name real identifiers (not paraphrases), no first person or hedging.
- `purpose: 'informative'` — required. The route forces it, but emit it explicitly for clarity.
- `markers` — start from the source's markers but feel free to: drop dead-end ones, tighten descriptions, reorder so the subject marker reads as the headline. **Do not emit `kind: 'subject'` on any marker.**
- `views` — keep the same shape as the source (`[{ kind: 'sequence', markers, edges, layout? }]`). Update `markerId` references if you changed marker ids.
- `authoredAt` / `repos[]` — carry over from the source.
- `createdAt` / `updatedAt` — ISO 8601 now; the route will fill if you omit.
- `share` — leave undefined. Sharing is a separate user action.
- `notes` — leave undefined. Notes are renderer-only.

Match the schema documented in `author-informative-trail` / `author-investigation-trail` — every field has the same meaning here.

### 4. POST it to the fork route

```bash
curl -s -X POST http://localhost:3044/api/file-city/trail/fork-informative \
  -H 'content-type: application/json' \
  -d @body.json
```

Where `body.json` is:

```json
{
  "sourceId": "<the investigation's id>",
  "repositoryPath": "<absolute filesystem path on this machine, same as the source>",
  "payload": {
    "id": "<new id>",
    "title": "…",
    "summary": "…",
    "purpose": "informative",
    "markers": [ … ],
    "views": [ … ],
    "createdAt": "…",
    "updatedAt": "…"
  }
}
```

`repositoryPath` is the **absolute filesystem path** on the producer machine (not in the portable payload — it's a top-level field on the body). Carry it over from the source's library entry if you can; if you don't have it, omit and the route won't auto-open a window.

### 5. Handle the response — loop on validation errors

Success looks like:

```json
{
  "success": true,
  "id": "<new id>",
  "derivedFrom": "<sourceId>",
  "broadcastTo": 1,
  "evictedIds": [],
  "windowOpened": "focused"
}
```

Report the new `id` to the user and stop.

Failure looks like:

```json
{
  "success": false,
  "error": "<human-readable message>"
}
```

Read `error`, fix the offending field in your payload, and POST again. Common cases:

| `error` substring | Fix |
|---|---|
| `markers must be a non-empty array` | You stripped too much. Keep at least one marker. |
| `every marker must have a string id` | A marker is missing `id`. |
| `duplicate marker id` | You reused an id when remixing. Make them unique. |
| `unknown markerId` | A `views[0].markers[].markerId` doesn't match any `payload.markers[].id`. Usually after renaming. |
| `marker X: snippet requires a sourcePath` | A marker has `snippet` but no `sourcePath`. Either drop the snippet or add the path. |
| `id must be a non-empty string` | You forgot to set `payload.id`. |
| `title must be a non-empty string` | You forgot `payload.title`. |
| `fork must use a new id distinct from sourceId` | You reused the source's id. Generate a new one. |
| `source trail "<id>" not found` | You passed the wrong `sourceId`. Re-check. |

**Stop after 5 attempts** if you keep hitting validation errors. Report the last error to the user — there's likely something they need to clarify.

## Quality bar

A good conversion isn't just a copy of the investigation with `purpose: 'informative'`. The informative trail should:

- **Lead with the subject.** The first marker the reader encounters should be the destination, or the markers should walk them to it without detours.
- **Drop investigation cruft.** Dead ends, "let me check…" descriptions, debug markers, things the original author backtracked from — all go.
- **State the answer in the summary.** Not "I was investigating X" — "X happens because Y, surfaced at Z."
- **Be tight.** If the investigation was 12 markers, the informative might be 5–8. Density matters.
- **Strip editorial flourishes from descriptions.** Investigations often carry author commentary — *"The whole feature."*, *"This is where the magic happens."*, *"Everything hinges on this."*, *"Critically important step."*, *"TL;DR: …"*, *"The key insight is …"*, *"Note that …"*, *"Importantly, …"*. Conversion is the moment to delete them. Rewrite the description as a factual statement about what the code does; if the marker is load-bearing, demonstrate that through specific detail (the branch taken, the value checked, the side-effect emitted) — never by telling the reader it matters. Show the cookie, don't announce that there is a cookie worth showing. (Mirrors `author-informative-trail`'s "No editorializing" quality-bar rule — apply it on the way through even when the source violated it.)

## What doesn't change

- `TrailMarker`, `TrailView`, `TrailNote` schemas — same as `author-informative-trail` / `author-investigation-trail`.
- Snippet validation — same line/file rules.
- Repo identity discipline — `sourcePath` repo-relative; `repositoryPath` host-private absolute.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `404` on the source fetch | `sourceId` is wrong or the trail was deleted. |
| Repeated `unknown markerId` errors | You're editing markers in `payload.markers` but not updating the corresponding `views[0].markers[].markerId` references. |
| Agent stops without POSTing | You read the source, decided "no clear subject" — say so to the user instead of doing nothing. |

## Reference

- Sister skills for authoring from scratch: `author-informative-trail` (canonical), `author-investigation-trail` (exploratory)
- Sister skill for publishing to web-ade: `publish-trail`
- Upgrade context: `web-ade/docs/file-city-trail-panel-0.5.81-upgrade.md`
- Schema source: `industry-themed-file-city-panels/src/types/Trail.ts`
