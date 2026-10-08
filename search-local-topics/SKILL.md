---
name: search-local-topics
description: List or search locally stored topics and open a selected topic in the running electron-app. Uses GET /api/topics?q=… and POST /api/topics/:id/activate on the local Principal MCP Bridge. Use when the user asks which local topics exist, wants to find a topic by title or description, or asks to open a topic. NOT for creating topics (use create-topic), or reading/updating a specific topic description (use topic-context).
---

# Search Local Topics

Search the local topics managed by the running electron-app and, when asked,
open a selected topic in the app. A topic is a brief scoped to repositories;
it is not a bundle of trails.

## When to use

- The user asks what local topics exist.
- The user asks to find a topic by subject, title, or description.
- The user asks to open a local topic in the app.

Use `create-topic` to create a new topic. Use `topic-context` to read or update
the brief of a topic once identified.

## Prerequisite

The electron-app must be running. Its local Principal MCP Bridge listens on
port `3044`. Check it before querying:

```bash
curl -s http://localhost:3044/health
```

If unavailable, ask the user to launch the app; don't guess another port. The
bridge is local and unauthenticated.

## List and search

The `GET /api/topics` endpoint returns lightweight topic summaries, sorted by
most recently updated. Its `q` parameter filters title and description
case-insensitively:

```bash
curl -s http://localhost:3044/api/topics \
  | jq '.topics[] | {id, title, descriptionPreview, state, updatedAt}'

curl -s "http://localhost:3044/api/topics?q=auth" \
  | jq '.topics[] | {id, title, descriptionPreview, state, updatedAt}'
```

The directory entries include `id`, `href`, `title`, `descriptionPreview`,
`state`, `createdAt`, and `updatedAt`; they don't include the full description.
If several topics match, show a shortlist with ids and titles and let the user
choose. Fetch the selected topic with `GET /api/topics/:id` before summarizing
its full brief.

## Open a topic

After identifying the requested topic, open it in the running app:

```bash
curl -s -X POST "http://localhost:3044/api/topics/$TOPIC_ID/activate"
```

The response reports whether the topic was delivered to the focused app window.
This action only opens the topic; it does not edit it.

## Limits

- This skill searches local topics only; it does not browse published
  web-ade topics.
- The list endpoint searches title and description text, not repository paths
  or file contents.
- It does not create or edit topics. Use `create-topic` or `topic-context` for
  those operations.

## References

- Topic HTTP routes: `electron-app/src/main/topics/topicRoutes.ts`
- Sibling skills: `create-topic` (create), `topic-context` (read/update).
