---
name: principal-ai-desktop-app-tools
description: Discover the local HTTP API exposed by the running Principal desktop app through the Principal MCP Bridge (production port 3044, development port 3054). Explains bridge categories, GET /routes discovery, and POST /routes batch documentation lookup. Use when you need to know what the bridge supports, find a request contract, open a GitHub repository in a native Principal tab, clone or register a local repository checkout, watch a GitHub pull request so the user sees its checks/review/merge status, open a localhost preview, or diagnose a local request. The bridge is local and unauthenticated. Workflow skills own topic, document-note, and File City tour workflows.
---

# Principal Desktop App Tools

The **Principal MCP Bridge** is the running desktop app's local HTTP API.
Discover available endpoints and fetch their contracts from the intended running
build instead of relying on copied request schemas in a skill.

## Connect

- Production: `http://localhost:3044`
- Development: `http://localhost:3054`

Choose the intended build explicitly, especially when both are running. These
examples target production:

```bash
BASE=http://localhost:3044
curl -fsS "$BASE/health"
```

If the probe fails, ask the user to launch the intended app. Do not guess
another port or silently switch builds.

## Categories

`GET /routes` returns a lightweight directory grouped by category. Each group
has `name`, `description`, and endpoint summaries (`method`, `path`, `summary`).

| Category           | Capabilities                                                                                                                                 |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `diagnostics`      | Health, category discovery, and endpoint documentation lookup                                                                                |
| `topics`           | Create/search/read/edit/open/delete local briefs, validate links, and attach repositories, subsystem models, C4 sketches, and related topics |
| `repos`            | Open native GitHub repository tabs; discover, clone, register, and de-register local repositories; resolve their PURLs                       |
| `pull-requests`    | Watch GitHub pull requests: the app polls checks, review, and merge state, lists them on Home, and notifies the user on changes             |
| `dependency-graph` | Find direct dependents among analyzed local clones                                                                                           |
| `dependencies`     | Resolve a dependency using repository-monitoring context                                                                                     |
| `document-notes`   | Read/write text-anchored note overlays without editing document source                                                                       |
| `documents`        | Open a document in an already-focused document-capable window                                                                                |
| `processes`        | Start/inspect/stop/restart/remove app-owned managed processes                                                                                |
| `browser`          | Open localhost preview tabs, optionally linked to managed processes                                                                          |
| `theme`            | Inspect theme schemas and update theme preferences                                                                                           |
| `bruno`            | Validate/parse Bruno request content, files, and collections without executing requests                                                      |

Use the running build's directory as the source of available categories and
exact route patterns:

```bash
curl -fsS "$BASE/routes"
```

GET discovery lists summaries, not full request/response contracts. It does not
filter by category; select the relevant group from the returned directory.

## Look up endpoint contracts

**POST `/routes`** is a read-only batch documentation lookup. It does not invoke
the endpoints named in the request.

```bash
curl -fsS "$BASE/routes" \
  -H 'Content-Type: application/json' \
  -d '{
    "routes": [
      {"method":"POST","path":"/api/topics"},
      {"path":"/api/processes/:id/logs"},
      {"method":"POST","path":"/api/browser/open"}
    ]
  }'
```

Request:

- `routes`: required array of 1–100 objects.
- `path`: required exact registered pattern from GET discovery, such as
  `/api/topics/:id`, **not** a concrete ID, full URL, prefix, or glob. Do not
  include query strings or fragments.
- `method`: optional, case-insensitive HTTP method. Omit it to retrieve every
  documented method on that path.

Response:

```json
{
  "success": true,
  "results": [
    {
      "request": { "method": "POST", "path": "/api/topics" },
      "found": true,
      "routes": [
        {
          "category": "topics",
          "method": "POST",
          "path": "/api/topics",
          "summary": "...",
          "body": { "title": "..." },
          "response": "...",
          "errors": { "400": "..." },
          "notes": ["..."],
          "example": { "title": "..." }
        }
      ]
    }
  ]
}
```

Each matching contract includes its category, method, path, summary, response,
and applicable query/body/path parameters, errors, notes, and example.
Results preserve request order. An unknown path or method/path pair returns
`found: false`, empty `routes`, and an `error` for that entry; other lookups
still succeed. A malformed entry or invalid batch size rejects the whole
request with HTTP `400`.

Fetch the relevant contracts **before constructing an action request**. Read
their notes as well as their fields: UI delivery, process lifecycle, and
validation success have endpoint-specific meanings.

Older builds may lack POST lookup and may return an incomplete GET catalog.
If lookup returns `404`, do not assume the requested action endpoint is
missing. Report the build limitation and ask for an updated app rather than
guessing contracts or silently switching builds.

## Open GitHub repositories in Principal

For GitHub repository links, prefer the existing native repository explorer
instead of a browser preview. Discover and fetch the contract for
`POST /api/repos/open` before calling it:

```bash
curl -fsS "$BASE/api/repos/open" \
  -H 'Content-Type: application/json' \
  -d '{"repo":"https://github.com/TimBroddin/gitstatus"}'
```

- Accepts a GitHub repository root URL, `owner/repo`, or
  `pkg:github/owner/repo`. Optional `.git` suffix and trailing slash are allowed.
- Reuses Quick Open's repository explorer, opening or focusing a tab by PURL.
  Works without a local clone. Opening does not clone, register, or fork.
- Requires an already-focused Principal MAIN window. No new windows are
  created or focused. If `delivered: false`, ask the user to focus MAIN.
- `delivered: true` confirms event dispatch only, not renderer acknowledgement
  or successful GitHub data loading. Report that distinction accurately.
- Rejects other hosts, local paths, credentials, query strings, fragments,
  and repository subpaths. This is not an arbitrary URL-opening endpoint.
- If the running build does not list this endpoint, ask for an updated app.
  Do not send GitHub URLs to `/api/browser/open`, which remains localhost-only.

## Watch a pull request for the user

After opening or pushing to a pull request, add it to the app's watch list so
the user can follow it without asking you to check. Fetch the contract for
`POST /api/pull-requests/watch` before calling it:

```bash
curl -fsS "$BASE/api/pull-requests/watch" \
  -H 'Content-Type: application/json' \
  -d '{"pr":"https://github.com/owner/repo/pull/123"}'
```

- Accepts a GitHub pull request URL, `owner/repo#123`, or
  `owner/repo/pull/123`. Returns `201` with the first snapshot, or `200` with
  `alreadyWatched: true` when the PR was already on the list.
- This is for the user, not for you. The app polls GitHub about once a minute,
  shows the PR under **Pull Requests** on Home with a link to open it in the
  browser, and raises a desktop notification when checks finish, a review is
  decided, or the PR is merged or closed. Do not poll this route or
  `GET /api/pull-requests/watch` to wait for CI.
- `404` means the PR could not be loaded (unknown, private without access, or
  GitHub unreachable) and nothing was added. Report that rather than retrying.
- `DELETE /api/pull-requests/watch/:owner/:repo/:number` stops watching.
  Merged and closed PRs drop off on their own after a day.
- If the running build does not list the `pull-requests` category, ask for an
  updated app.

## Local clones

The `repos` category owns the checkout lifecycle. Fetch the contracts with POST
`/routes` before acting; the workflow is:

| Need                                          | Route                   |
| --------------------------------------------- | ----------------------- |
| See what is cloned locally                    | `GET /api/repos`        |
| Clone by PURL, `owner/repo`, or clone URL     | `POST /api/repos/clone` |
| Register an existing checkout                 | `POST /api/repos`       |
| Unregister (and optionally delete) a checkout | `DELETE /api/repos`     |

- `GET /api/repos` returns registered clones as
  `{ path, name, remoteUrl?, purl?, registeredAt }`. Use `purl` — never a
  filesystem path — when writing a topic's `repos`.
- `POST /api/repos/clone` accepts repo PURLs (`pkg:github/owner/repo`),
  `owner/repo` slugs, and clone URLs, and clones into the app's convention:
  `<baseDefaultDirectory>/<owner>/<repo>`. Pass `depth` for a shallow clone and
  `register: false` to skip registration. Entries succeed or fail individually —
  read `results[]` and retry only the entries with `ok: false`.
- Cloning requires a base directory configured in the app
  (Settings → General); without one the route returns `400` with that message.
  Clones are slow: the request holds until they finish, up to five minutes each.
- `DELETE /api/repos` only unregisters by default. `deleteLocal=true` also
  deletes the checkout on disk and is refused for relative paths, non-git
  directories, filesystem roots, and the base directory itself.
- Local-only repositories cannot be cloned. Register existing checkouts with
  `POST /api/repos` and an absolute path.

## Shared conventions and safety

- Local, unauthenticated API. Do not expose the bridge publicly.
- JSON bodies use `Content-Type: application/json`; the limit is 10 MB.
- URL-encode path identifiers and query values on action requests.
- `repositoryPath` is route-specific, not universally required. Repository
  PURLs and absolute local paths are different identifiers.
- Most action responses use `success`; errors can use `error`, `message`, or
  `errors`. Internal failures generally return HTTP `500`. Inspect both HTTP
  status and endpoint-specific response fields.
- `success: true` is not proof that a tab opened, a document loaded, a server
  became ready, or content validated. Check the fetched contract and result.
- A connected bridge is not permission to mutate state. Perform only actions
  the user requested. Keep secrets out of documentation, logs, and responses.

## Workflow skills

| Need                            | Skill                 |
| ------------------------------- | --------------------- |
| Create a local topic            | `create-topic`        |
| Read/update a topic description | `topic-context`       |
| Find/open a local topic         | `search-local-topics` |
| Read/edit document notes        | `document-notes`      |
| Author/publish a File City tour | `file-city-tours`     |
