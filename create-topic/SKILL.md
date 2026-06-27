---
name: create-topic
description: Create a topic in the running electron-app — a curated bundle of trails on one subject, with a markdown description that doubles as the working brief for agents pointed at it. Posts to the local Principal MCP Bridge (POST /api/topics), so the topic shows up in the app's Topics surface immediately. Use when the user says "create a topic", "make a topic for this work", "start a topic on X", "bundle these trails into a topic", or invokes /create-topic. The topic is local-only until the user publishes it from the app UI. NOT for authoring a single trail — use author-{investigation,informative}-trail. NOT for reading or updating a topic that already exists — use topic-context. NOT for browsing topics published on web-ade — use discover-trails.
---

# Create Topic

A **topic** is a curated bundle of trails on one subject, with a markdown
`description` that doubles as the working brief for agents pointed at it. This
skill creates one **locally**, in the running electron-app, by POSTing to its
Principal MCP Bridge. The new topic appears in the app's Topics surface right
away and can be linked to trails, briefed to agents, and later published to
web-ade from the app UI.

This is the **local-bridge** creation skill — the create sibling to
`topic-context` (read + keep-current over the same bridge). It does **not**
touch web-ade; publishing stays a human action in the app UI.

## When to fire

Fire on phrases like:

- "create a topic" / "make a topic for this work"
- "start a topic on <subject>"
- "bundle these trails into a topic"
- "make a topic so we can brief an agent on it"
- explicit `/create-topic` invocation

Don't fire when the user wants to:

- **Author a single trail** — use `author-investigation-trail` /
  `author-informative-trail`, then publish from the app UI.
- **Read or update an existing topic** — use `topic-context` (read the brief,
  append context, upsert a status section).
- **Browse topics published on web-ade** — use `discover-trails`.
- **Edit the title or trail list of an existing topic** — those route through
  the app UI / web-ade owner affordances.

## Prerequisites

The electron-app must be running. The bridge listens on:

```
http://localhost:3044
```

Confirm the bridge is up before any call:

```bash
curl -s http://localhost:3044/health   # → {"status":"ok",...,"port":3044}
```

If that fails, ask the user to launch the app rather than guessing other ports.

No GitHub token is needed: the bridge is local and unauthenticated. (Web-ade
publishing — which *does* need auth — happens later, from the app UI.)

## Endpoint

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/topics` | Create a local topic. Returns `201 { success, topic, trails }`. |

### Request body

```ts
{
  title: string;            // required, non-empty (trimmed)
  description?: string;     // markdown — the working brief
  trailIds?: string[];      // ordered local trail ids to bundle (default: [])
  visibility?: 'private' | 'sharable';  // intent only; default 'sharable'
}
```

The server fills in everything else — `id`, `createdAt`, `updatedAt`, and
`createdBy` are minted by the registry. `trailIds` are **local** trail ids (the
ids you get from `/api/file-city/trail/library`), not web-ade ids; an empty or
omitted list creates an empty topic you can add trails to later.

### Response

Same `{ topic, trails }` shape `topic-context` returns, so you can read it back
immediately:

```ts
{
  success: true,
  topic: {
    id: string;            // newly minted — capture this
    title: string;
    description: string;
    trailIds: string[];    // ordered
    createdAt: string;
    updatedAt: string;
  },
  trails: Array<{
    id: string;
    href: string;          // /api/file-city/trail/<id>
    title?: string;
    missing?: true;        // id was supplied but doesn't resolve locally
    // …summaryPreview, purpose, repoNames, counts
  }>
}
```

A `missing: true` entry means a `trailId` you passed isn't in the local trail
library — surface it rather than assuming the topic is fully wired.

## The flow

1. **Confirm the bridge is up** (`/health`) and pick the live port.

2. **Gather the trail ids** (optional). If bundling existing trails, list the
   local library and pick ids:

   ```bash
   curl -s http://localhost:3044/api/file-city/trail/library | jq '.entries[] | {id, title}'
   ```

   Order matters — the topic renders trails in the order you pass them.

3. **Write a title + description.** Title is the heading. Description is
   markdown and is the **working brief**: state the subject, the comparison or
   investigation axis, and what each trail contributes. See "Author discipline".

4. **Create the topic:**

   ```bash
   curl -s -X POST http://localhost:3044/api/topics \
     -H 'content-type: application/json' \
     -d '{
       "title": "Filesystem change detection across three CLIs",
       "description": "## Subject\nHow three agent CLIs pick up edits made during a long-running session.\n\n## Status\nMapping in progress.",
       "trailIds": ["4f2b…", "9d1a…"]
     }' | jq
   ```

5. **Capture the returned `topic.id`.** That id is how you (or a briefed agent)
   reach it afterward via `topic-context` (`GET /api/topics/<id>`, append,
   section upsert). Report it to the user.

6. **Validate the references** in the description you just wrote:

   ```bash
   curl -s -X POST http://localhost:3044/api/topics/<id>/validate-links | jq
   ```

   It returns a `summary` and a `findings[]` severity ladder. Act on it before
   handing off: fix every **error** (a repo link that isn't purl-qualified, or a
   malformed purl) and every **finding** (a file that doesn't exist — `missing`;
   a repo the topic doesn't claim — `out-of-scope`); fold in **suggestions**
   (path-like inline code that should be a purl link) where they help. Apply
   fixes with `topic-context`'s section-upsert, then re-validate. See
   "Referencing files & docs".

7. **Hand off / next steps.** To keep the description current as work proceeds,
   switch to `topic-context`. To publish to web-ade and get a shareable
   `app.principal-ade.com/topic/<id>` link, tell the user to publish from the
   app UI (publishing is owner-authenticated and intentionally not a bridge
   operation).

## Recipes

### Create an empty topic to brief an agent on

```bash
curl -s -X POST http://localhost:3044/api/topics \
  -H 'content-type: application/json' \
  -d '{"title":"Auth refresh investigation","description":"## Goal\nFigure out where token refresh lives and when it fires."}' \
  | jq '.topic.id'
```

### Create with bundled trails, in order

```bash
curl -s -X POST http://localhost:3044/api/topics \
  -H 'content-type: application/json' \
  -d '{
    "title": "Agent FS-change detection",
    "description": "Three CLIs, three strategies. Read in order.",
    "trailIds": ["4f2b…", "9d1a…", "7c3e…"]
  }' | jq
```

### Create a private (don't-auto-suggest-publish) topic

```bash
curl -s -X POST http://localhost:3044/api/topics \
  -H 'content-type: application/json' \
  -d '{"title":"Internal spike","visibility":"private"}' | jq
```

## Author discipline — what makes a good topic

A topic is a curation plus a brief, not a folder. The description should make
the **axis** explicit: what dimension are these trails varying along, and
what's the reader (human or next agent) supposed to notice?

Bad (just a list):
> Three trails about agent CLIs.

Good (axis stated):
> Three CLIs solve "detect filesystem changes during a long-running session"
> three different ways: inotify-driven invalidation in Trail 1, a 200 ms
> polling loop in Trail 2, OS-level FSEvents delegation in Trail 3. Read in
> order to see the responsiveness/CPU trade-off shift at each step.

- **Structure the description with `##`/`###` headings** (e.g. `## Subject`,
  `## Status`, `## Notes`). That's the shape `topic-context`'s section-upsert
  expects, so later status edits replace cleanly instead of stacking.
- **Order trails so the reader's mental model builds** across the list.
- **Don't restate the trails** — their titles/summaries already render. The
  description is for the connective narrative and status.

## Referencing files & docs

A topic spans multiple repos and has no single "current repo", so a bare path
(`src/auth/token.ts`) is ambiguous and a `github.com/...` URL is provider-locked.
**Reference a file with a purl-qualified link** so it resolves anywhere — for a
human clicking it in the app, or an agent fetching it:

```markdown
[token refresh](pkg:github/owner/repo#src/auth/token.ts)
```

- The repo is the purl (`pkg:<type>/<owner>/<repo>`); the file is the `#subpath`.
  Line ranges ride *outside* the purl (e.g. trailing `#Lx-Ly`), not inside it.
- **Pin to a commit** (`@<sha>` or `?commit=<sha>`) when the reference must stay
  valid as the repo moves — the validator's "not found on main" finding will
  tell you when this is needed.
- **Only reference repos the topic claims** (the repos its trails were authored
  in); a purl into any other repo comes back `out-of-scope`.
- Mentioning a symbol or a path in passing? Inline code (`` `AuthService.refresh` ``)
  is fine — the validator only *suggests* converting a path-shaped span. But if
  you mean "go open this file", make it a purl link.

The `validate-links` route (flow step 6) enforces this; write purl links from
the start so the report comes back clean.

## What this skill doesn't do

- It doesn't **author trails** — use the `author-*` skills, then add their ids
  to a topic (here, or via the app UI).
- It doesn't **read or update** an existing topic — use `topic-context`.
- It doesn't **publish to web-ade** — the created topic is local until the user
  publishes from the app UI. Publishing is owner-authenticated and stamps a
  `sharedUrl`; it deliberately isn't a bridge operation.
- It doesn't **edit the title or trail list** after creation over the bridge —
  the bridge create route sets them once; later membership/title changes go
  through the app UI.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `400 title (non-empty string) is required` | Missing/empty `title` in the body. |
| `400 trailIds must be an array of strings` | `trailIds` passed as something other than a string array. |
| Response shows a trail with `missing: true` | A passed `trailId` isn't in the local library — wrong id, or the trail was never authored/saved locally. |
| `500 failed to create topic` | Registry write failed — check the app logs. |

## Reference

- HTTP routes: `electron-app/src/main/topics/topicRoutes.ts` (`POST /api/topics`,
  `POST /api/topics/:id/validate-links`)
- Reference validation: `electron-app/src/main/topics/validateTopicLinks.ts`
  (+ pure layers `src/shared/utils/docReferences.ts`,
  `classifyTopicReferences.ts`)
- Registry + sync (and web-ade write-through once published):
  `electron-app/src/main/stores/TopicRegistryService.ts` (`createTopic`)
- Create input shape: `CreateTopicInput` in
  `electron-app/src/shared/main-process-api-interfaces/TopicAPI.ts`
- Sister skills: `topic-context` (local-bridge read/update), `discover-trails`
  (web-ade browse), `author-*` (trail authoring)
