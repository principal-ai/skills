---
name: migrate-docs-to-trails
description: Triage existing markdown documents and migrate the worthwhile ones into trails (or topics-with-trails), leaving the rest alone. Categorizes each doc along four axes — is it anchorable to code that exists now, one flow or many, canonical-truth vs how-it-was-found, and how badly it has drifted — then audits drift by re-resolving every file reference, dedupes against trails that already exist, and routes to the right authoring skill. Use when the user says "migrate this doc to a trail", "should this markdown become a trail or a topic", "triage our docs folder for trail conversion", "audit this doc and convert it", or invokes /migrate-docs-to-trails. NOT for authoring a trail from scratch with no source doc — use author-investigation-trail / author-informative-trail. NOT for forking an existing trail — use convert-investigation.
---

# Migrate Docs to Trails

Take a markdown document (or a whole `docs/` folder) and decide, per doc,
whether it should become **a trail**, **a topic that owns several trails**,
or be **left alone / archived** — then carry the worthwhile ones across.

The hard part isn't the authoring (the `author-*` skills already do that).
The hard part is **triage**: most docs should not become trails, and the
ones that should are usually stale. This skill is the categorize → audit →
dedupe → route pipeline that sits in front of the authoring skills.

## When to fire

Fire on phrases like:

- "migrate this doc to a trail" / "convert this markdown into a trail"
- "should this doc be a trail or a topic?"
- "triage our docs folder for trail conversion"
- "audit this doc against the code and convert it"
- explicit `/migrate-docs-to-trails` invocation

Don't fire when:

- The user wants to author a trail from scratch with no source document —
  use `author-investigation-trail` (exploratory) / `author-informative-trail`
  (canonical), or the `author-local-*` siblings.
- The user wants to fork an existing **trail** into another trail — use
  `convert-investigation`.
- The user wants notes anchored to a doc rather than a trail — use
  `document-notes`.

## The four triage axes

Every decision reduces to four questions. Answer them in order; the first
that disqualifies a doc stops the pipeline.

**1. Anchorability — does it describe real, current code in THIS repo?**
A trail's value is markers pinned to `file:line`. If the doc can't be
anchored, it can't be a trail.
- External / domain knowledge (e.g. a medical-billing research note) →
  **LEAVE ALONE**. Nothing to pin.
- Aspirational / future work (`*_PLAN.md`, roadmaps) → the code doesn't
  exist yet → **TOPIC** (a tracked intent), not a trail.
- Describes this codebase → continue.

**2. Cardinality — one walkable sequence, or many?**
- A single linear flow (A → B → C, one hop per step) → **ONE TRAIL**.
- A broad subject with several independent flows → **TOPIC + several
  trails**, one trail per flow.

**3. Stance — canonical truth, or a record of how it was found?**
- "This is how it works now" → **informative** trail.
- "Here's what we investigated / decided / audited" (titles like *Audit*,
  *Design*, *Proposal*) → **investigation** trail, optionally promoted to
  informative later via `convert-investigation`.

**4. Drift — do its references still resolve?** (decides effort, not bucket)
- Mostly resolves → convert directly.
- Partial → convert **with** a re-resolution pass (see Step 2).
- 0% resolves → **ARCHIVE / leave alone**; the code it describes is gone.

### Decision flow

```
Describes real, current code in THIS repo?
 ├─ No, external/domain knowledge ............... LEAVE ALONE
 ├─ No, future/aspirational ..................... TOPIC (intent, no trail yet)
 └─ Yes
     ├─ All references dead (0% resolve) ........ ARCHIVE / leave alone
     ├─ One linear flow ......................... ONE TRAIL
     │     └─ "how it works now" → informative
     │        "how we found/decided" → investigation
     └─ Multiple flows / broad subject .......... TOPIC + several trails
```

## Workflow

### Step 0 — Scope

Single doc, or a folder sweep? For a folder, run Step 1's drift scan across
every `*.md` first to produce a ranked worklist, then process docs
high-resolution-first (cleanest conversions build confidence in the rubric
before you hit the heavily-drifted ones).

### Step 1 — Drift audit (anchorability axes 1 + 4, measured)

Don't eyeball it — measure. Extract every code reference the doc cites and
re-resolve each against the current tree. The cheap, high-signal version is
per-referenced-basename:

```bash
DOC=docs/repository-monitoring-git-watching.md
total=0; alive=0
for ref in $(grep -oE '[A-Za-z0-9_-]+\.(ts|tsx|js|jsx)' "$DOC" | sort -u); do
  total=$((total+1))
  if find src -name "$ref" -not -path "*/node_modules/*" 2>/dev/null | grep -q .; then
    alive=$((alive+1))
  else
    echo "DEAD: $ref"
  fi
done
echo "$DOC: $alive/$total references still exist"
```

For docs that already carry `file:line` (the best conversion candidates),
also check the *path* resolves, not just the basename — files move between
directories, so a surviving basename at a new path still means the doc's
path is stale and the marker needs re-anchoring.

- **0/total** → ARCHIVE candidate. Don't invent a trail for dead code —
  but before deleting, run the **archive → live-coverage** check below.
  Confirm the components are *gone*, not just *moved*, by grepping for the
  doc's named identifiers (classes/services), not only filenames:
  `grep -rIl --include='*.ts' --include='*.tsx' -- '<Identifier>' src`.
  Zero hits across all of them = the architecture was replaced, not relocated.
- **partial** → note each DEAD ref; you'll re-discover its current home in
  Step 4 (`find src -name <basename>` → confirm it's the same component).
- **most/all** → straight conversion.

Record the score — surface it to the user so the whole folder can be
triaged at a glance.

### Step 2 — Categorize

Apply the four axes to land the doc in exactly one bucket: LEAVE ALONE,
ARCHIVE, ONE TRAIL (informative|investigation), or TOPIC (+trails). State
the bucket and the one-line reason before doing any authoring.

### Step 3 — Dedupe against existing trails

Before authoring, check whether a trail already covers this flow. **There
is no full-text or file-overlap search yet** (see "Search limitations"),
so use the per-user listing surface from `discover-trails`:

```bash
# List trails the current user has published, keep those in this repo,
# keyword-match the doc's subject against title + summaryPreview.
# NOTE: the CLI prints its JSON envelope to STDERR — pipe 2>&1 (piping
# only stdout drops the payload and jq sees nothing). Match on title +
# summaryPreview only; repoNames is often [] on published entries, so
# don't gate on it — confirm the repo by reading owner/repo instead.
npx -y @principal-ai/principal-view-cli@latest trail list 2>&1 \
  | jq --arg kw 'monitor|git.status|file.tree|repository.watch' '.entries[]?
      | select((.title + " " + .summaryPreview) | test($kw; "i"))
      | {id, title, summaryPreview, markerCount, owner, repo}'
```

For any near-match, fetch it and compare its markers' `sourcePath` to the
doc's referenced files — overlap means the flow is already trailed:

```bash
npx -y @principal-ai/principal-view-cli@latest trail <id> \
  | jq -r '.markers[].sourcePath' | sort -u
```

- Strong overlap → don't duplicate. Offer to **update** the existing trail
  (re-anchor drifted markers) or to fold the doc in as added context.
- Several users' trails touch the same subject → suggest curating a
  **topic** via `create-topic` instead of a new standalone trail.
- No match → proceed to author.

> Run dedupe with the user's GitHub token reachable (`gh auth token` or git
> credential helper), exactly as `discover-trails` documents. No token →
> the `list` call can't resolve identity; say so and skip dedupe rather
> than guessing.

### Step 4 — Author (hand off, don't re-implement the schema)

Re-anchor every DEAD reference from Step 1 (`find` the basename, read the
file, pick the tight line window), then route by bucket:

| Bucket | Author with |
| --- | --- |
| ONE TRAIL — informative | `author-informative-trail` (in-app) / `author-local-informative-trail` (no app) |
| ONE TRAIL — investigation | `author-investigation-trail` / `author-local-investigation-trail` |
| TOPIC + trails | author each flow as a trail, then `create-topic` to group them |
| TOPIC — intent only | `create-topic`; no trail until the code lands |
| LEAVE ALONE | nothing — report the decision |
| ARCHIVE | run the archive → live-coverage check below, then delete |

The electron-app must be running for the in-app authoring routes
(`http://localhost:3044`); confirm with `curl -s http://localhost:3044/health`.
Map the doc's structure onto the trail: each flow step → one marker with a
tight `file:line` slice; the doc's prose → the marker `description` and the
trail `summary`. An *Audit*/*Design* doc keeps its exploratory framing as an
**investigation** trail; promote later with `convert-investigation`.

### Step 4a — Archive → live-coverage check

A doc archives because the *code it documented* is gone — but the
*functionality* often still exists, rewritten under a new design. Don't
just delete and move on: the rewritten subsystem is usually the thing most
worth a trail, and there's now a stale-doc-shaped hole where its
documentation used to be.

So when a doc lands in ARCHIVE:

1. **Locate the live replacement.** The subsystem name usually survives
   even when the classes don't — `find src -type d -iname '*<subsystem>*'`
   and grep for the current entry points.
2. **Verify whether a trail already covers it** using the Step 3 dedupe
   surface (keyword-match the subsystem against published trails). Remember
   `repoNames` is often empty — match on title/summary and confirm via
   `owner/repo`.
3. **Decide:**
   - A trail already covers the live functionality → delete the doc; note
     the trail id as its replacement. Done.
   - No trail covers it → **offer to author a fresh trail** through the
     current implementation (`author-investigation-trail` /
     `author-informative-trail`). This turns "delete a dead doc" into
     "replace dead prose with a living trail" — the higher-value outcome.
   - User declines the new trail → delete the doc and report that the live
     functionality is currently untrailed, so the gap is visible.

Only delete the source doc after this check — a git-tracked delete is
recoverable, but the point is to not silently drop the only pointer to a
subsystem that still ships.

### Step 5 — Report + provenance

For each doc processed, report: bucket, drift score, dedupe result, and
(if authored) the new trail/topic id. Note in the trail summary that it was
migrated from `<doc path>` so the lineage is traceable. Recommend whether
the source doc should now be deleted, kept as prose, or marked superseded.

## Search limitations (read before Step 3)

The dedupe step is constrained by what the read side supports today:

- **No full-text search.** `discover-trails` lists only summaries
  (`summaryPreview`); marker bodies aren't indexed. Deep matches require
  fetching by id and grepping client-side.
- **No file-overlap query** — the signal migration actually wants ("which
  trails touch these files?") doesn't exist. Compare `sourcePath` lists by
  hand after fetching candidates.
- **No global feed.** Discovery is per-user; you can only dedupe against
  users you know to list (`--user <login>`).
- **Local bridge can't enumerate.** The running app exposes no list/library
  route over the bridge — only (at most) the currently-active payload. Don't
  rely on the local bridge for dedupe; use the web-ade CLI listing.

When dedupe is necessarily incomplete (e.g. you only listed your own
trails), **say so** — report it as "checked my published trails; a flow
already trailed by another user wouldn't have been caught."

## What this skill doesn't do

- It doesn't author trails itself — it triages and routes. The `author-*`,
  `create-topic`, and `convert-investigation` skills do the writing.
- It doesn't mutate or delete the source doc — it recommends; the user
  decides.
- It doesn't fabricate markers for dead code — a 0%-resolving doc is
  archived, not trailed.
