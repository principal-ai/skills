---
name: publish-trail
description: Build and publish a slice/flow trail to web-ade so it can be shared as a https://app.principal-ade.com/trail/<id> link, without needing the Principal ADE desktop app installed. Use when the user says "publish a trail", "share this flow as a link", "make a shareable trail", "create a trail link", or invokes /publish-trail. Posts via the `principal-ai` CLI using the user's GitHub token (resolved locally from `gh auth token` or git credential helper). NOT for local visualization in the desktop app — use author-investigation-trail or author-informative-trail for that. NOT for PR diff walkthroughs — those need a separate diff-aware skill.
---

# Publish Trail

Build a slice/flow **trail** — markers + sequence view, anchored to file + line ranges — and publish it to web-ade so anyone with GitHub read access to the named repository can view it at `https://app.principal-ade.com/trail/<id>`.

This skill is for the **no-app / share-only** case. It produces the same payload shape as the `author-investigation-trail` / `author-informative-trail` skills, but instead of POSTing to a local Principal MCP Bridge it shells out to the published `principal-ai` CLI, which authenticates with the user's local GitHub token and POSTs to web-ade.

## When to fire

Fire on phrases like:

- "publish a trail of this flow"
- "share this sequence as a link"
- "make a shareable trail for X"
- "create a public trail" / "create a trail link"
- explicit `/publish-trail` invocation

Don't fire when:

- The user wants to *visualize* the flow inside their running desktop app — use `author-investigation-trail` (exploratory) or `author-informative-trail` (canonical).
- The user wants a PR / branch diff walkthrough — that's a sister skill (TODO).

## Prerequisites

The CLI runs via `npx`, so nothing has to be installed up front:

```bash
npx -y @principal-ai/principal-view-cli@latest trail publish --help
```

What the user *does* need:

- A working GitHub token reachable by one of the CLI's two sources (it tries them in order):
  1. `gh auth token` — usual case if they have the GitHub CLI authed.
  2. `git credential fill` for `github.com` — Keychain / Credential Manager fallback.
- That token must have **read access to the GitHub repository** the trail is being published under. The web-ade API gates publishing by repo access — if the token can read the repo, the publish succeeds.

If neither token source resolves, the CLI exits with a message pointing at `gh auth login`. Don't try to feed a token via env or argv.

## The trail schema in 60 seconds

A trail is markers + views + metadata:

```ts
interface TrailPayload {
  id: string;                     // REQUIRED — producer-supplied (e.g. crypto.randomUUID() or a stable kebab id)
  title: string;                  // REQUIRED — shown in the trail page header
  summary?: string;               // markdown — appears in the left panel until a marker is picked
  kind?: string;                  // free-form tag, e.g. 'flow', 'trace', 'lifecycle'
  authoredAt?: { sha: string; ref?: string };  // single-repo provenance shorthand
  repos?: TrailRepo[];            // multi-repo registry; omit for single-repo
  markers: TrailMarker[];         // REQUIRED, non-empty
  views: TrailView[];             // REQUIRED, non-empty — ship `[{ kind: 'sequence', ... }]`
  createdAt: string;              // REQUIRED — ISO 8601
  updatedAt: string;              // REQUIRED — ISO 8601
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

- `id` — short, stable, unique. Lowercase-kebab reads well (`callback-received`).
- `sourcePath` — **repo-relative**. Required when `snippet` is set.
- `description` — markdown. 2–6 sentences. Surface the *why* and the surrounding context.
- `snippet.kind` — always `'slice'` for this skill. (Diff snippets need bake-time file contents and are out of scope.)
- `snippet.startLine` / `endLine` — 1-based, inclusive. Keep windows tight (5–25 lines).
- `snippet.focusLine` — the single line you most want the reader to look at.

Field guidance — sequence view ref (`views[0].markers[]`):

- `markerId` — foreign key into `payload.markers[].id`.
- `name` — namespaced dot-path that reads as a fully-qualified marker identity (`auth.workos.callback.received`). The renderer derives lane buckets from the first segment.
- `participant` — explicit lane bucket override when the marker should land in a lane that doesn't match its `name`'s namespace.

Edges (`views[0].edges[]`) define order:

```json
{ "id": "e1", "fromEvent": "request-received", "toEvent": "auth-checked", "label": "then" }
```

Use `label` to convey edge semantics: `"then"`, `"on success"`, `"on 401"`, `"async"`. Field names are `fromEvent`/`toEvent` even though the trail noun is "marker" — this is the upstream renderer's edge type, reused unchanged.

## Authoring workflow

### 1. Map out the flow

Before writing JSON, get the steps straight in plain English. A useful flow has:

- A clear entry point.
- 2–6 stable lanes that bundle related markers.
- An ordered chain, with branches only where the flow genuinely forks.

If you can't articulate the lanes and order without looking at code, do the trace first (read the entry-point file, follow the calls) and only then build the payload.

### 2. Identify source locations per marker

For each step, decide whether it warrants a code snippet:

- **Yes** — the step lives at a specific call site. Capture `sourcePath` and a tight line window.
- **No** — the step is conceptual ("user clicks button", "browser issues request"). Leave `sourcePath` and `snippet` off; the marker still appears, it just won't open a snippet drawer.

Mix freely.

### 3. Build the payload

Write the full payload to a temp file (e.g. `/tmp/trail-payload.json`) so you can pipe it cleanly:

```json
{
  "id": "auth-workos-callback-flow",
  "title": "WorkOS callback flow",
  "summary": "Walks the WorkOS OAuth callback from the redirect to a signed session.",
  "authoredAt": { "sha": "abc1234", "ref": "main" },
  "markers": [ ... ],
  "views": [
    {
      "kind": "sequence",
      "markers": [ ... ],
      "edges": [ ... ],
      "layout": { "laneOrder": ["client", "auth-server", "workos"] }
    }
  ],
  "createdAt": "2026-05-07T18:00:00.000Z",
  "updatedAt": "2026-05-07T18:00:00.000Z"
}
```

For single-repo trails, the simplest path is to use `authoredAt: { sha, ref }` and pass `--owner` and `--repo` to the CLI at publish time. Owner/repo are not part of the portable payload — they identify which GitHub repo gates access to the trail.

For multi-repo trails, ship `repos: [...]` with one entry per repo and set `marker.repo` on every marker. The CLI auto-infers owner/repo from `repos[0].remote.{owner,name}`; `--owner`/`--repo` flags override.

### 4. Publish

Single-repo (recommended for first-time use):

```bash
cat /tmp/trail-payload.json | \
  npx -y @principal-ai/principal-view-cli@latest trail publish - \
    --owner <github-owner> --repo <github-repo>
```

Or with a file argument:

```bash
npx -y @principal-ai/principal-view-cli@latest trail publish /tmp/trail-payload.json \
  --owner <github-owner> --repo <github-repo>
```

Multi-repo (owner/repo inferred from `repos[0].remote`):

```bash
npx -y @principal-ai/principal-view-cli@latest trail publish /tmp/trail-payload.json
```

On success, **stdout is the share URL on a single line** — capture it and surface it to the user:

```
https://app.principal-ade.com/trail/auth-workos-callback-flow
```

On failure, the CLI exits non-zero and prints a sanitized status to stderr (`Payload.title is required [INVALID_PAYLOAD]`, `No read access to this repository (token may lack repo scope) [NO_REPO_ACCESS]`, `Could not resolve a GitHub token...`, etc.).

### 5. Tell the user what to do

After a successful publish:

1. Hand them the URL. Anyone with GitHub read access to `<owner>/<repo>` can open it.
2. Mention that publishing is **not destructive** — re-running `trail publish` with the same `id` updates the existing trail in place. To replace a published trail, just re-publish with the same id.
3. If they want to delete the published trail, the trash icon on the web-ade trail page handles that (no CLI delete in v1).

## Owner / repo: how it gates access

The web-ade publish endpoint (`POST /api/trails`) requires an explicit `<owner>/<repo>` and verifies the user's GitHub token has read access to it before storing. Two consequences worth knowing:

- The owner/repo you publish under controls **who can read the trail**. Anyone with GitHub read access to that repo can open the share link; nobody else can.
- If the user lacks read access on the repo they're trying to publish under, the publish fails with `NO_REPO_ACCESS [403]`. Tell them to check the GitHub repo membership; pass `--owner`/`--repo` only after confirming access.

When in doubt, publish under the same `<owner>/<repo>` whose code the trail is documenting.

## Validation rules to obey

The web-ade route rejects payloads that violate these. Pre-check before posting; the CLI surfaces server messages on stderr so you'll see them, but you'll be faster catching them locally.

- `id` is a non-empty string. **Reserved id**: `publish` is not allowed (it would collide with the CLI sub-action name).
- `title` is a non-empty string.
- `markers` is a non-empty array; every marker has a string `id`; marker ids are unique within the payload.
- Every marker with `snippet` also has `sourcePath`.
- For `snippet.kind: 'slice'`: `startLine`/`endLine` are positive integers with `endLine >= startLine`; `focusLine` (when present) is a positive integer; `contextLines` (when present) is non-negative.
- `views` is a non-empty array; every view has a string `kind`. v1 fully validates `kind: 'sequence'` blocks.
- For sequence views: every `markerId` references an existing marker; every entry has a string `name`; every edge has string `id`, `fromEvent`, `toEvent` (and `fromEvent`/`toEvent` should reference valid marker ids).
- Body must stay under 10MB.

## Authoring quality bar

Trails that read well share these traits:

- **One thing per marker.** If a step contains an `if/else` whose branches diverge, model them as two markers with separate edges.
- **Stable lanes.** Reusing 3 lanes across 12 markers makes the trail clean; using 12 different lanes makes it confetti.
- **Snippets land on the right line.** `focusLine` should point at the *call* or *decision* the marker is about, not at the surrounding boilerplate.
- **Descriptions answer "why this exists" or "what's surprising here".**
- **Order matches reading order.** The first marker is where you'd start explaining the flow on a whiteboard.

## Naming conventions

Good `name` values (on the sequence view ref) are namespaced and read as a sentence on their own:

| Flow | Examples |
|---|---|
| OAuth | `auth.workos.callback.received`, `auth.workos.code.exchanged`, `auth.session.created` |
| Ingest pipeline | `ingest.batch.received`, `ingest.batch.validated`, `ingest.batch.persisted` |
| UI boot | `ui.app.bootstrap`, `ui.workspace.layout.loaded`, `ui.panel.first-render` |
| Cross-process IPC | `ipc.invoke.fileSystem.readFile`, `ipc.handler.fileSystem.readFile`, `ipc.return` |

Stick with one prefix per flow. Avoid generic names like `step1`, `handler`, `do-thing`.

## Lane ordering

The sequence renderer resolves each marker's lane by reading its dotted `name`. Left-to-right order is controlled two ways:

- **Default — first-marker order.** The lane whose first marker appears earliest in `views[0].markers[]` becomes leftmost. For most flows this is enough.
- **Explicit — `views[0].layout.laneOrder`.** Pass an array of resolved namespaces left-to-right. Listed lanes appear first in the given order; unlisted lanes fall back to first-marker order behind them.

```json
{
  "kind": "sequence",
  "markers": [...],
  "edges": [...],
  "layout": { "laneOrder": ["client", "auth", "workos", "database"] }
}
```

Use explicit `laneOrder` when the temporal/causal order doesn't match the spatial layout you want.

## Common shapes

### Trace a request through a multi-package monorepo

Each hop is a marker; `sourcePath` points at the entry function in that hop's package; `name` puts each marker in its package's lane.

```
ui  →  api-gateway  →  auth-service  →  user-service  →  db
```

### Document a startup / lifecycle sequence

Markers run in temporal order: bootstrap → config → service init → first render. Snippets land on the line where each phase begins. Useful for onboarding contributors.

### Bare trail with no code

If the flow is conceptual ("user clicks 'export', browser downloads file, server logs it"), skip `sourcePath` and `snippet` on most markers. The trail still renders cleanly; the share page just won't open snippet drawers for those markers.

### Multi-repo trail

Ship `repos: [...]` with one entry per repo and set `marker.repo` on every marker. The CLI auto-infers owner/repo from `repos[0].remote`:

```json
{
  "repos": [
    { "id": "auth-server", "name": "auth-server", "remote": { "host": "github", "owner": "principal-ai", "name": "auth-server" }, "authoredAtSha": "abc1234" },
    { "id": "api-gateway", "name": "api-gateway", "remote": { "host": "github", "owner": "principal-ai", "name": "api-gateway" }, "authoredAtSha": "def5678" }
  ],
  "markers": [
    { "id": "ingress",   "repo": "api-gateway", "sourcePath": "src/server.ts", ... },
    { "id": "auth-call", "repo": "auth-server", "sourcePath": "src/routes/verify.ts", ... }
  ]
}
```

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `Could not resolve a GitHub token...` | `gh` not installed/authed and no git credential helper for `github.com`. Tell the user to run `gh auth login`. |
| `No read access to this repository (token may lack repo scope) [NO_REPO_ACCESS]` | Token doesn't have read access to `<owner>/<repo>`. Check GitHub repo membership / token scopes. |
| `Payload.<field> is required [INVALID_PAYLOAD]` | Server validation rejected the payload — fix the named field and re-publish. |
| `Could not determine owner/repo. Pass --owner and --repo, or include 'repos[0].remote' in the payload.` | Single-repo payload with no flags. Add `--owner`/`--repo` or populate `repos[0].remote`. |
| `Trail id 'publish' is reserved` | You generated `id: "publish"`. Pick a different id (this collides with the CLI sub-action name). |
| Network error | The CLI couldn't reach `app.principal-ade.com`. Check connectivity. |
| Re-publishing creates duplicates | The `id` is changing between runs. Pick a stable id (kebab string or uuid you persist) so re-publishing updates in place. |

## Reference

- Sister skills for local visualization in the desktop app: `author-investigation-trail`, `author-informative-trail`
- CLI: `@principal-ai/principal-view-cli` — `principal-ai trail publish --help`
- Web-ade endpoint: `POST https://app.principal-ade.com/api/trails` (gated by GitHub repo read access)
