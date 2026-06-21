# Troubleshooting Guide

Common issues when creating and using File City tours.

## Tour Not Showing Up

**Problem**: Tour doesn't render when you `tour view` it (locally or by id)

Tours are no longer auto-discovered from the repository root. You author a tour file, preview it with `tour view`, and optionally publish it to share. If a tour won't show:

**Solutions**:
1. **Validate it**: Run validation to catch JSON syntax and schema errors
   ```bash
   npx @principal-ai/principal-view-cli@latest tour validate your-tour.tour.json
   ```
2. **Confirm `repos` is present and correct**: A tour requires at least one `TourRepoRef` in `repos`, each with `id` (e.g. `pkg:github/owner/name`) and `name`. Without it the tour has no codebase to render.
3. **Point at the right checkout**: In local mode the viewer builds File City from a working tree. Make sure you run `tour view` from a checkout of the tour's repo, or pass `--repo-root <path>` at one:
   ```bash
   npx @principal-ai/principal-view-cli@latest tour view your-tour.tour.json --repo-root ../widgets
   ```
4. **For published tours**: Confirm `tour publish` succeeded and printed an id / share URL, then retrieve with `tour fetch <id>` or `tour view <id>`.
5. **Viewer platform**: The bundled viewer is a macOS arm64 prebuild (`@principal-ai/trail-viewer`). On other platforms, pass `--viewer-dir <path>` to a source checkout of the viewer.

## Validation Errors

### "Invalid 'version' - must be semantic version"

**Problem**: Version field doesn't match X.Y.Z format

**Solution**: Use semantic versioning
```json
"version": "1.0.0"  // ✓ Correct
"version": "1.0"     // ✗ Wrong
"version": "v1.0.0"  // ✗ Wrong
```

### "Tour 'id' must be kebab-case"

**Problem**: ID contains invalid characters

**Solution**: Use lowercase, numbers, and hyphens only
```json
"id": "my-tour"         // ✓ Correct
"id": "my_tour"         // ✗ Wrong (underscore)
"id": "My-Tour"         // ✗ Wrong (uppercase)
"id": "my tour"         // ✗ Wrong (space)
```

### "'focusDirectory' must be a relative path (no leading slash)"

**Problem**: Path starts with `/`

**Solution**: Use relative paths from repository root
```json
"focusDirectory": "src/components"   // ✓ Correct
"focusDirectory": "/src/components"  // ✗ Wrong
"focusDirectory": "./src/components" // ✗ Wrong
```

### "Invalid hex color"

**Problem**: Color doesn't match hex format

**Solution**: Use 6-digit or 3-digit hex colors
```json
"color": "#3b82f6"  // ✓ Correct (6-digit)
"color": "#fff"     // ✓ Correct (3-digit)
"color": "blue"     // ✗ Wrong (name)
"color": "#3b82f"   // ✗ Wrong (5 digits)
```

### "'opacity' must be between 0 and 1"

**Problem**: Opacity value out of range

**Solution**: Use decimal between 0 (transparent) and 1 (opaque)
```json
"opacity": 0.7   // ✓ Correct
"opacity": 70    // ✗ Wrong (not percentage)
"opacity": 1.5   // ✗ Wrong (too high)
```

## Path Issues

### Files/directories not highlighting

**Problem**: Paths in tour don't match actual files

**Checklist**:
1. ✅ Paths are relative to repository root
2. ✅ No leading slash (/)
3. ✅ No leading dot-slash (./)
4. ✅ Correct case (file systems are case-sensitive)
5. ✅ Forward slashes (/) not backslashes (\\)

**Example**:
```json
{
  "items": [
    { "path": "src/App.tsx", "type": "file" },        // ✓ Correct
    { "path": "/src/App.tsx", "type": "file" },       // ✗ Wrong
    { "path": "./src/App.tsx", "type": "file" },      // ✗ Wrong
    { "path": "src\\App.tsx", "type": "file" }        // ✗ Wrong (Windows)
  ]
}
```

## Step Issues

### Duplicate step ID error

**Problem**: Multiple steps have the same `id`

**Solution**: Each step needs a unique ID
```json
{
  "steps": [
    { "id": "step-1", ... },  // ✓ Unique
    { "id": "step-2", ... },  // ✓ Unique
    { "id": "step-1", ... }   // ✗ Duplicate
  ]
}
```

### Auto-advance not working

**Problem**: Step doesn't advance automatically

**Solution**: Set both `autoAdvance` and `autoAdvanceDelay`
```json
{
  "autoAdvance": true,
  "autoAdvanceDelay": 3000  // Must be >= 1000ms
}
```

## Interactive Actions

### "Action requires a 'target'"

**Problem**: Action type needs a target but none provided

**Solution**: Add target for these action types:
- `click-file` → file path
- `hover-directory` → directory path
- `toggle-layer` → layer ID

```json
{
  "type": "click-file",
  "description": "Click on the main file",
  "target": "src/index.ts"  // Required
}
```

## CLI Installation Issues

### "tour: command not found"

**Problem**: CLI not installed globally

**Solutions**:
```bash
# Option 1: Install globally
npm install -g @principal-ai/principal-view-cli

# Option 2: Use npx
npx @principal-ai/principal-view-cli@latest tour validate my-tour.tour.json

# Option 3: Use local install
npm install --save-dev @principal-ai/principal-view-cli
npx tour validate my-tour.tour.json
```

## Best Practices

### Keep tours focused
- 5-10 steps is ideal
- Each step should teach one concept
- Don't overload with too much information

### Use appropriate color modes
- `fileTypes` - Default, shows file extensions
- `git` - Show modified files
- `coverage` - Test coverage visualization
- `typescript` - Type safety issues

### Test your tour
1. Create the tour file (include `repos`)
2. Validate with CLI: `npx @principal-ai/principal-view-cli@latest tour validate my-tour.tour.json`
3. Preview locally: `npx @principal-ai/principal-view-cli@latest tour view my-tour.tour.json` (run from the repo checkout, or add `--repo-root <path>`)
4. Walk through each step
5. Verify highlights and actions work
6. Share it: `npx @principal-ai/principal-view-cli@latest tour publish my-tour.tour.json`, then others use `tour fetch <id>` / `tour view <id>`

### Progressive disclosure
- Start with overview (step 1)
- Focus on specific areas (steps 2-N)
- End with next steps or resources
