# Introduction Tour Format Specification

This document defines the JSON format for creating guided tours through codebases using the File City visualization.

## Overview

Introduction tours are JSON files that guide users through a codebase step-by-step, highlighting key areas, explaining architecture, and providing interactive learning experiences.

## File Format

Tours are defined as JSON files that match the `IntroductionTour` interface.

### Basic Structure

```json
{
  "id": "unique-tour-id",
  "title": "Tour Title",
  "description": "Overview of what this tour covers",
  "version": "1.0.0",
  "repos": [/* Array of TourRepoRef (at least 1 required) */],
  "audience": "beginner",
  "prerequisites": ["Required knowledge"],
  "steps": [/* Array of tour steps */],
  "metadata": {/* Optional metadata */}
}
```

## Top-Level Fields

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Unique identifier for the tour (kebab-case recommended) |
| `title` | `string` | Human-readable tour title |
| `description` | `string` | Brief overview of what the tour covers |
| `version` | `string` | Semantic version (e.g., "1.0.0") |
| `repos` | `TourRepoRef[]` | Repo(s) the tour's directories belong to (at least 1 required). `repos[0]` is the primary repo; additional entries make a multi-repo tour. |
| `steps` | `IntroductionTourStep[]` | Array of tour steps (at least 1 required) |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `audience` | `string` | Target audience (e.g., "beginner", "New Users & AI Assistants") |
| `prerequisites` | `string[]` | List of required knowledge or setup |
| `coverImage` | `string` | Cover image URL or path relative to repo root (e.g., "assets/tour-cover.png"). Displayed as overlay on File City during welcome screen. Supports static (jpg, png, svg) and animated (gif, webp). |
| `metadata` | `object` | Additional tour metadata |

### Metadata Object

```json
{
  "metadata": {
    "author": "Author Name",
    "createdAt": "2026-02-03",
    "updatedAt": "2026-02-03",
    "tags": ["onboarding", "architecture", "tutorial"]
  }
}
```

## Repos

A tour describes directories of a codebase, so it must name the repo(s) those directories belong to. The required top-level `repos` field is an array of `TourRepoRef` entries. `repos[0]` is the primary repo; supplying more than one entry makes the tour multi-repo.

```json
"repos": [
  { "id": "pkg:github/acme/widgets", "name": "widgets", "remote": { "host": "github", "owner": "acme", "name": "widgets" }, "roots": ["packages/cli"] }
]
```

### TourRepoRef Fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `id` | `string` | Package URL identifying the repo, e.g. `pkg:github/owner/name` | **required** |
| `name` | `string` | Short repo name, e.g. `Backlog.md` | **required** |
| `remote` | `{ host, owner, name }` | Remote location. `host` is `"github" \| "gitlab" \| "bitbucket"` | - |
| `roots` | `string[]` | Repo-relative subtree paths to scope the rendered city to | `[]` (whole repo) |
| `authoredAtSha` | `string` | Commit SHA the tour was authored against | - |

`roots` keeps very large repos legible by scoping the rendered city to specific subtrees; absent or empty means the whole repo is shown. The array form of `repos` supports multi-repo tours. (Viewer rendering of `roots` and multi-repo is future work, but the data is valid now.)

## Tour Steps

Each step represents one stage of the guided tour.

### Required Step Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Unique step identifier (e.g., "step-1-overview") |
| `title` | `string` | Step title shown to user |
| `description` | `string` | Detailed explanation of this step |

### Optional Step Fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `estimatedTime` | `number` | Estimated time in seconds | - |
| `focusDirectory` | `string` | Directory path to zoom/focus on | - |
| `coverImage` | `string` | Cover image URL or path relative to repo root. Displayed as overlay during this step. Useful for diagrams, architecture visuals, or explanatory images. | - |
| `colorMode` | `ColorMode` | Visualization mode to use | `"fileTypes"` |
| `highlightLayers` | `HighlightLayerConfig[]` | Custom highlight layers | `[]` |
| `highlightFiles` | `string[]` | Specific files to highlight | `[]` |
| `interactiveActions` | `InteractiveAction[]` | User actions to try | `[]` |
| `resources` | `TourResource[]` | Related links/docs | `[]` |
| `autoAdvance` | `boolean` | Auto-advance to next step | `false` |
| `autoAdvanceDelay` | `number` | Delay before auto-advance (ms) | `3000` |

### Color Modes

Available values for `colorMode`:
- `"fileTypes"` - Color by file extension
- `"git"` - Show git status (modified, staged, untracked)
- `"pr"` - Show pull request changes
- `"commit"` - Show commit changes
- `"coverage"` - Test coverage visualization
- `"eslint"` - ESLint quality
- `"typescript"` - TypeScript quality
- `"prettier"` - Code formatting quality
- `"knip"` - Dead code detection
- `"alexandria"` - Documentation coverage

## Highlight Layers

Highlight layers draw attention to specific files or directories with colored overlays.

```json
{
  "id": "layer-id",
  "name": "Layer Name",
  "color": "#3b82f6",
  "items": [
    { "path": "src/components", "type": "directory" },
    { "path": "src/index.ts", "type": "file" }
  ],
  "opacity": 0.7,
  "borderWidth": 3,
  "renderStrategy": "border"
}
```

### HighlightLayerConfig Fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `id` | `string` | Layer identifier | **required** |
| `name` | `string` | Display name | **required** |
| `color` | `string` | Hex color code | **required** |
| `items` | `Array<{path, type}>` | Files/dirs to highlight | **required** |
| `opacity` | `number` | Opacity (0-1) | `1.0` |
| `borderWidth` | `number` | Border width in pixels | `2` |
| `renderStrategy` | `"fill" \| "border"` | How to render | `"fill"` |

### Item Types

Each item in the `items` array must specify:
- `path` - Relative path from repository root
- `type` - Either `"file"` or `"directory"`

## Interactive Actions

Interactive actions suggest tasks for the user to perform.

```json
{
  "type": "click-file",
  "description": "Click on App.tsx to see the main component",
  "target": "src/App.tsx",
  "required": false
}
```

### InteractiveAction Fields

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `type` | `ActionType` | Type of action | **required** |
| `description` | `string` | Action description | **required** |
| `target` | `string` | Target path/identifier | - |
| `required` | `boolean` | Must complete to proceed | `false` |

### Action Types

- `"click-file"` - Click on a specific file
- `"hover-directory"` - Hover over a directory
- `"toggle-layer"` - Toggle a highlight layer
- `"explore"` - Free exploration

## Resources

Resources provide links to additional documentation or context.

```json
{
  "title": "Component Documentation",
  "url": "https://docs.example.com/components",
  "type": "documentation"
}
```

### TourResource Fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | `string` | Resource title |
| `url` | `string` | URL or file path |
| `type` | `"documentation" \| "video" \| "article" \| "code"` | Resource type |

## Validation Rules

### Tour-Level Validation

1. ✅ `id` must be unique within the codebase
2. ✅ `version` must follow semantic versioning (X.Y.Z)
3. ✅ `repos` must contain at least 1 `TourRepoRef`, each with `id` and `name`
4. ✅ `steps` array must contain at least 1 step
5. ✅ All step IDs must be unique within the tour

### Step-Level Validation

1. ✅ `estimatedTime` must be positive if specified
2. ✅ `focusDirectory` must be a valid directory path
3. ✅ `colorMode` must be a supported color mode
4. ✅ `autoAdvanceDelay` must be >= 1000 (1 second) if auto-advance enabled
5. ✅ Paths in `highlightFiles` must be valid file paths
6. ✅ Paths in `focusDirectory` must not start with `/` (relative paths only)

### Highlight Layer Validation

1. ✅ `color` must be a valid hex color (#RRGGBB or #RGB)
2. ✅ `opacity` must be between 0 and 1
3. ✅ `borderWidth` must be positive
4. ✅ `items` array must not be empty
5. ✅ Each item path must be a valid repository path

### Interactive Action Validation

1. ✅ `target` is required for `click-file`, `hover-directory`, and `toggle-layer` actions
2. ✅ `target` should reference an existing file/directory path
3. ✅ `target` for `toggle-layer` should reference a layer ID in `highlightLayers`

## Best Practices

### Tour Design

1. **Target duration: 2 minutes ideal, 3 minutes maximum** - Keep tours concise and impactful:
   - **4-6 steps** for 2-minute tours (recommended)
   - **6-8 steps maximum** for 3-minute tours (absolute max)
   - **Keep each tour cohesive** - One subject per tour; don't sprawl across unrelated areas
   - **Test and time** - Walk through your tour and verify it fits within 3 minutes
2. **Start broad, then narrow** - Begin with overview, then focus on specific areas
3. **Estimate time accurately** - Use `estimatedTime` field, allocate 20-30 seconds per step
4. **Use progressive disclosure** - Don't overwhelm with too much information at once

### Step Design

1. **One concept per step** - Each step should teach one main idea
2. **Keep descriptions concise** - Aim for 200-250 characters, maximum 300. Focus on key points users need to know.
3. **Allocate 20-30 seconds per step** - Include time for reading description, viewing visualization, and interaction
4. **Total text budget** - For 2-minute tours: 800-1,500 characters across all steps. For 3-minute max: up to 2,000 characters.
5. **Use visual hierarchy** - Combine `focusDirectory` with `highlightLayers` for clarity
6. **Make it interactive** - Include at least one `interactiveAction` per step
7. **Provide resources** - Link to relevant documentation

### Cover Images

1. **Use descriptively** - Cover images should enhance understanding, not just decorate
2. **Show, don't tell** - Use diagrams for architecture, flowcharts for processes
3. **Organize assets** - Store images in `docs/assets/` or similar dedicated directory
4. **Name clearly** - Use descriptive names: `layered-architecture.png` not `img1.png`
5. **Optimize size** - Keep images under 2MB for fast loading
6. **Choose wisely** - Use cover images when visual explanation adds significant value

### Highlight Layers

1. **Limit colors** - Use 2-4 distinct colors maximum per step
2. **Contrast matters** - Choose colors that stand out against the default theme
3. **Border for groups** - Use `renderStrategy: "border"` for directories
4. **Fill for emphasis** - Use `renderStrategy: "fill"` for key files
5. **Adjust opacity** - Lower opacity (0.5-0.7) for large areas

### File Paths

1. **Use relative paths** - All paths relative to repository root
2. **Use forward slashes** - Even on Windows: `src/components/Button.tsx`
3. **No leading slash** - ❌ `/src/App.tsx` ✅ `src/App.tsx`
4. **Case sensitive** - Match exact file/directory casing

## Example Tour

See [examples/introduction-example.tour.json](../examples/introduction-example.tour.json) for a complete example.

### Minimal Example

```json
{
  "id": "quick-start",
  "title": "Quick Start Guide",
  "description": "Get started with the codebase in 5 minutes",
  "version": "1.0.0",
  "repos": [
    { "id": "pkg:github/acme/widgets", "name": "widgets" }
  ],
  "steps": [
    {
      "id": "step-1",
      "title": "Welcome!",
      "description": "This is a simple introduction to the project structure.",
      "estimatedTime": 30,
      "focusDirectory": "src",
      "colorMode": "fileTypes"
    }
  ]
}
```

## JSON Schema

A JSON schema is available for validation tools:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "IntroductionTour",
  "type": "object",
  "required": ["id", "title", "description", "version", "repos", "steps"],
  "properties": {
    "id": { "type": "string", "pattern": "^[a-z0-9-]+$" },
    "title": { "type": "string", "minLength": 1 },
    "description": { "type": "string", "minLength": 1 },
    "version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
    "repos": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/definitions/repo" }
    },
    "audience": { "type": "string" },
    "prerequisites": { "type": "array", "items": { "type": "string" } },
    "steps": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/definitions/step" }
    },
    "metadata": { "$ref": "#/definitions/metadata" }
  }
}
```

See [tour-schema.json](./tour-schema.json) for the complete schema definition.

## CLI Tool

Tours are authored, previewed, and shared with the published CLI, `@principal-ai/principal-view-cli@0.30.0` (run via `npx @principal-ai/principal-view-cli@latest tour …`).

### Commands

1. **`tour init`** - Initialize a new tour with a template
2. **`tour validate <file>`** - Validate tour JSON against schema and rules
3. **`tour stats <file>`** - Display tour statistics and timing analysis
4. **`tour view <file|id>`** - Preview a tour locally, or view a published tour by id
5. **`tour publish <file>`** - Publish a tour to the store; prints a share URL / mints an id
6. **`tour fetch <id>`** - Retrieve a published tour by id

### Validation

- JSON schema validation
- Path existence validation (files/directories exist in repo)
- Color code validation (valid hex colors)
- Unique ID validation (tour and step IDs)
- Cross-reference validation (layer IDs match in actions)

### Tour Statistics (`tour stats`)

Displays timing and content analysis to help meet the 2-minute ideal / 3-minute max target:

**Output should include:**
```
Tour Statistics: "Quick Start Guide"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Steps:               6 steps
Total duration:      2m 30s (150 seconds)
Total description:   1,423 characters

Target Guidelines:
  ✓ Steps: 6 (within 4-6 ideal range)
  ⚠ Duration: 2m 30s (exceeds 2min ideal, within 3min max)
  ✓ Characters: 1,423 (within 800-1,500 ideal range)

Per-Step Breakdown:
  1. Welcome          (25s, 245 chars) ✓
  2. Core Components  (30s, 280 chars) ⚠ Approaching limit
  3. Configuration    (20s, 198 chars) ✓
  4. Data Flow        (35s, 312 chars) ✗ Exceeds 300 char limit
  5. Testing          (22s, 210 chars) ✓
  6. Next Steps       (18s, 178 chars) ✓

Recommendations:
  • Reduce step 4 description by 12 characters
  • Consider reducing overall duration to meet 2-minute ideal
```

**Metrics to calculate:**
- Total step count
- Sum of all `estimatedTime` values (if provided)
- Total character count across all step descriptions
- Per-step character counts
- Comparison against targets:
  - ✓ Ideal: 4-6 steps, ≤120s, 800-1,500 chars
  - ⚠ Acceptable: 6-8 steps, ≤180s, 1,500-2,000 chars
  - ✗ Over limit: >8 steps, >180s, >2,000 chars
- Flag individual steps exceeding 300 characters
- Suggest estimated time per step if not provided (based on char count)

### Generation

- Interactive prompts for tour metadata
- Step-by-step wizard for creating steps
- Auto-discovery of important directories
- Suggested highlights based on git history
- Template selection (onboarding, feature tour, architecture overview)

### Output

- Formatted JSON with proper indentation
- Validation error messages with line numbers
- Suggestions for improvements
- Preview mode showing what tour will look like

## Authoring, Viewing, and Publishing

Tours are authored locally and either viewed locally or published to a store (the same backing model as trails) and fetched by id. They are **not** committed to the repository root or auto-discovered by a file scan.

1. **Author a tour file locally** - Create a `*.tour.json` anywhere; it is not tied to a fixed location in the repo. Make sure `repos` names the repo(s) the tour's directories belong to.

2. **Preview it locally**:
   ```bash
   npx @principal-ai/principal-view-cli@latest tour view my-tour.tour.json
   ```
   In local mode the viewer builds File City from a working tree. It defaults to the current working directory; point it at a different checkout with `--repo-root <path>`:
   ```bash
   npx @principal-ai/principal-view-cli@latest tour view my-tour.tour.json --repo-root ../widgets
   ```

3. **Publish it to share**:
   ```bash
   npx @principal-ai/principal-view-cli@latest tour publish my-tour.tour.json
   ```
   This mints an id and prints a share URL.

4. **Others retrieve it** by id:
   ```bash
   npx @principal-ai/principal-view-cli@latest tour fetch <id>
   npx @principal-ai/principal-view-cli@latest tour view <id>
   ```

### Standalone Viewer

The standalone viewer ships as a macOS arm64 prebuild (the optional dependency `@principal-ai/trail-viewer`). On other platforms, pass `--viewer-dir <path>` pointing at a source checkout of the viewer.

### Validation Errors

Validation runs on `tour validate`, and again before `view`/`publish`. Errors are reported with a message such as:

```
[Tour] Tour validation failed: Invalid 'version' - must be semantic version (e.g., '1.0.0')
```

## Related Documentation

- [TourPlayer Integration Guide](./STORYPLAYER_INTEGRATION.md)
- [Context Panel Guide](./CONTEXT_PANEL_GUIDE.md)
- [Type Definitions](../src/types/IntroductionTour.ts)
- [Tour Parser Utilities](../src/utils/tourParser.ts)

## Version History

- **1.0.0** (2026-02-03) - Initial format specification
- **2.0.0** (2026-06-20) - Required `repos` field added; tours authored/viewed/published via the CLI instead of git auto-discovery
