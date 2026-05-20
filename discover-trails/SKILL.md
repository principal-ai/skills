---
name: discover-trails
description: Browse trails and topics that already exist on web-ade — list everything a given user has published, fetch a specific trail or topic by id or URL, or list the topics curated by a user. Read-only; no publishing, no editing. Use when the user says "what trails has X published", "show me my recent trails", "list topics from <user>", "summarize this trail", "what's in this topic", "find trails about <subject>", or invokes /discover-trails. Posts via the `principal-ai` CLI using the user's GitHub token (resolved locally from `gh auth token` or git credential helper). NOT for authoring or publishing — use publish-trail or the author-* skills.
---

# Discover Trails

Browse the read side of web-ade — find out what trails and topics already
exist, fetch their payloads, and inspect them — without ever publishing
or editing anything.

This is the **read-only companion** to `publish-trail` and `create-topic`.
Same CLI (`principal-ai`), same GitHub token chain (gh CLI → git credential
helper), no app required. The four endpoints this skill covers form the
discovery surface every other web-ade-facing skill builds on.

## When to fire

Fire on phrases like:

- "what trails has @alice published?"
- "list my recent trails" / "show me what I've shared"
- "list topics by @alice" / "what topics has @alice curated?"
- "fetch this trail" / "what does <trail URL> say?" / "summarize this trail"
- "what's in this topic?" / "show me the trails in <topic URL>"
- "find trails about <subject>" (use the listing + read pattern below)
- explicit `/discover-trails` invocation

Don't fire when:

- The user wants to **author** a trail — use `author-investigation-trail`
  (in-app, exploratory), `author-informative-trail` (in-app, canonical),
  or their `author-local-*` siblings (no app).
- The user wants to **publish** a local trail to web-ade — use
  `publish-trail`.
- The user wants to **curate** a topic — use `create-topic`.
- The user wants to **review a PR** — use `file-city-trail-review`.

## Prerequisites

The CLI runs via `npx`, so nothing has to be installed up front:

```bash
npx -y @principal-ai/principal-view-cli@latest trail list --help
npx -y @principal-ai/principal-view-cli@latest topic list --help
```

A GitHub token is **required for `list`** (the CLI resolves the
authenticated user via the GitHub API, or maps `--user <login>` to a
numeric id) and **optional for `view` / fetching by id** (those endpoints
are public-by-link, but the CLI will attach a token if available so rate
limits favor authenticated callers).

The token chain — same as every other web-ade-facing skill:

1. `gh auth token` — usual case if they have the GitHub CLI authed.
2. `git credential fill` for `github.com` — Keychain / Credential Manager
   fallback.

If neither resolves, the CLI exits with a message pointing at
`gh auth login`. Don't try to feed a token via env or argv.

## The discovery surface

Four operations cover the whole read side. Two for trails, two for
topics, all parameterless beyond user/id arguments.

| Operation | CLI | HTTP | Purpose |
| --- | --- | --- | --- |
| List trails by user | `principal-ai trail list [--user <login> \| --id <n>]` | `GET /api/trails/by-user/{githubId}` | What has this user published? |
| Fetch trail by id | `principal-ai trail <id-or-url>` | `GET /api/trails/by-id/{id}` | What does this specific trail say? |
| List topics by user | `principal-ai topic list [--user <login> \| --id <n>]` | `GET /api/topics/by-user/{githubId}` | What collections has this user curated? |
| Fetch topic by id | `principal-ai topic view <id-or-url>` | `GET /api/topics/by-id/{id}` | What trails are in this topic, in what order? |

The `list` endpoints read a per-user manifest (`trails/_by-user/{githubId}.json`
or `topics/_by-user/{githubId}.json`) that web-ade maintains on every
publish/delete. Listings are O(1) — they don't scan repos.

## What the listings give you

`trail list` returns a JSON envelope:

```ts
{ entries: TrailByUserEntry[] }

interface TrailByUserEntry {
  id: string;
  title: string;
  summaryPreview: string;     // first ~200 chars of `summary`, single-lined
  markerCount: number;
  repoNames: string[];        // for multi-repo trails
  hasDiffSnippets: boolean;
  owner: string;              // GitHub owner of the repo the trail was published under
  repo: string;
  createdBy: { githubId: number; githubLogin: string };
  githubRepoId: number;
  createdAt: string;          // ISO 8601
  updatedAt: string;          // ISO 8601
  sizeBytes: number;
}
```

`topic list` returns the same envelope shape:

```ts
{ entries: TopicByUserEntry[] }

interface TopicByUserEntry {
  id: string;
  title: string;
  descriptionPreview: string; // first ~140 chars of `description`, single-lined
  trailCount: number;
  createdAt: string;
  updatedAt: string;
}
```

Entries come back sorted by `updatedAt` descending — "most recent first."

## Recipes

### What have I published lately?

```bash
npx -y @principal-ai/principal-view-cli@latest trail list | jq '.entries[0:5]'
npx -y @principal-ai/principal-view-cli@latest topic list | jq '.entries[0:5]'
```

The bare form resolves "me" via the GitHub token's identity, so no
`--user` is needed.

### What has @alice published?

```bash
npx -y @principal-ai/principal-view-cli@latest trail list --user alice | jq
```

Login is resolved to a numeric GitHub id via the public GitHub API. Pass
`--id 12345` to skip the lookup if you already have the numeric id.

### What's in this trail?

Two forms — bare id or full URL — both work for the fetch:

```bash
npx -y @principal-ai/principal-view-cli@latest trail 7c3e1a0b-… | jq .title,.markers,.views
npx -y @principal-ai/principal-view-cli@latest trail https://app.principal-ade.com/trail/7c3e1a0b-…
```

Fetched trails are cached locally; pass `--refresh` to `trail view` if
you want to bypass the cache (the bare fetch always re-reads).

### What trails are in this topic?

```bash
# Get the topic record (includes ordered trailIds)
TOPIC_JSON=$(npx -y @principal-ai/principal-view-cli@latest topic view "https://app.principal-ade.com/topic/<id>")
echo "$TOPIC_JSON" | jq .title,.description,.trailIds

# Fetch each trail in the topic, in order
echo "$TOPIC_JSON" | jq -r '.trailIds[]' | while read TID; do
  npx -y @principal-ai/principal-view-cli@latest trail "$TID" | jq -r '"\(.title) — \(.summary // "no summary")"'
done
```

### Find trails by subject across multiple users

There's no full-text search endpoint yet. The pattern is "list per user
you know about, then filter by title/summaryPreview client-side":

```bash
for U in alice bob charlie; do
  npx -y @principal-ai/principal-view-cli@latest trail list --user "$U" \
    | jq --arg user "$U" '.entries[] | select(.title | test("auth"; "i")) | {user: $user, title, id, repo: "\(.owner)/\(.repo)"}'
done
```

If you find the same subject covered by several users' trails, that's a
strong signal to suggest curating a topic via `create-topic`.

### Summarize "what's @alice been working on"

```bash
USER=alice
TRAILS=$(npx -y @principal-ai/principal-view-cli@latest trail list --user "$USER" | jq '.entries[0:10]')
TOPICS=$(npx -y @principal-ai/principal-view-cli@latest topic list --user "$USER" | jq '.entries[0:5]')

jq -n --argjson trails "$TRAILS" --argjson topics "$TOPICS" \
  '{recent_trails: $trails, recent_topics: $topics}' | jq
```

Pipe the JSON into your model context for a single-shot summary. The
`summaryPreview` and `descriptionPreview` fields are pre-trimmed so the
context stays small.

## Auth + access posture

- **`/api/*/by-user/{githubId}`** — public. Anyone can list a user's
  trails/topics. The manifest carries only summaries (id, title,
  previews, counts, timestamps); the per-trail payload is still gated
  separately.
- **`/api/trails/by-id/{id}`** — gated by GitHub repo read access. A
  caller without access to the trail's repo gets a 403 / "No read
  access" — even though they could see the entry in the list. Errors
  surface verbatim through `describeHttpError`.
- **`/api/topics/by-id/{id}`** — public. The topic record carries no
  repo-access metadata of its own; individual trails inside the topic
  still self-gate when fetched.

In practice: a trail listed but not readable means the lister has shared
something the current viewer can't see. Mention this in any summary you
produce so the user knows there's something they don't have access to.

## Common errors

| HTTP / code | Meaning | What to do |
| --- | --- | --- |
| 401 `NOT_AUTHENTICATED` | Token missing / rejected | `gh auth login` (or check `git credential fill` for github.com) |
| 403 `NO_REPO_ACCESS` (trail by-id) | The trail's repo isn't visible to this token | Ask the publisher to invite the viewer, or skip the trail in the summary |
| 404 `NOT_FOUND` | Trail / topic id doesn't resolve | Wrong id, deleted, or never minted |
| 400 (invalid githubId) | `--id` wasn't a positive number | Pass `--user <login>` instead |
| GitHub user not found | `--user <login>` doesn't exist on GitHub | Check the spelling; GitHub logins are case-insensitive |

## What this skill doesn't do

- It doesn't **search the full text** of a trail's markers — listings
  carry only `summaryPreview`. For deeper search, fetch the trail by id
  and grep the payload client-side.
- It doesn't **list a repo's trails** — `/api/trails/{owner}/{repo}` is
  the repo-scoped index, but the CLI surface for it is deferred. Use the
  per-user listings or open the repo page on web-ade for now.
- It doesn't **discover trails you haven't been told about** — there's
  no global "all recent trails" feed in v1. Discovery is per-user.
- It doesn't **modify anything** — every command here is `GET`. Pair
  with `publish-trail` / `create-topic` for the write side.
