---
name: topic-context
description: Read the topic an agent was briefed on and keep its description current as work progresses — list/search local topics, fetch a topic + its trails from the local Principal MCP Bridge, append discovered context, or replace one ## / ### section of the description in place (e.g. update a status block). Use when you were handed a "Fetch http://localhost:3044/api/topics/<id> to begin working on topic …" brief, or when the user says "update the topic", "mark this section done on the topic", "leave context on the topic", "fix the status on the topic", "what's this topic about", "what topics are there", "find the topic about X", or "open the topic" / "open it in the app" (activate it as a tab in the running app). Topics can be listed/filtered via GET /api/topics?q=…; the description is editable over the bridge via append (grow) and section upsert (replace-in-place); a topic can be opened in the app via POST /api/topics/:id/activate. NOT for creating a topic — use create-topic (local-bridge create). NOT for browsing topics a user published — use discover-trails. NOT for editing a topic's title or trail list — this skill is description-only.
---

# Topic Context

A **topic** is a curated bundle of trails on one subject, with a markdown
`description` that doubles as the working brief for agents pointed at it.
When you're briefed on a topic (`Fetch http://localhost:3044/api/topics/<id>
to begin working on topic "…"`), this skill is how you (a) read the brief and
its trails, and (b) write back what you learned so the next reader — human or
agent — sees current context instead of a stale one.

This is the **local-bridge** topic skill. It talks to the running
electron-app's Principal MCP Bridge, not web-ade. It's the read-and-keep-current
sibling to `create-topic` (local-bridge creation) and `discover-trails`
(web-only browsing).

## When to fire

Fire on phrases / situations like:

- You were handed a brief: `Fetch http://localhost:3044/api/topics/<id> to
  begin working on topic "…"`. Reading it is the first move; updating it as
  you finish is the last.
- "what topics are there?" / "find the topic about X" / "which topic was I
  working on?" — list/search them with `GET /api/topics` (see "Finding a
  topic").
- "update the topic" / "update the topic description"
- "mark the status section done on the topic" / "fix the status on the topic"
- "leave context on the topic for the next person"
- "what's this topic about?" / "summarize the topic I'm working on"
- "record what we found on the topic"
- "open the topic" / "open it in the app" / "open it in the editor" — activate
  it as a tab in the running app with `POST /api/topics/:id/activate` (see
  "Endpoints").
- "validate the topic's links" / "check the references on the topic" / "are the
  file links on the topic valid?" — `POST /api/topics/:id/validate-links` (see
  "Validate after editing").

Don't fire when the user wants to:

- **Create** a topic — use `create-topic` (creates locally over this same
  bridge via `POST /api/topics`).
- **Browse** topics a user has published — use `discover-trails`.
- **Edit the title or the trail list** — this skill only edits the
  **description** of the local topic over the bridge. Changing the title or
  adding/removing/reordering a topic's trails is out of scope here.
- **Author a trail** — use the `author-*` skills (publish from the app UI).

## Prerequisites

The electron-app must be running. The bridge listens on:

```
http://localhost:3044
```

The agent brief hardcodes `3044`. Confirm the bridge is up before any call:

```bash
curl -s http://localhost:3044/health   # → {"status":"ok",...,"port":3044}
```

If that fails, ask the user to launch the app rather than guessing other ports.

No GitHub token is needed: the bridge is local and unauthenticated. (That's
also why it's append/section only — see "Why no full replace?" below.)

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/topics` | List/search local topics (directory, no trails). Supports `?q=` substring filter. Returns `{ success, topics, count }`. |
| `GET` | `/api/topics/:id` | Read the topic + its trails. Returns `{ success, topic, trails }`. |
| `POST` | `/api/topics/:id/description/append` | Append a paragraph to the description. Body: `{ text }`. |
| `POST` | `/api/topics/:id/description/section` | Replace one `##`/`###` section in place, or append it if absent. Body: `{ heading, body, level? }`. |
| `POST` | `/api/topics/:id/activate` | Open the topic as a tab in the focused/main window (opens/focuses a window as needed). No body. Returns `{ success, delivered, windowOpened }`. |
| `POST` | `/api/topics/:id/validate-links` | Validate the file/doc references in the description. No body. Returns `{ success, summary, findings }`. Read-only. |

The read + write-description routes return the updated `{ success, topic, trails }`
(the section route also returns `action: "replaced" | "inserted"`). The
`activate` route doesn't edit anything — it just opens the topic in the app and
returns `{ success, delivered, windowOpened }`.

### The topic shape you get back

```ts
{
  topic: {
    id: string;
    title: string;
    trailIds: string[];      // ordered
    description: string;     // markdown — the working brief
    createdAt: string;
    updatedAt: string;
  },
  trails: Array<{
    id: string;
    href: string;            // /api/file-city/trail/<id> — fetch full payload here
    title?: string;
    summaryPreview?: string;
    purpose?: 'investigation' | 'informative' | ...;
    repoNames?: string[];
    missing?: true;          // id is in the topic but the trail no longer resolves locally
    // …marker/file/sign-off counts
  }>
}
```

`missing: true` means the topic remembers a trail id that's been deleted
locally — surface the dangling reference rather than pretending it's gone.

## Finding a topic (when you weren't handed an id)

If you have an id (from a brief, or from the user), skip to the next section.
When you *don't* — "what topics are there?", "find the topic about X", "which
topic was I working on?" — list them over the bridge. `GET /api/topics` is a
**directory**: it returns lightweight summaries (no description body, no trail
resolution), newest-updated first, and takes an optional `?q=` case-insensitive
substring filter over title + description.

```bash
curl -s http://localhost:3044/api/topics | jq '.topics[] | {id, title, trailCount, state, updatedAt}'
curl -s "http://localhost:3044/api/topics?q=storage" | jq '.topics[] | {id, title, descriptionPreview}'
```

Each entry:

```ts
{
  topics: Array<{
    id: string;
    href: string;               // /api/topics/<id> — GET this for the full topic + trails
    title: string;
    descriptionPreview: string; // first ~200 chars of the description, single-lined
    trailCount: number;
    state?: string;             // topic status, e.g. 'working'
    createdAt: string;
    updatedAt: string;          // entries sorted by this, descending
  }>,
  count: number
}
```

Pick the id, then GET `/api/topics/:id` for the real brief — the list never
carries the full description or the trails. This is local-bridge discovery; for
topics a user *published* to web-ade, use `discover-trails` instead.

## Reading the brief (always do this first)

```bash
curl -s http://localhost:3044/api/topics/$TOPIC_ID | jq
```

Read the `description` as your brief, and skim the `trails` list to see what's
already mapped. To go deeper on a trail, fetch its `href`:

```bash
curl -s "http://localhost:3044$(curl -s http://localhost:3044/api/topics/$TOPIC_ID | jq -r '.trails[0].href')" | jq
```

Continue your task with that context in mind.

## Writing back — two tools, different jobs

The description is markdown, almost always organized under `##`/`###`
headings (e.g. `## Status`, `## Notes`, `### To do`). You have two ways to
change it. **Pick by intent:**

### Append — for *new* discovered context

When you found a fact worth leaving and there's no section it belongs in (or
you're just adding a note), append a paragraph. It never touches existing
content.

```bash
curl -s -X POST http://localhost:3044/api/topics/$TOPIC_ID/description/append \
  -H 'content-type: application/json' \
  -d '{"text":"The token refresh lives in [AuthService.refresh](pkg:github/owner/repo#src/main/auth/AuthService.ts)."}'
```

A blank-line separator is inserted between the old body and your text.

### Section upsert — for *status* and anything that supersedes prior text

When you're updating something that already has a heading — a status block, a
"to do" list, a findings section — **replace the section in place** instead of
appending a contradictory copy. This is what keeps a topic truthful over many
sessions instead of stacking three different "## Status" blocks.

The loop is **GET → read the exact heading → upsert**:

1. `GET` the topic and find the section you mean. Copy its heading text
   **exactly** as written — matching is exact (only trailing whitespace is
   ignored). `## Status update` and `## Status — paste-to-open shipped` are
   *different* headings.
2. POST the new body under that heading:

```bash
curl -s -X POST http://localhost:3044/api/topics/$TOPIC_ID/description/section \
  -H 'content-type: application/json' \
  -d '{
    "heading": "Status update",
    "body": "**Done** — shipped X and Y. Remaining: Z."
  }'
```

Behavior:

- **Exactly one section matches** → its body is replaced in place; position and
  the original heading line are preserved. `action: "replaced"`.
- **No section matches** → a new section is appended at the end, at `level`
  (default `2`, i.e. `##`; pass `level: 3` for `###`). `action: "inserted"`.
- **More than one section matches** → **409**, nothing is written. The doc has
  duplicate headings; resolve them first (see below).

`heading` may be passed with or without leading `#`s — `"Status"` and
`"## Status"` target the same section. The section you replace owns its nested
deeper headings (replacing `## Notes` replaces everything down to the next
`##`), so scope your `body` to match.

### Referencing files & docs

When your text points at a file, write a **purl-qualified link**, not a bare
path or a `github.com` URL — a topic spans repos, so a bare path is ambiguous
and won't resolve when someone clicks it:

```
[AuthService.refresh](pkg:github/owner/repo#src/main/auth/AuthService.ts)
```

Repo = the purl (`pkg:<type>/<owner>/<repo>`); file = the `#subpath`; pin with
`@<sha>` when the reference must stay valid as the repo moves. Only reference
repos the topic claims. A symbol or path mentioned in passing can stay inline
code. (Full conventions live in the `create-topic` skill.)

### Validate after editing

After any append/section write that touched file references, validate them:

```bash
curl -s -X POST http://localhost:3044/api/topics/$TOPIC_ID/validate-links | jq
```

Fix every `error` (a repo link that isn't purl-qualified, a malformed purl) and
every `finding` (`missing` file, `out-of-scope` repo) — apply the fix with a
section upsert, then re-validate — and weigh the `suggestion`s (path-like inline
code that should be a purl link). Read-only; it never edits the topic for you.

### Resolving a 409 (duplicate headings)

If section upsert returns
`heading "X" matches N sections; resolve the duplicates first`, the
description already has N identically-titled sections (a classic append-era
mess). Decide which one is canonical, then either:

- `append` a single corrected section and ask the user to delete the stale
  ones via the UI, or
- if you're confident, the cleanest fix is to make the headings distinct first
  (e.g. retitle the stale one) — but heading retitling isn't a bridge
  operation, so this usually means flagging it to the user.

Don't try to force it; the 409 is the guard doing its job.

## Author discipline

- **Status sections: upsert, don't append.** The whole point is one truthful
  status, not an append log of contradictions.
- **Match headings verbatim.** GET first, copy the exact string. A near-miss
  silently *inserts a new section* (`action: "inserted"`) instead of replacing
  — that's how duplicates are born.
- **Scope the body to the section.** Replacing `## Status` replaces its entire
  subtree down to the next `##`. Include the sub-bullets you mean to keep.
- **Be concrete and current.** Reference files/symbols (`TopicRegistryService.updateTopic`),
  state what's done vs. remaining, and convert "now/next" into something a
  later reader can act on.
- **Don't restate the trails.** The `trails` list already renders; the
  description is for the connective narrative and status.

## What this skill doesn't do

- It doesn't **create** topics (`create-topic`) or **browse**
  published ones (`discover-trails`).
- It doesn't edit the **title** or the **trail list** — its *write* surface is
  description-only (it can also **open** a topic in the app via `activate`, which
  changes nothing).
- It doesn't **fully replace** an arbitrary description blob — by design. You
  grow it (append) or edit it a section at a time (upsert). See below.

## Why no full replace? (the design)

The bridge is local and **unauthenticated** — any process on the machine can
hit it. A blunt "replace the whole description" route would let one agent
silently clobber another agent's (or the user's) accumulated context. So the
write surface is deliberately narrow: **append** only grows, and **section
upsert** changes one named section and refuses when the target is ambiguous.
Full-document edits stay a human action in the UI. Work within that grain.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `404 unknown topic id` | The id isn't in the local registry. It may be a remote-only topic — this skill is local-bridge only. |
| `400 text (non-empty string) is required` | `append` body missing/empty `text`. |
| `400 heading (non-empty string) is required` | `section` body missing/empty `heading`. |
| `400 body (non-empty string) is required` | `section` body missing/empty `body`. |
| `409 heading "X" matches N sections` | Duplicate headings — resolve before upserting (see above). |
| Upsert created a *new* section you meant to replace (`action:"inserted"`) | Heading didn't match exactly. GET, copy the verbatim heading, retry. |

## Reference

- HTTP routes: `electron-app/src/main/topics/topicRoutes.ts`
- Registry + sync (write-through to web-ade when published): `electron-app/src/main/stores/TopicRegistryService.ts`
- Section engine: `upsertSection` / `splitSections` in `@principal-ade/markdown-utils` (≥ 0.3.1)
- Reference validation: `electron-app/src/main/topics/validateTopicLinks.ts` (`POST /api/topics/:id/validate-links`)
- The brief that points agents here: `electron-app/src/renderer/components/Titlebar/BriefAgentButton.tsx`
- Sister skills: `create-topic` (local-bridge create), `discover-trails` (web read), `document-notes` (local-bridge notes)
