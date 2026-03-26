# figma-mcp-skills

A collection of Claude Code skills for working with Figma via the [Figma MCP server](https://www.figma.com/developers/mcp). These skills give Claude structured workflows for reading designs, writing to the Figma canvas, building design systems, generating code, and connecting components to code.

## Prerequisites

- Claude Code CLI installed
- Figma MCP server connected in your Claude Code config (`.claude.json` or `.mcp.json`)
- A Figma account with API access

### Figma MCP server config (`.mcp.json`)

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "@figma/mcp-server"]
    }
  }
}
```

---

## Skills Overview

| Skill | Trigger | What it does |
|-------|---------|--------------|
| [`figma-use`](#figma-use) | Any canvas write | Core skill — execute Plugin API JS in Figma |
| [`figma-generate-design`](#figma-generate-design) | "Build this page in Figma" | Build full screens from code using design system components |
| [`figma-generate-library`](#figma-generate-library) | "Create a design system" | Build professional-grade Figma design systems from a codebase |
| [`figma-implement-design`](#figma-implement-design) | "Implement this Figma design" | Generate production code from Figma designs |
| [`figma-code-connect-components`](#figma-code-connect-components) | "Code Connect this component" | Map Figma components to codebase components |
| [`figma-create-design-system-rules`](#figma-create-design-system-rules) | "Create design system rules" | Generate `CLAUDE.md`/`AGENTS.md` rules for Figma-to-code workflows |
| [`figma-create-new-file`](#figma-create-new-file) | "Create a new Figma file" | Create a blank Figma or FigJam file |

---

## Skills

### `figma-use`

**The foundation skill.** Must be loaded before every `use_figma` MCP tool call. It teaches Claude the Plugin API rules, critical gotchas, and incremental workflow patterns that prevent silent failures.

**Invoke:** any canvas write task, or as a prerequisite to other skills

**What it covers:**
- Plugin API rules (return pattern, async/await, color range 0–1, font loading, page context reset)
- Incremental workflow: one section per `use_figma` call, validate after each step
- How to avoid the most common bugs (FILL before appendChild, resize before HUG/AUTO, etc.)
- Error recovery: failed scripts are atomic — read the error, fix, retry
- Full reference docs in `references/`:

| Reference | Contents |
|-----------|----------|
| `gotchas.md` | Every known pitfall with WRONG/CORRECT code examples |
| `common-patterns.md` | Script scaffolds for shapes, text, auto-layout, variables, components |
| `component-patterns.md` | Importing by key, finding variants, `setProperties`, text overrides |
| `variable-patterns.md` | Creating/binding variables, importing library variables, scopes |
| `text-style-patterns.md` | Creating/applying text styles, importing library text styles |
| `effect-style-patterns.md` | Creating/applying effect styles (shadows, blurs) |
| `plugin-api-patterns.md` | Fills, strokes, Auto Layout, effects, grouping, cloning, styles |
| `api-reference.md` | Node creation, variables API, core properties |
| `validation-and-recovery.md` | `get_metadata` vs `get_screenshot` workflow, error recovery steps |
| `plugin-api-standalone.d.ts` | Full Figma Plugin API type definitions (~450KB) |
| `plugin-api-standalone.index.md` | Index of all types, methods, and properties |
| `working-with-design-systems/` | Deep guides on variables, components, text styles, and effect styles |

**Critical rules (summary):**

```js
// ✅ Return data — never figma.closePlugin()
return { createdNodeIds: [...] };

// ✅ Top-level await — no async IIFE wrapper
const comp = await figma.importComponentByKeyAsync("abc123");

// ✅ Colors are 0–1, not 0–255
{ r: 1, g: 0, b: 0 }  // red

// ✅ FILL sizing AFTER appendChild
parent.appendChild(child);
child.layoutSizingHorizontal = "FILL";

// ✅ resize() BEFORE primaryAxisSizingMode
frame.resize(1440, 10);
frame.primaryAxisSizingMode = "AUTO"; // HUG

// ✅ loadFontAsync BEFORE any text change
await figma.loadFontAsync({ family: "Inter", style: "Regular" });

// ✅ Page switch uses async setter
await figma.setCurrentPageAsync(page);

// ❌ Never use figma.notify() — throws "not implemented"
```

---

### `figma-generate-design`

Build full-page screens in Figma by reusing published design system components, variables, and styles — not drawing primitives with hardcoded values.

**Invoke:** "Build this page in Figma", "Create a landing page using our design system"

**Workflow:**

1. **Understand the screen** — read source code, identify sections and components
2. **Discover the design system** — search components, variables, and styles via `search_design_system`; inspect existing screens first for the most authoritative map
3. **Create the page wrapper frame first** — one `use_figma` call, return its ID
4. **Build each section inside the wrapper** — one section per call, append directly to wrapper (never reparent across calls)
5. **Validate** — `get_screenshot` after each section; screenshot individual sections to catch clipped text and overlapping elements
6. **Fix issues** — targeted fixes only, don't rebuild the whole screen

**Key patterns:**

```js
// Import design system components by key
const buttonSet = await figma.importComponentSetByKeyAsync("BUTTON_SET_KEY");
const primary = buttonSet.children.find(c => c.name.includes("variant=primary"));
const instance = primary.createInstance();

// Import and bind design system variables (never hardcode hex/px)
const colorVar = await figma.variables.importVariableByKeyAsync("COLOR_VAR_KEY");
const paint = figma.variables.setBoundVariableForPaint(
  { type: "SOLID", color: { r: 0, g: 0, b: 0 } }, "color", colorVar
);
frame.fills = [paint];

// Override text via component properties
instance.setProperties({ "Label#1530:99": "Get started" });

// Or find the TEXT node directly for components without exposed properties
const textNode = instance.findOne(n => n.type === "TEXT");
if (textNode) textNode.characters = "Get started";
```

**For web apps:** run `generate_figma_design` in parallel to capture a pixel-perfect screenshot reference, then align your `use_figma` output to match it.

---

### `figma-generate-library`

Build or update a professional-grade design system in Figma from a codebase — variables/tokens, component libraries, theming (light/dark modes), documentation pages.

**Invoke:** "Create a design system in Figma", "Build a component library from our codebase", "Set up tokens and theming"

**This is never a one-shot task.** Requires 20–100+ `use_figma` calls across multiple phases, with mandatory user checkpoints between phases.

**Phase order (never skip or reorder):**

```
Phase 0: DISCOVERY
  0a. Analyze codebase → extract tokens, components, naming conventions
  0b. Inspect Figma file → existing pages, variables, components, styles
  0c. Search subscribed libraries → avoid duplicating existing assets
  0d. Lock v1 scope → agree on exact token set + component list
  ✋ USER CHECKPOINT — await explicit approval before any writes

Phase 1: FOUNDATIONS (tokens before components — always)
  1a. Create variable collections and modes
  1b. Create primitive variables (raw values)
  1c. Create semantic variables (aliased to primitives, mode-aware)
  1d. Set scopes on all variables (never use ALL_SCOPES)
  ✋ USER CHECKPOINT

Phase 2: TYPOGRAPHY
  2a. Load and validate fonts
  2b. Create text styles (heading, body, caption, code, etc.)
  ✋ USER CHECKPOINT

Phase 3: COMPONENTS (one component per call)
  3a. Create base/primitive components
  3b. Combine as variants (combineAsVariants)
  3c. Add component properties (VARIANT, BOOLEAN, TEXT, INSTANCE_SWAP)
  3d. Bind design tokens to component properties
  ✋ USER CHECKPOINT per component type

Phase 4: DOCUMENTATION
  4a. Create component documentation page
  4b. Add usage examples and annotations
```

**Reference docs in `figma-generate-library/references/`:**

| Reference | Contents |
|-----------|----------|
| `discovery-phase.md` | Codebase analysis, Figma inspection, scope locking |
| `token-creation.md` | Variable collections, modes, scopes, aliasing patterns |
| `component-creation.md` | combineAsVariants, properties, variant layout, naming |
| `naming-conventions.md` | Token naming, component naming, page structure |
| `documentation-creation.md` | Documentation page layout, annotation patterns |
| `code-connect-setup.md` | Publishing components and setting up Code Connect |
| `error-recovery.md` | State rehydration, orphan cleanup, recovery patterns |

**Reusable scripts in `figma-generate-library/scripts/`:**

| Script | Purpose |
|--------|---------|
| `createVariableCollection.js` | Create a variable collection with modes |
| `createSemanticTokens.js` | Create semantic alias variables |
| `createComponentWithVariants.js` | Create a component set with variants |
| `bindVariablesToComponent.js` | Bind tokens to component nodes |
| `inspectFileStructure.js` | Read existing file structure before writing |
| `createDocumentationPage.js` | Build a documentation page |
| `validateCreation.js` | Verify created assets match expected state |
| `rehydrateState.js` | Recover state across context resets |
| `cleanupOrphans.js` | Remove orphaned frames from failed calls |

---

### `figma-implement-design`

Translate Figma designs into production-ready application code with 1:1 visual fidelity.

**Invoke:** "Implement this Figma design", "Generate code from this component", "Build this UI from Figma"

**Workflow:**

1. **Get node ID** — parse from Figma URL (`node-id=1-2` → `nodeId: "1:2"`) or use Figma desktop MCP
2. **Call `get_design_context`** — primary tool; returns code reference, screenshot, and hints
3. **Analyze the output** — Code Connect snippets mean use the mapped codebase component directly; raw hex/absolute positioning means the design is loosely structured, use the screenshot
4. **Check the project** — find existing components, tokens, and patterns that match the design intent
5. **Implement** — adapt to the project's stack, not the raw output

**URL parsing:**
```
https://figma.com/design/:fileKey/:fileName?node-id=1-2
                   ↑ fileKey                       ↑ nodeId = "1:2" (hyphen → colon)
```

**Key principle:** `get_design_context` output is a reference, not final code. Always adapt to the target project's stack, components, and conventions. Reuse what exists instead of generating from scratch.

---

### `figma-code-connect-components`

Map Figma design components to their code implementations using Figma's Code Connect feature. Once mapped, Figma's Dev Mode shows the real component code (not generated approximations) when developers inspect a design.

**Invoke:** "Code Connect this component", "Connect my Figma components to code", "Map these components"

**Requirements:**
- Figma Organization or Enterprise plan (Code Connect is not available on Pro/Starter)
- Components must be published to a team library
- Figma URL must include `node-id` parameter

**Workflow:**

1. **Call `get_code_connect_suggestions`** — pass `fileKey` + `nodeId`; returns unmapped components with names, properties, and thumbnails
2. **Search the codebase** — find the matching source component for each Figma component
3. **Build mappings** — match Figma variant properties to code component props
4. **Call `send_code_connect_mappings`** — submit all mappings in one call

**Mapping example:**
```json
{
  "figmaNodeId": "1:23",
  "codeComponent": "Button",
  "filePath": "src/components/Button.tsx",
  "props": {
    "variant": { "figmaProp": "Variant", "mapping": { "Primary": "primary", "Outline": "outline" } },
    "size": { "figmaProp": "Size", "mapping": { "Default": "md", "Large": "lg" } }
  }
}
```

---

### `figma-create-design-system-rules`

Generate project-specific design system rules that encode your team's conventions — which components to use, where files live, what never to hardcode, how to handle tokens — so AI agents produce consistent output without repeated prompting.

**Invoke:** "Create design system rules for my project", "Set up Figma-to-code guidelines"

**Supported rule files:**

| Agent | Rule file |
|-------|-----------|
| Claude Code | `CLAUDE.md` |
| Codex CLI | `AGENTS.md` |
| Cursor | `.cursor/rules/figma-design-system.mdc` |

**What gets generated:**
- Which layout primitives and components to use (and when)
- Component file locations and naming conventions
- What must never be hardcoded (colors, spacing, radii)
- How to handle design tokens and CSS variables
- Project-specific architectural patterns
- Stack-specific guidance (Next.js, React Native, etc.)

---

### `figma-create-new-file`

Create a blank Figma design or FigJam file. Used before `use_figma` when you need a fresh file.

**Invoke:** "Create a new Figma file", `/figma-create-new-file [editorType] [fileName]`

**Arguments:**
- `editorType`: `design` (default) or `figjam`
- `fileName`: name for the file (default: `"Untitled"`)

**Examples:**
```
/figma-create-new-file
/figma-create-new-file figjam My Whiteboard
/figma-create-new-file design Landing Page
```

**Requires a `planKey`** — call `whoami` first to resolve which team/org to create the file in, then pass the plan's `key` to `create_new_file`.

---

## Skill Relationships

```
figma-use          ← foundation, always loaded alongside any other skill
  └── figma-generate-design    ← build screens from code using design system
  └── figma-generate-library   ← build the design system itself
  └── figma-create-new-file    ← create a blank file first

figma-implement-design         ← Figma → code (reads Figma, writes to repo)
figma-code-connect-components  ← map Figma components to code components
figma-create-design-system-rules ← generate CLAUDE.md/AGENTS.md rules
```

**Direction of data flow:**
- `figma-use`, `figma-generate-design`, `figma-generate-library`, `figma-create-new-file` → **write to Figma**
- `figma-implement-design` → **read from Figma, write to codebase**
- `figma-code-connect-components` → **bidirectional** (read Figma + codebase, write mappings back)
- `figma-create-design-system-rules` → **read from both, write rules to codebase**

---

## Common Workflows

### Build a new screen in Figma from your web app

1. Load `figma-use` + `figma-generate-design`
2. Read the source page/component files
3. Discover design system components with `search_design_system`
4. Create a page wrapper frame
5. Build each section one `use_figma` call at a time
6. Validate with `get_screenshot` after each section

### Build a design system from your codebase

1. Load `figma-use` + `figma-generate-library`
2. Run Phase 0 discovery (no writes yet)
3. Get user approval on scope
4. Run Phases 1–4 in order with checkpoints

### Implement a Figma design as code

1. Copy the Figma URL including `?node-id=`
2. Load `figma-implement-design`
3. Run `get_design_context` with the parsed `fileKey` and `nodeId`
4. Adapt the output to your project's stack and conventions

### Connect Figma components to code

1. Load `figma-code-connect-components`
2. Run `get_code_connect_suggestions` with the Figma URL
3. Match each component to its codebase counterpart
4. Run `send_code_connect_mappings`

---

## Source & Credits

These skills are sourced from the **official Figma Claude plugin** maintained by Figma and Anthropic. They are bundled with Claude Code and cached locally at:

```
~/.claude/plugins/cache/claude-plugins-official/figma/2.0.2/skills/
```

This repo is a readable, browsable snapshot of those skills at **version 2.0.2**. All skill content is the work of Figma and Anthropic — this repo exists purely for reference and discoverability.

- Official Figma MCP docs: https://www.figma.com/developers/mcp
- Claude Code: https://claude.ai/code
