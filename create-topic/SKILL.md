---
name: create-topic
description: Create a topic on web-ade — a curated collection of trails on a shared subject, e.g. "how do these three agent CLIs detect filesystem changes?" — and share it as a https://app.principal-ade.com/topic/<id> link. Topics group existing trails (by id or URL) into a single page so readers can compare implementations across repos. Use when the user says "create a topic", "make a topic out of these trails", "group these trails into a topic", "curate a topic on X", or invokes /create-topic. Posts via the `principal-ai` CLI using the user's GitHub token (resolved locally from `gh auth token` or git credential helper). NOT for authoring a single trail — use author-{investigation,informative}-trail (in-app), then publish from the Principal app UI. NOT for editing a topic that already exists — use the topic's owner UI on web-ade or call the routes directly.
---

# Create Topic

Group existing trails into a **topic** on web-ade and share it as
`https://app.principal-ade.com/topic/<id>`. The topic record is just
`{ title, description, ordered trailIds, owner }` — it doesn't own the
trails it lists. Each embedded trail keeps its own notes, sign-offs, and
`/trail/<id>` page; the topic is a curation layer on top.

This skill is the **no-app / web-only** path. Topics live entirely on
web-ade; there is no desktop-app surface for them. The skill shells out to
the published `principal-ai` CLI, which authenticates with the user's
local GitHub token and POSTs to web-ade.

## When to fire

Fire on phrases like:

- "create a topic of these trails"
- "make a topic out of <trail1>, <trail2>, <trail3>"
- "curate a topic on <subject>"
- "group these flows into a topic and share the link"
- "compare these trails side-by-side in a topic"
- explicit `/create-topic` invocation

Don't fire when:

- The user wants to **author a single trail** — use `author-investigation-trail` /
  `author-informative-trail` (in-app authoring), then publish from the app UI.
- The user wants to **edit a topic that already exists** — the
  owner-only edit affordances live on `https://app.principal-ade.com/topic/<id>`.
  For trail add/remove from a script use the
  `principal-ai topic add-trail` subcommand directly.
- The user wants a **PR diff walkthrough** — that's a different surface.

## Prerequisites

The CLI runs via `npx`, so nothing has to be installed up front:

```bash
npx -y @principal-ai/principal-view-cli@latest topic --help
```

What the user *does* need:

- A working GitHub token reachable by one of the CLI's two sources (it
  tries them in order):
  1. `gh auth token` — the usual case if they have the GitHub CLI authed.
  2. `git credential fill` for `github.com` — Keychain / Credential Manager
     fallback.
- The trails they want to embed must already exist on web-ade (so they
  resolve via `trails/_by-id/<id>.json`). Publish them from the app UI first if
  any of the trails are still local-only.

The user does **not** need:

- The desktop app running. Topics are web-only.
- Read access to every embedded trail's repo. The topic record itself is
  public-by-link; individual trails still self-gate on repo read access
  on the per-trail page. A reader without access to one embedded trail
  sees an inline "in a private repository" card without losing the rest
  of the topic.

If neither token source resolves, the CLI exits with a message pointing at
`gh auth login`. Don't try to feed a token via env or argv.

## The topic shape in 60 seconds

```ts
interface TopicPayload {
  id: string;                    // server-minted uuid
  title: string;                 // ≤ 200 chars
  description: string;           // markdown, ≤ 8 000 chars
  trailIds: string[];            // ordered; foreign keys into trails/_by-id/
  createdBy: { githubId: number; githubLogin: string };
  createdAt: string;             // ISO 8601
  updatedAt: string;             // ISO 8601
}
```

Caps (enforced server-side):

- `MAX_TITLE_CHARS = 200`
- `MAX_DESCRIPTION_CHARS = 8_000`
- `MAX_TRAILS_PER_TOPIC = 50`

The server mints `id`, `createdBy`, `createdAt`, and `updatedAt` — the
client only supplies `title`, `description`, and `trailIds`.

## The flow

1. **Curate the trail list.** Collect the trail ids or `/trail/<id>` URLs
   you want to include. Order matters — the topic page renders them as a
   numbered list in this order. The CLI accepts either form via `--trail`
   (repeatable).

2. **Pick a title + description.** Title is the page heading. Description
   is markdown — use it to explain the subject and what each trail
   contributes (e.g. "Three agent CLIs solving filesystem change
   detection — A picks inotify, B polls, C delegates to OS-level
   FSEvents"). Both can be edited later via the owner UI; the trail list
   can be added/reordered/removed without recreating the topic.

3. **Create the topic.** `principal-ai topic create` POSTs to
   `/api/topics`, validates each `trailId` resolves before minting, and
   prints the resulting share URL on stdout.

4. **(Optional) Add more trails.** `principal-ai topic add-trail
   <topic> <trail>` appends a single trail. Useful when adding trails
   that get published after the topic was created. Order follows
   insertion order; use the owner UI on `/topic/<id>` for a manual
   reorder (the API accepts a permutation but the CLI doesn't surface a
   reorder subcommand yet).

5. **Share the URL.** Anyone with a link can read the topic record;
   per-trail repo access is enforced on the embedded trail pages.

## Recipes

### One-liner: create a topic with three trails

```bash
npx -y @principal-ai/principal-view-cli@latest topic create \
  --title "Filesystem change detection across three CLIs" \
  --description "Side-by-side: how each agent picks up on edits made during a long-running session." \
  --trail https://app.principal-ade.com/trail/4f2b… \
  --trail https://app.principal-ade.com/trail/9d1a… \
  --trail 7c3e1a0b-…
```

Stdout on success: `https://app.principal-ade.com/topic/<id>` — share
that.

### Create empty, then add trails one at a time

```bash
TOPIC_URL=$(npx -y @principal-ai/principal-view-cli@latest topic create \
  --title "Agent FS-change detection" \
  --description "Compare across CLIs as their trails are published.")
echo "$TOPIC_URL"

npx -y @principal-ai/principal-view-cli@latest topic add-trail "$TOPIC_URL" https://app.principal-ade.com/trail/4f2b…
npx -y @principal-ai/principal-view-cli@latest topic add-trail "$TOPIC_URL" 9d1a…
```

`topic add-trail` accepts the topic id or URL on the first arg and the
trail id or URL on the second. Adds are idempotent in spirit — adding the
same trail twice returns a 409 (`TRAIL_ALREADY_ADDED`).

### Inspect or list

```bash
# Pretty-print a topic record
npx -y @principal-ai/principal-view-cli@latest topic view 7c3e1a0b-… | jq

# List topics for the authenticated user
npx -y @principal-ai/principal-view-cli@latest topic list

# List topics for a different user by GitHub login
npx -y @principal-ai/principal-view-cli@latest topic list --user octocat
```

## Common errors

| Server code | Meaning | What to do |
| --- | --- | --- |
| `TRAIL_NOT_FOUND` | A `--trail` value doesn't resolve via `trails/_by-id/<id>.json` | The trail needs to be published first; check the id with `principal-ai trail <id>` |
| `TRAIL_ALREADY_ADDED` | Tried to `add-trail` a trail already in the topic | Skip it; the topic already has it |
| `TOO_MANY_TRAILS` | Topic hit `MAX_TRAILS_PER_TOPIC = 50` | Split into two topics or remove a stale entry first |
| `NOT_OWNER` (403) | Mutating a topic owned by someone else | Topics are owner-only mutates; create your own copy |
| `NOT_AUTHENTICATED` (401) | Token rejected or absent | `gh auth login` (or check `git credential fill` for github.com) |

## Author discipline — what makes a good topic

A topic is a curation, not a folder. The description should make the
**comparison axis** explicit: what dimension are these trails varying
along, and what's the reader supposed to notice?

Bad (just a list):
> Three trails about agent CLIs.

Good (comparison axis stated):
> Three CLIs solve "detect filesystem changes during a long-running
> session" three different ways: inotify-driven invalidation in Trail 1,
> a 200 ms polling loop in Trail 2, and OS-level FSEvents delegation in
> Trail 3. Read them in order to see how the trade-off between
> responsiveness and CPU shifts at each step.

Order the trails so the reader's mental model builds across the list.
The topic page numbers them; that ordering is editorial, not arbitrary.

## What this skill doesn't do

- It doesn't **author trails** — it only curates existing ones. Author with
  the in-app authoring skills and publish from the app UI first.
- It doesn't **edit titles / descriptions** after creation — use the
  owner UI on `https://app.principal-ade.com/topic/<id>` or `PATCH
  /api/topics/by-id/<id>` directly.
- It doesn't **reorder** the trail list — the API accepts a permutation
  via `PATCH /api/topics/by-id/<id>/trails`, but the CLI surface for that
  is deferred. The owner UI handles reorders today.
- It doesn't **delete** topics — use the owner UI or `DELETE
  /api/topics/by-id/<id>` directly. Deletes leave embedded trails
  untouched.
