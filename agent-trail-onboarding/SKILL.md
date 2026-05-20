---
name: agent-trail-onboarding
description: Explain Principal trails and topics to a user who is new to the system or unsure which surface they need — what a trail is, what a topic is, shared (web-ade) vs local trails, that there is a `principal-ai` CLI for agent-side work, that trails and topics are gated by GitHub repo permissions by default, and that there is an optional desktop app for richer File City visualization (the CLI also ships with a minimal Electrobun UI for users who don't want the full app). Read-only conceptual onboarding; ends by pointing at the right action skill. Use when the user says "what's a trail?", "what's a topic?", "what's the difference between a shared trail and a local trail?", "do I need the app?", "is there a CLI?", "how do permissions work?", "I'm new to this — where do I start?", "help me pick the right skill", "what can I do with Principal?", or invokes /agent-trail-onboarding. NOT for actually authoring, publishing, or fetching anything — explain, then hand off to author-*-trail, publish-trail, create-topic, discover-trails, or file-city-trail-review.
---

# Agent Trail Onboarding

A read-only orientation skill. The user is new to Principal trails (or
unsure which surface they need), and they're asking questions. Explain
the model — trails, topics, shared vs local, the CLI, permissions, and
the optional desktop app — then hand off to the right action skill.

This skill **never authors, publishes, or fetches**. It is purely
explanatory. The moment the user knows what they want to do, route them
to the matching skill.

## When to fire

Fire on phrases like:

- "What's a trail?" / "what is a code trail?"
- "What's a topic?" / "what does curating a topic mean?"
- "What's the difference between a shared trail and a local one?"
- "Do I need the desktop app?" / "is there a CLI?"
- "How do permissions work?" / "who can see my trails?"
- "I'm new to this — where do I start?"
- "Help me figure out which skill to use."
- "What can I do with Principal?"
- Explicit `/agent-trail-onboarding` invocation.

Don't fire when:

- The user already knows what they want to do — go straight to the
  matching action skill (see "Hand-off" below).
- The user is asking about the *implementation* of trails (payload
  shape, S3 layout, marker schema) — that's a code question for the
  repo, not an onboarding question.

## The model in one paragraph

A **trail** is a guided walk through a codebase: an ordered list of
markers, each pinned to a file + line range, with a title and a blurb.
Trails come in two flavors — **informative** (the durable, canonical
version of a flow or insight; "this is what is true about the code now")
and **investigation** (the exploratory version you lay as you figure
something out; allowed to carry exploratory titles, a "subject" marker
pointing at the answer, and a record of how the answer was found). A
**topic** is a curated collection of existing trails on a shared
subject — usually the same conceptual problem solved across different
repos — rendered as a single page so readers can compare.

## Shared trails vs local trails

This is the distinction that confuses users most.

| | **Shared trail (web-ade)** | **Local trail** |
|---|---|---|
| Where it lives | `https://app.principal-ade.com/trail/<id>` | A JSON file on disk |
| Who can see it | Anyone with the link **plus** read access to the underlying repo | Only the user (and whoever they hand the file to) |
| Auth needed | GitHub token (`gh auth token` or git credential helper) | None |
| Best for | Sharing with teammates, comparing across repos via a topic | Onboarding yourself, scratch investigations, working offline |
| How to author | `author-informative-trail` / `author-investigation-trail` (in-app) **or** `publish-trail` (CLI-only, no app) | `author-local-informative-trail` / `author-local-investigation-trail` |

A **local trail can be promoted to a shared trail** later via
`publish-trail`. The reverse doesn't really happen — once published,
the trail's home is web-ade.

## Tooling: app vs CLI vs minimal UI

There are three ways to interact with trails. Pick whichever fits how
the user is working.

1. **The desktop app (electron-app).** Full File City — the 2D file
   tree visualization where each square is a file. Trails light up
   buildings, snippets open in Pierre drawers, sequence diagrams,
   trail review for PRs, notes anchored to markdown. Best experience,
   but it's a download. Skills that need it: `author-informative-trail`,
   `author-investigation-trail`, `convert-investigation`,
   `file-city-trail-review`, `document-notes`.

2. **The `principal-ai` CLI.** Lets an agent author, publish, curate,
   and discover trails from the terminal — no app required. Resolves
   the user's GitHub token from `gh auth token` first, then falls back
   to `git credential fill` for `github.com`. Runs via npx so nothing
   has to be installed up front:

   ```bash
   npx -y @principal-ai/principal-view-cli@latest trail --help
   ```

   Skills that use it: `publish-trail`, `create-topic`,
   `discover-trails`, `author-local-informative-trail`,
   `author-local-investigation-trail`.

3. **The minimal Electrobun viewer that ships alongside the CLI.**
   When a local trail is authored or fetched, the CLI can open it in
   a tiny Electrobun viewer:

   ```bash
   principal-ai trail view --file <path>
   ```

   This exists specifically for users who don't want to download the
   full desktop app — they get a visual viewer, just without the full
   File City and panel ecosystem. The viewer is the
   `@principal-ai/trail-viewer` package, installed as an optional
   dependency of the CLI; today only macOS arm64 prebuilds ship, so
   on other platforms the user either points at a source checkout
   with `--viewer-dir <path>` or falls back to the desktop app.
   Local authoring skills land here by default.

The right way to frame this to a user: **start with the CLI**; if they
hit a workflow that wants the richer File City (PR review, sequence
diagrams, notes), suggest the desktop app then.

## Permissions

Trails and topics are **gated by GitHub repo permissions by default**.
The rules:

- **Publish.** To publish a trail for a repo, the authenticated user
  must have write access (or appropriate role) on that repo via their
  GitHub token. The publish endpoint checks this server-side.
- **Read a shared trail.** The trail's *metadata* (title, who
  published it, when) is public via the share link — anyone can list
  it. But the **payload** (markers, snippets, file content) self-gates
  on read: the server checks the caller's GitHub token against the
  underlying repo, and returns 403 if they don't have access.
- **Topics.** Curated by their owner; the topic page is public, but
  each trail inside still applies its own per-trail repo-access check
  when the reader clicks through.
- **Local trails.** No gating — they're just files on disk.

Net effect: a user can safely share a trail link in public (Slack, a
tweet, a PR description) without leaking private repo contents.
Someone without repo access gets a "you don't have access" page, not
the snippets.

## Hand-off — pick the matching skill

Once the user knows what they want, route them. Don't keep explaining.

| The user wants to… | Use |
|---|---|
| Author a durable, canonical trail (in the desktop app) | `author-informative-trail` |
| Lay an exploratory trail as they figure something out (in the app) | `author-investigation-trail` |
| Same as above, but locally without the app | `author-local-informative-trail` / `author-local-investigation-trail` |
| Publish a local trail to web-ade as a shareable link | `publish-trail` |
| Curate a topic from existing trails | `create-topic` |
| Browse / list / fetch existing trails or topics | `discover-trails` |
| Walk through a PR or branch diff visually | `file-city-trail-review` |
| Turn an investigation trail into a canonical one | `convert-investigation` |

## How to deliver this

When you fire this skill:

1. **Answer the user's specific question first** — don't dump the whole
   doc. If they asked "what's a topic?", lead with the topic paragraph,
   then offer the related context they probably need next.
2. **Use the table for shared-vs-local** if they're stuck on that
   choice — it's the only part of the doc that's worth showing verbatim
   because the comparison is the value.
3. **End with a routing question.** "Which of these are you trying to
   do?" → then fire the matching skill from the table above.
4. **Don't recite the permission rules unless asked.** They matter, but
   they're not what a first-time user needs to know to get started.
5. **Don't author anything from this skill.** The instant the user
   says "ok, let's do X," hand off.
