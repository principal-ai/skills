---
name: document-notes
description: Read or write user-authored notes anchored to specific text inside a markdown document. Notes live on the running electron-app's Principal MCP Bridge and surface as inline highlights in the MarkdownPanel. Use when the user says "what notes are on X.md", "list my notes for this doc", "leave a note on Y that says…", "what did I write about…", or invokes /document-notes. NOT for creating sequence-diagram annotations — use file-city-sequence for those.
---

# Document Notes

User-authored notes anchored to a span of text inside a markdown file. Each note has a W3C text-quote anchor (`exact` text plus optional `prefix`/`suffix` for disambiguation) and a markdown `body`. Notes are stored on disk under the electron-app's `userData/document-notes/` and exposed over HTTP through the Principal MCP Bridge so agents can read what the user has flagged or attach commentary that will show up the next time the user opens the doc.

## When to fire

Fire on phrases like:

- "what notes are on `<file>`"
- "show me my notes for this doc / for `<file>`"
- "leave a note on `<text>` that says…"
- "annotate this section with…"
- "what did I write about `<X>` in the docs"
- "list every doc I've taken notes on"
- explicit `/document-notes` invocation

Don't fire when the user wants:

- A sequence diagram annotation — that's `file-city-sequence` / `file-city-review`.
- Edits to the markdown source itself — open the file and use the regular edit tools. Notes are an overlay, not a replacement for editing the document.

## Prerequisites

The electron-app must be running. Endpoints live on the Principal MCP Bridge port:

```
http://localhost:3044
```

Confirm with `curl -s http://localhost:3044/health` before any call. If it's down, ask the user to launch the app rather than guessing other ports.

The user does not need to have the markdown panel open to read or write — notes are persisted server-side. They'll appear as inline highlights the next time they open the doc.

## Anchor model — text-quote selectors

```ts
interface TextQuoteAnchor {
  exact: string;       // the literal text being annotated
  prefix?: string;     // ~32 chars immediately before exact (disambiguator)
  suffix?: string;     // ~32 chars immediately after exact (disambiguator)
}
```

Picking a good anchor:

- **`exact` must stay inside a single block** — one paragraph, one heading, one list item. Anchors that span paragraph breaks or headings currently trigger a renderer bug and get filtered out by the panel (`MarkdownPanel.tsx:isSafeAnchor`). If your `exact` contains a blank line (`\n\s*\n`), shorten it.
- Keep `exact` short — a phrase, a sentence at most. Long quotes are brittle and harder to disambiguate.
- Add `prefix` and `suffix` (~32 chars each) when the same `exact` text appears more than once in the doc. Without them, the first match wins.
- Don't include leading/trailing whitespace in `exact` — strip it.

### Reading the doc to pick an anchor

Before posting, `Read` the markdown file (or `cat` it). Locate the exact substring the user wants annotated, copy it verbatim into `exact`, and grab the surrounding text for `prefix`/`suffix`. Don't paraphrase — the anchor must match the live document character-for-character.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/document-notes?repositoryPath=<abs>&relativeFilePath=<rel>` | List notes for a single document. Returns `{ success, repositoryPath, relativeFilePath, notes }`. |
| `GET` | `/api/document-notes/library?repositoryPath=<abs>` | List every file-with-notes (optionally filtered by repo). Returns `{ success, entries: [{ repositoryPath, relativeFilePath, fileHash, noteCount, updatedAt, sizeBytes }] }`. |
| `POST` | `/api/document-notes` | Create a note. Body: `{ repositoryPath?, relativeFilePath, anchor, body, author? }`. |
| `PATCH` | `/api/document-notes/:noteId` | Update a note's body. Body: `{ repositoryPath?, relativeFilePath, body }`. |
| `DELETE` | `/api/document-notes/:noteId?repositoryPath=<abs>&relativeFilePath=<rel>` | Delete a note. |

Notes are keyed by `(repositoryPath, relativeFilePath)`. When `repositoryPath` is omitted (or unknown), notes land in a "repo-agnostic" bucket keyed by the absolute file path; portable across machines they aren't, but they still work.

## Authoring workflow

### 1. Resolve the doc's repo and relative path

For a path the user gave you (`docs/foo.md`, `/Users/me/.../docs/foo.md`):

```bash
# Absolute repo root for the file
git -C "$(dirname <abs-path>)" rev-parse --show-toplevel
```

That's `repositoryPath`. `relativeFilePath` is the path relative to that root (e.g., `docs/foo.md`). If the file isn't inside a git repo, omit `repositoryPath` and use the absolute path as `relativeFilePath`.

### 2. Read existing notes first

When the user asks "what's noted on X" or before adding a note, fetch what's already there so you don't duplicate or contradict prior context:

```bash
curl -s "http://localhost:3044/api/document-notes?repositoryPath=$REPO&relativeFilePath=$REL" | jq
```

Each returned note has shape:

```ts
{
  id: 'note-<uuid>',
  anchor: { exact, prefix?, suffix? },
  metadata: {
    body: string,           // markdown
    author?: string,
    createdAt: string,      // ISO
    updatedAt: string,      // ISO
  }
}
```

When summarizing notes for the user, surface `metadata.body` and `metadata.author` (truncated if very long), plus a quote of the `anchor.exact` so they know which span the note attaches to.

### 3. Compose a note

```json
{
  "repositoryPath": "/Users/griever/Developer/desktop-app/electron-app",
  "relativeFilePath": "docs/file-city-sequence-diagram-persistence.md",
  "anchor": {
    "exact": "Today, payloads delivered to localhost",
    "prefix": "Background\n\n",
    "suffix": ":3044 are persisted in"
  },
  "body": "Worth noting: this section predates the cap-eviction logic added in 0.5.49.",
  "author": "Claude (claude-opus-4-7)"
}
```

Field guidance:

- `anchor.exact` — copy the literal substring from the file. Single-block constraint above.
- `body` — markdown. Be concrete; agent-authored notes are most useful when they explain *why* a passage matters, point at a related file, or flag a gotcha. Avoid restating what the doc already says.
- `author` — when an agent posts, **set this explicitly** with a string the user can recognize as agent-authored (e.g., `Claude (claude-opus-4-7)`). If you omit it, the server falls back to the local `git config` `user.name` / `user.email`, which would impersonate the user. Don't impersonate.

POST it:

```bash
curl -s -X POST http://localhost:3044/api/document-notes \
  -H 'content-type: application/json' \
  -d @note.json | jq
```

Successful response:

```json
{ "success": true, "note": { "id": "note-…", "anchor": {…}, "metadata": {…} } }
```

Capture the `id` if you intend to update or delete the note later.

### 4. Update an existing note

Only `body` is mutable via the API; the anchor is fixed at create time.

```bash
curl -s -X PATCH "http://localhost:3044/api/document-notes/$NOTE_ID" \
  -H 'content-type: application/json' \
  -d "{\"repositoryPath\":\"$REPO\",\"relativeFilePath\":\"$REL\",\"body\":\"<new body>\"}" | jq
```

If you need to re-anchor a note (e.g., the doc was edited and the `exact` text moved), delete the old one and create a fresh note.

### 5. Delete a note

```bash
curl -s -X DELETE \
  "http://localhost:3044/api/document-notes/$NOTE_ID?repositoryPath=$REPO&relativeFilePath=$REL" | jq
```

Returns `{ success: true }` when removed; `404` if the id was unknown for that file.

## Library discovery

To see what the user has annotated across the project:

```bash
curl -s "http://localhost:3044/api/document-notes/library?repositoryPath=$REPO" | jq
```

Returns a sorted list (most-recent first) of every file with at least one note. Useful when the user asks "where have I taken notes" or before a sweep that might re-organize docs (so you don't orphan their notes).

Omit `repositoryPath` to list every file in every repo plus the repo-agnostic bucket — only do this when the user explicitly asks across-projects.

## Path discipline

- `repositoryPath` is **absolute**. Use `git rev-parse --show-toplevel`.
- `relativeFilePath` is **relative to that repo path** (e.g., `docs/foo.md`, not `/Users/.../docs/foo.md`).
- The notes for a file in a working copy and in a clone of the same repo at a different path don't share storage — keying is by `(absolute repo path, relative file path)` from the machine's perspective. This matches what the panel sees at runtime.
- Renaming a file orphans its notes (accepted v1 tradeoff). If the user asks you to rename a file that has notes, surface the loss before doing it.

## Validation rules

The route rejects payloads that violate these — pre-check before posting:

- `relativeFilePath` is a non-empty string on every endpoint that takes one.
- `body` is a non-empty string on `POST`; a string (possibly empty) on `PATCH`.
- `anchor` is an object with a non-empty `exact` string. `prefix` and `suffix`, when present, are strings.
- `repositoryPath`, `author` — strings when present.

## Don't impersonate the user

The server fills `author` from `git config user.name` / `user.email` when the field is missing. That's intended for the human typing into the panel, not for agents. **Always set `author` explicitly** when posting from an agent context, with a value the user can recognize. Otherwise the user later sees what looks like one of their own notes appear with content they didn't write — confusing at best, misleading at worst.

## When things go wrong

| Symptom | Likely cause |
|---|---|
| `curl: (7) Failed to connect to localhost port 3044` | App isn't running. |
| `400` with `relativeFilePath is required` | Forgot the field, or sent `repositoryPath` without `relativeFilePath`. |
| `400` with `body must be a non-empty string` | Body is empty / missing. PATCH allows empty strings; POST does not. |
| `400` with `anchor.exact must be a non-empty string` | Anchor missing or `exact` empty. |
| `404` on PATCH/DELETE | The `noteId` doesn't exist for the `(repositoryPath, relativeFilePath)` you passed. Check the file key — a note id only resolves under the file it was created on. |
| Note created but doesn't show in the panel | The anchor's `exact` doesn't appear in the live doc (whitespace mismatch, the file was edited, or the prefix/suffix don't disambiguate). The note still exists on disk; reading the file and adjusting the anchor (delete + recreate) fixes it. |
| Note created, panel reload crashes | Anchor `exact` spans block boundaries (contains `\n\s*\n`). The panel filters these out so it won't crash on the next reload, but the highlight won't render. Delete and recreate with a single-block anchor. |
| Library list returns absolute path as `relativeFilePath` | Note was created without `repositoryPath` (repo-agnostic bucket). Recreate under the right repo if you want it grouped. |

## Reference

- Sister skill for runtime/architecture flows: `file-city-sequence`
- Sister skill for PR diff walkthroughs: `file-city-review`
- Persistence implementation: `electron-app/src/main/document-notes/documentNotesPersistence.ts`
- HTTP routes: `electron-app/src/main/document-notes/documentNotesRoutes.ts`
- Panel wiring: `electron-app/src/renderer/panels/markdown-panel/MarkdownPanel.tsx`
