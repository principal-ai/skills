---
name: search-local-topics-trails
description: Search the trails and topics already stored on this machine — read the on-disk indexes under ~/.principal, filter by title / summary / repo / recency, and open a matching trail in the viewer with `principal-ai trail view <id>` or a matching topic with `principal-ai topic open <id>`. Read-only; no authoring, no publishing, no network. Use when the user says "what local trails do I have", "find my trail about X", "do I have a trail for this repo", "search my topics for Y", "open that trail", "pull up the trail for the fundraising repo", or invokes /search-local-topics-trails. NOT for browsing web-ade (published) trails — use discover-trails. NOT for authoring — use the author-* skills.
---

# Search Local Topics & Trails

Find the trails and topics that already live **on this machine** — the ones
authored locally or cached from web-ade — and open a trail in the viewer for
the user to review. This is the **local, on-disk** read companion to
`discover-trails` (which reads the published web-ade side). No app required,
no GitHub token, no network: everything here is a file read plus one CLI call.

## When to fire

Fire on phrases like:

- "what local trails do I have?" / "list my trails"
- "find my trail about <subject>" / "do I have a trail for <repo>?"
- "search my topics for <subject>" / "what topics are on disk?"
- "open that trail" / "pull up the trail for the fundraising repo"
- explicit `/search-local-topics-trails` invocation

Don't fire when:

- The user wants **published** trails/topics on web-ade ("what has @alice
  shared") — use `discover-trails`.
- The user wants to **author** a trail — use `author-investigation-trail`
  (in-app, exploratory) or `author-informative-trail` (in-app, canonical).
- The user wants to **read or update a topic's description** — use
  `topic-context`.
- The user wants the **HTTP bridge routes** the running app exposes — use
  `principal-ai-desktop-app-tools`.

## Where they live on disk

Everything is under `~/.principal`. Each kind has a single index file that is
the only thing you need to read to **search** — the indexes carry trimmed
previews, so you almost never have to open a full payload.

| Kind | Index (search here) | Payloads |
| --- | --- | --- |
| Trails | `~/.principal/trails/_index.json` | `~/.principal/trails/<owner>/<repo>/<id>.json`, `local/<path-slug>/<id>.json`, `by-id/<id>.json` |
| Topics | `~/.principal/topics/_index.json` | `~/.principal/topics/<id>.json` (one file per topic) |

Both indexes share the shape `{ "version": 2, "entries": [ … ] }`.

A **trail** entry:

```ts
{
  id: string;              // pass this to `trail view`
  title: string;
  summaryPreview: string;  // first ~200 chars of the summary, single-lined
  markerCount: number;
  repoNames: string[];
  hasDiffSnippets: boolean;
  createdAt: string;       // ISO 8601
  updatedAt: string;
  sizeBytes: number;
  repositoryPath: string;  // absolute path of the working tree it was laid in
  cachePath: string;       // payload location relative to ~/.principal/trails/
  fileCount: number;
  signOffCount: number;
}
```

A **topic** entry:

```ts
{
  id: string;                 // e.g. topic-1779510063456-5dwl9ycjw
  title: string;
  descriptionPreview: string; // first ~140 chars of the description
  trailCount: number;
  state: string;              // e.g. "done", "working"
  createdAt: string;
  updatedAt: string;
  hasAssets: boolean;
  sizeBytes: number;
  fileName: string;           // file under ~/.principal/topics/
}
```

There is **no CLI command that lists local trails/topics** — `trail list` and
`topic list` both hit web-ade. Local search is reading these two index files
and filtering them yourself.

## Searching

Read the index and filter with `jq`. Entries are most-useful sorted by
`updatedAt` descending.

### What local trails do I have?

```bash
jq -r '.entries
  | sort_by(.updatedAt) | reverse
  | .[0:15][]
  | "\(.id)\t\(.title)\t\(.repositoryPath)"' ~/.principal/trails/_index.json
```

### Find a trail by subject

Match across title and summary preview (case-insensitive):

```bash
jq -r --arg q "fundraising" '.entries[]
  | select((.title + " " + .summaryPreview | ascii_downcase) | contains($q | ascii_downcase))
  | "\(.id)\t\(.title)"' ~/.principal/trails/_index.json
```

### Trails for a specific repo

```bash
jq -r --arg repo "/Users/griever/Developer/principal-ade/fundraising" \
  '.entries[] | select(.repositoryPath == $repo) | "\(.id)\t\(.title)"' \
  ~/.principal/trails/_index.json
```

### Search topics

```bash
jq -r --arg q "auth" '.entries[]
  | select((.title + " " + .descriptionPreview | ascii_downcase) | contains($q | ascii_downcase))
  | "\(.id)\t\(.title)\t(\(.trailCount) trails, \(.state))"' \
  ~/.principal/topics/_index.json
```

When several candidates match, show the user the shortlist (id + title +
repo/preview) and let them pick before opening anything.

## Opening a trail for review

Hand the matched `id` to the CLI:

```bash
principal-ai trail view <id>
```

(If `principal-ai` isn't on the PATH, the same command is
`npx -y @principal-ai/principal-view-cli@latest trail view <id>`.)

That opens the trail in the viewer so the user can review it. You can pass an
id, a full `https://app.principal-ade.com/trail/<id>` URL, or a payload file
directly with `--file <path>`. Don't narrate *where* it opens — just open it
and tell the user it's up.

Example, end to end — "open the trail for the fundraising repo":

```bash
ID=$(jq -r --arg repo "/Users/griever/Developer/principal-ade/fundraising" \
  '.entries[] | select(.repositoryPath == $repo) | .id' \
  ~/.principal/trails/_index.json | head -1)
principal-ai trail view "$ID"
```

## Opening a topic for review

Hand the matched `id` to the CLI:

```bash
principal-ai topic open <id>
```

(If `principal-ai` isn't on the PATH, the same command is
`npx -y @principal-ai/principal-view-cli@latest topic open <id>`.)

That opens the topic in the **running desktop app** (via the 3044 bridge's
`POST /api/topics/:id/activate` route). When no desktop app is available it
falls back to opening the web-ade page in the default browser. The app must
be running for the local desktop-app path; the browser fallback works without
the app.

Example — "open the topic about moving a terminal between windows":

```bash
principal-ai topic open topic-1782763212946-kw5zfpaej
```

## What this skill doesn't do

- It doesn't **list or fetch web-ade (published) trails/topics** — that's
  `discover-trails`.
- It doesn't **full-text search inside a trail's markers** — the indexes only
  carry `summaryPreview` / `descriptionPreview`. For deeper matching, read the
  payload at `~/.principal/trails/<cachePath>` and grep it.
- It doesn't **open a topic in a standalone viewer** — topics are surfaced via the desktop app or browser (see above).
- It doesn't **author, publish, or edit** anything — every step here is a read
  plus, at most, `trail view`.
