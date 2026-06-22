---
name: file-city-tours
description: Create, validate, preview, and publish introduction tours for File City visualizations. Use when users want to (1) create onboarding tours for codebases, (2) build guided walkthroughs highlighting architecture, (3) validate or fix tour files, (4) preview a tour locally in the viewer, or (5) publish/fetch tours from the store. Tours require a `repos` field and are published (not committed to the repo root).
license: MIT
---

# File City Tours

Create guided introduction tours that help users navigate and understand codebases using File City visualizations.

## Quick Start

### Using the CLI (Recommended)

Tour authoring lives under the Principal CLI's `tour` subcommand. Use `npx` to run `@principal-ai/principal-view-cli@latest tour …` without installing globally:

```bash
# Create a new tour from template (stamps `repos` from the cwd git remote)
npx @principal-ai/principal-view-cli@latest tour init --template onboarding

# Validate a tour file
npx @principal-ai/principal-view-cli@latest tour validate my-tour.tour.json

# Preview it locally in the File City viewer (run from a checkout of the repo,
# or pass --repo-root <path>)
npx @principal-ai/principal-view-cli@latest tour view my-tour.tour.json

# Publish it to the store; prints a share URL with the minted tour id
npx @principal-ai/principal-view-cli@latest tour publish my-tour.tour.json

# Fetch / open a published tour by id or URL
npx @principal-ai/principal-view-cli@latest tour fetch <id-or-url>
npx @principal-ai/principal-view-cli@latest tour view <id-or-url>

# Analyze timing/length against the authoring guidelines
npx @principal-ai/principal-view-cli@latest tour stats my-tour.tour.json

# Available templates: minimal, onboarding, architecture
```

**Note**: `npx … @latest` runs the current CLI (≥ 0.30.0), which adds
`view`/`publish`/`fetch` and requires the `repos` field (below).

**Viewer**: `tour view` launches the standalone viewer, shipped as a macOS
arm64 prebuild (the optional `@principal-ai/trail-viewer` dependency). On other
platforms, pass `--viewer-dir <path>` pointing at a source checkout.

### Manual Creation

Use template files from `assets/` directory:
- `minimal-template.json` - Simple single-step tour
- `onboarding-template.json` - Multi-step with highlights
- `architecture-template.json` - Layered architecture showcase

Each template carries a placeholder `repos` entry (`pkg:github/OWNER/REPO`) —
replace `OWNER`/`REPO` with the real repository before validating or publishing.

## Tour Structure

Tours are JSON files ending with `.tour.json`. They are **authored locally**,
**previewed** in the viewer (`tour view`), and **published to the store**
(`tour publish`) — they are no longer committed to the repo root or
auto-discovered by a file scan. A repo can have many tours; each gets its own id
when published, and is retrieved by id (`tour fetch` / `tour view <id>`).

### Minimal Structure

```json
{
  "id": "tour-id",                    // kebab-case
  "title": "Tour Title",
  "description": "What this covers",
  "version": "1.0.0",                 // semantic versioning
  "repos": [                          // REQUIRED — which repo(s) the tour describes
    {
      "id": "pkg:github/owner/name",
      "name": "name",
      "remote": { "host": "github", "owner": "owner", "name": "name" }
    }
  ],
  "steps": [
    {
      "id": "step-1",                 // kebab-case
      "title": "Step Title",
      "description": "Detailed explanation",
      "focusDirectory": "src",        // optional
      "colorMode": "fileTypes"        // optional
    }
  ]
}
```

### Repos (required)

A tour describes *directories* of a codebase, so it must name the repo(s) those
directories belong to — that's how the viewer knows which working tree to build
File City from, and how the store buckets and serves the tour.

```json
"repos": [
  {
    "id": "pkg:github/owner/name",                          // Purl (required)
    "name": "name",                                          // short name (required)
    "remote": { "host": "github", "owner": "owner", "name": "name" },  // optional
    "roots": ["packages/cli"],                               // optional: scope to subtree(s)
    "authoredAtSha": "<sha>"                                 // optional: provenance
  }
]
```

- `repos[0]` is the **primary** repo. The array supports **multi-repo** tours.
- `roots` scopes the rendered city to specific subtree(s) — use it to keep very
  large repos legible. Absent/empty = render the whole repo.
- `tour init` and `tour publish` fill `repos` in for you (derived from the cwd
  git remote). Hand-authored tours **must** include it, or validation fails.
- Honoring `roots`/multi-repo in the 3D rendering is in progress; the field is
  valid and stored today regardless.

## Tour Content Strategy

### Focus on Concepts, Not Just Structure

**Tours should teach mental models, not just show directories.**

When creating tours, prioritize explaining:
- **What the code does** - Core functionality and purpose
- **How it works** - Architectural patterns and mechanisms (state management, reconciliation, etc.)
- **Why it's designed this way** - Design decisions and trade-offs
- **How components relate** - Relationships between different parts
- **Where to extend** - How developers can build on top of it

**Concrete descriptions beat abstract labels:**
- ❌ "Core packages with framework functionality"
- ✅ "LexicalEditor.ts manages the editor instance and wires everything together - updates, listeners, commands, and DOM reconciliation"

**Show implementation of concepts:**
- Connect architectural concepts to actual code files
- Use highlights to show which files implement which concepts
- Explain patterns like immutability, double-buffering, command dispatching

**Build understanding progressively:**
- Start with high-level concepts ("What is this?")
- Show the core engine/system ("How does it work?")
- Highlight key abstractions ("What are the building blocks?")
- Demonstrate extensibility ("How can I use/extend it?")
- Point to examples ("Where can I see it in action?")

### Key Features

**Focus & Zoom**
```json
"focusDirectory": "src/components",  // Zoom into specific directory
"focusDirectory": "",                // Repository root (entire codebase)
"focusDirectory": "src"              // Top-level directory
```

**Important**: Steps with `highlightLayers` **must** include `focusDirectory`. Use `""` (empty string) to focus on repository root.

**Highlight Layers**
```json
"highlightLayers": [{
  "id": "core-files",
  "name": "Core Components",
  "color": "#3b82f6",                // hex color
  "items": [
    { "path": "src/index.ts", "type": "file" }
  ],
  "opacity": 0.7,
  "renderStrategy": "fill"           // see render-strategy rule below
}]
```

**Render strategy — prefer `fill` for files.** A `border` is a thin
outline drawn around a building. Files are small buildings, so a border on a
file is easy to miss — especially when the step also sets a `focusDirectory`,
because the whole focused district keeps its file-type fill colors and the thin
outline blends in. Use:
- **`fill`** for **file** highlights (and any time you need a few specific
  buildings to pop) — it recolors the whole building with the layer color.
- **`border`** for **directory / region** highlights, where outlining a large
  area reads clearly and a fill would swamp it.

Mixing types in one layer? Split files (`fill`) and directories (`border`)
into separate layers so each gets the right strategy.

**Interactive Actions**
```json
"interactiveActions": [{
  "type": "click-file",              // or hover-directory, toggle-layer, explore
  "description": "Click to view",
  "target": "src/App.tsx"
}]
```

**Color Modes**
Available: `fileTypes` (default), `git`, `pr`, `commit`, `coverage`, `eslint`, `typescript`, `prettier`, `knip`, `alexandria`

## Tour Philosophy: Concepts Over Structure

**IMPORTANT**: Focus tours on **what the code does** and **core concepts**, not just file structure.

### Good Tour Practices ✅
- **Explain architectural concepts** - "This uses an immutable state model for reliable updates"
- **Show relationships between components** - "Editor manages EditorState, which contains the node tree"
- **Describe functionality** - "These nodes are immutable - getWritable() creates clones for editing"
- **Connect files to concepts** - "LexicalEditor.ts manages the editor instance and wires everything together"
- **Build understanding progressively** - Start with core concepts, then show how they're implemented

### What to Avoid ❌
- **Don't just list directories** - "The packages directory contains all modules" (too generic)
- **Don't focus only on structure** - "These are the source files" (doesn't explain what they do)
- **Avoid surface-level descriptions** - "Config files" without explaining their purpose
- **Don't skip the "why"** - Always explain why something exists, not just where it is

### Example Comparison

**Before (Structure-focused):**
```
"The packages directory contains the core functionality.
The lexical package has the main code, and lexical-react has React bindings."
```

**After (Concept-focused):**
```
"Lexical's core: LexicalEditor.ts manages the editor instance,
LexicalEditorState.ts holds immutable content snapshots.
Updates use double-buffering - clone state, mutate, reconcile to DOM."
```

## Common Patterns

### Concept-Driven Onboarding
1. **What is it?** - Explain the core purpose and architecture (e.g., "immutable state editor framework")
2. **Core Engine** - Show the main system files and how they work together (Editor, State, Updates, Reconciler)
3. **Key Abstractions** - Highlight the primary abstractions (Nodes, Commands, Transforms) and their relationships
4. **Extensibility** - Show the plugin/extension system and how features compose
5. **See It Working** - Point to examples/playground showing concepts in action
6. **Next Steps** - Resources and documentation, `focusDirectory: ""` for complete overview

### Architecture Tour
1. **Core Concepts** - Explain the fundamental architectural patterns (state management, reconciliation, etc.)
2. **Component Relationships** - Show how major components interact using multiple highlight layers
3. **Data Flow** - Explain how data moves through the system with visual highlights
4. **Extension Points** - Show where and how developers can extend functionality
5. **Patterns in Practice** - Link to examples demonstrating architectural patterns

### Feature Deep-Dive
1. **Feature Overview** - What does this feature do and why does it exist?
2. **Implementation** - Show the core files implementing the feature
3. **Integration Points** - How does it integrate with the rest of the system?
4. **Usage Examples** - Where is this feature used in practice?

### Recent Changes
Use `"colorMode": "git"` to highlight modified files, but **explain what changed and why**, not just "these files were modified"

## Validation

**Always validate before deploying:**
```bash
npx @principal-ai/principal-view-cli@latest tour validate your-tour.tour.json
```

Common errors and fixes are in `references/troubleshooting.md`.

## Best Practices

### Content Guidelines
1. **Teach concepts, not just structure** - Explain what the code does and how it works, not just where files are
2. **Connect code to concepts** - Highlight files while explaining the concepts they implement
3. **Explain the "why"** - Include architectural reasoning and design decisions
4. **Show relationships** - Use highlight layers to show how components work together
5. **One concept per step** - Don't overwhelm, stay focused on a single idea

### Technical Guidelines
6. **Set `repos`** - Required; let `tour init`/`publish` derive it from the git remote, or fill it by hand. A repo can have many tours (each published under its own id).
7. **Target duration: 2 minutes ideal, 3 minutes max** - Keep tours concise and focused:
   - **4-6 steps** for 2-minute tours (ideal)
   - **6-8 steps maximum** for 3-minute tours
   - **20-30 seconds per step** - Include reading + viewing + interaction time
   - **200-250 characters per description** - Max 300 characters
   - **Total text: 800-1,500 chars** for 2 minutes, up to 2,000 chars for 3 minutes
8. **Use relative paths** - No leading `/` or `./`
9. **Test thoroughly** - Walk through the tour in File City and verify timing
10. **Hex colors only** - Format: `#RRGGBB` or `#RGB`
11. **Kebab-case IDs** - Lowercase, hyphens only
12. **Always set focusDirectory with highlightLayers** - Ensures camera focuses on highlighted area:
    - Use `""` for repository root (full codebase view)
    - Use `"src"` to focus on a specific top-level directory
    - Use `"src/components"` to zoom into nested areas
13. **Last step must focus on root** - Set `"focusDirectory": ""` on the final step for complete overview
14. **`fill` for files, `border` for directories** - A border on a small file building is hard to see; `fill` recolors the building so it pops. Reserve `border` for directory/region highlights (see "Render strategy" above)

## Path Rules

**Correct:**
- `src/components`
- `src/App.tsx`
- `package.json`

**Incorrect:**
- `/src/components` (leading slash)
- `./src/components` (dot-slash)
- `src\components` (backslash)

## References

- **Full specification**: `references/tour-format-spec.md` - Complete API reference
- **Examples**: `references/examples.md` - Ready-to-use tour templates
- **Troubleshooting**: `references/troubleshooting.md` - Common issues and solutions

## Workflow

1. **Create** - `tour init`, or copy from `assets/` templates (then set `repos`)
2. **Customize** - Edit tour steps for your codebase
3. **Validate** - `tour validate`
4. **Preview** - `tour view <file>` from a checkout of the repo (or `--repo-root <path>`)
5. **Publish** - `tour publish <file>` → share the printed URL / id
6. **Iterate** - Refine based on feedback; re-publish

## Templates

Use as starting points (in `assets/` directory):
- **minimal-template.json** - Quick 1-step intro
- **onboarding-template.json** - 3-step developer onboarding
- **architecture-template.json** - Layered architecture showcase

Copy template, customize paths/content for your codebase, then validate.
