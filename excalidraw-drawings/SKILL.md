---
name: excalidraw-drawings
description: Locate and edit the Excalidraw drawings created in the electron-app's Drawings view. Drawings are plain `.excalidraw` JSON files stored under `~/.alexandria/drawings/`, one file per drawing. Use when the user shares a drawing path (e.g. from the Drawings view "Copy file path" button) and asks you to edit, fix, annotate, restructure, or add to a diagram, or says "edit my drawing", "change this excalidraw", "add a box/arrow/label to the diagram", or invokes /excalidraw-drawings. NOT for laying File City trails or sequence diagrams — those are separate trail skills.
---

# Excalidraw drawings — find & edit

The electron-app's **Drawings** view (left-nav, principal window) lets the user
create and edit [Excalidraw](https://excalidraw.com) diagrams. Each drawing is a
plain JSON file on disk, so an agent can read and edit one directly with the
normal file tools — no app or bridge required.

This skill covers **where the files live** and **how to edit them safely** so
your changes load cleanly the next time the user opens the drawing.

## Where drawings are stored

```
~/.alexandria/drawings/<id>.excalidraw
```

- One file per drawing. `<id>` is a UUID (or a timestamp for older drawings) —
  it is **not** the display name.
- The directory is global and repo-free; there is no per-project or per-topic
  scoping yet.
- The file extension is always `.excalidraw`. The contents are JSON.

The user usually hands you the absolute path via the **"Copy file path"** button
on each drawing card in the Drawings view (the copy icon next to the trash
icon). Prefer that exact path over guessing — the display name is not the
filename.

To list what exists when you only have a name:

```bash
ls -t ~/.alexandria/drawings/*.excalidraw
# then grep for the display name, which lives in appState.name:
grep -l '"name": "My Diagram"' ~/.alexandria/drawings/*.excalidraw
```

## File format

A drawing is the standard Excalidraw scene JSON. Top-level keys:

| Key            | What it is                                                        |
| -------------- | ----------------------------------------------------------------- |
| `elements`     | Array of scene elements (rectangles, arrows, text, …). The diagram. |
| `appState`     | Editor state. `appState.name` holds the **display name**.         |
| `files`        | Map of embedded binary assets (images), keyed by file id. Often `{}`. |
| `libraryItems` | Saved library shapes. Often `[]`.                                 |
| `type`         | Always `"excalidraw"`.                                             |
| `version`      | Scene schema version (currently `2`).                             |
| `source`       | Origin tag, e.g. `"excalidraw-panel"`.                            |

The app reads the display name from `appState.name` — if you rename a drawing,
change it there (not the filename).

### Element shape (the part you'll actually edit)

Every entry in `elements` is an object. Common fields:

```jsonc
{
  "id": "jmfSYgePFIDEEIlHhZUU2",   // unique string id, referenced by bindings
  "type": "rectangle",              // rectangle | ellipse | diamond | arrow | line | text | freedraw | image | frame
  "x": 238.5, "y": 256.6,           // top-left position on the canvas
  "width": 289.9, "height": 128.0,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",           // solid | dashed | dotted
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "index": "a0",                    // fractional z-order key; must stay ordered
  "roundness": { "type": 3 },
  "seed": 141786169,
  "version": 20,                    // bump on edit
  "versionNonce": 2008955129,
  "isDeleted": false,               // soft-delete flag; set true to hide
  "boundElements": null,            // arrows/labels bound to this element
  "updated": 1781286998158,         // epoch ms
  "link": null,
  "locked": false
}
```

`text` elements additionally carry `text`, `fontSize`, `fontFamily`,
`textAlign`, `verticalAlign`, and `containerId` (set when the text is bound
inside a shape). `arrow`/`line` elements carry `points` (an array of `[x, y]`
offsets from the element's `x`/`y`) and `startBinding`/`endBinding` linking them
to other elements by `id`.

## Editing rules

When you modify a drawing, keep it loadable:

1. **Edit a copy of the parsed JSON, write valid JSON back.** Preserve the
   2-space pretty formatting the app writes — it keeps diffs readable.
2. **Give new elements a unique `id`** (any unique string). Anything that
   references an element — `boundElements`, `startBinding`/`endBinding`,
   `containerId`, `groupIds` — must use these ids consistently.
3. **Set `index` so z-order stays sorted.** `index` values are fractional
   strings (`"a0"`, `"a1"`, `"a2"`…) sorted ascending = back-to-front. A new
   element on top can use an `index` lexicographically after the current max
   (e.g. `"a0"` → `"b0"`). Out-of-order indices make Excalidraw re-sort on load.
4. **Don't delete elements by removing them if they're referenced** — prefer
   `"isDeleted": true`, or also scrub every reference to their id. Dangling
   bindings render as broken arrows.
5. **Bump `version`** and set a fresh `versionNonce`/`updated` on elements you
   change. Not strictly required to load, but it keeps Excalidraw's internal
   reconciliation honest.
6. **Leave `appState` largely alone.** Only `appState.name` is meaningful to
   the app. Don't strip the other keys — Excalidraw expects them.
7. **Validate before saving:** `python3 -c "import json; json.load(open(PATH))"`.

### Worked example — add a labeled box

```bash
PATH=~/.alexandria/drawings/<id>.excalidraw
python3 - "$PATH" <<'PY'
import json, sys
p = sys.argv[1]
d = json.load(open(p))
maxidx = max((e.get("index","a0") for e in d["elements"]), default="a0")
box = {
  "id": "box-new-1", "type": "rectangle", "x": 600, "y": 260,
  "width": 200, "height": 100, "angle": 0,
  "strokeColor": "#1e1e1e", "backgroundColor": "transparent",
  "fillStyle": "solid", "strokeWidth": 2, "strokeStyle": "solid",
  "roughness": 1, "opacity": 100, "groupIds": [], "frameId": None,
  "index": maxidx + "V", "roundness": {"type": 3},
  "seed": 1, "version": 1, "versionNonce": 1, "isDeleted": False,
  "boundElements": None, "updated": 0, "link": None, "locked": False
}
d["elements"].append(box)
json.dump(d, open(p, "w"), indent=2)
print("added; elements:", len(d["elements"]))
PY
```

## After you edit

The Drawings view re-scans the directory on its own actions, but it does **not**
hot-reload a file changed underneath an open editor. Tell the user to **reopen
the drawing** (click another drawing and back, or reselect it from the list) to
see your edit. If they had it open and unsaved, warn them that saving in the app
will overwrite your on-disk change — coordinate on who writes last.
