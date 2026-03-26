---
name: figma-build-from-url
description: Build any web page in Figma from a URL, replicating content exactly and using real design system components. Use this skill when the user provides a URL and asks to "build this in Figma", "create a Figma design from this page", "replicate this site in Figma", "turn this URL into a Figma design", or "build [URL] in Figma". Always use this skill when a URL is provided alongside any Figma intent.
---

# Build a Web Page in Figma from a URL

This skill takes a live URL, fetches the page content, and builds a faithful Figma replica using real design system component instances — not primitive frames styled to look like components.

**MANDATORY**: Load [figma-use](../figma-use/SKILL.md) and [figma-generate-design](../figma-generate-design/SKILL.md) before any `use_figma` call. Those skills contain the Plugin API rules and screen-building workflow that this skill depends on.

## Inputs

- **URL** (required): The page to replicate
- **Figma file key** (optional): Target file to build in. If not provided, create a new file first using `create_new_file`.
- **Design system** (optional): Which published library to use for components. If the user has a known kit (e.g. Obra shadcn-ui), use it. Otherwise discover via `search_design_system`.

## Required Workflow

Follow these steps in order. Do not skip steps.

### Step 1: Fetch and Analyse the Page

Use `WebFetch` to retrieve the URL. Extract:

- **Page title and meta description**
- **Sections** in document order (nav, hero, features, pricing, testimonials, footer, etc.)
- **Exact copy** for every visible text element — headings, body, labels, CTAs, captions
- **Component inventory**: for each section, list which UI components are present (buttons, badges, cards, avatars, inputs, separators, accordions, etc.) and their variants (primary/outline/ghost, default/large, etc.)
- **Layout signals**: column counts, alignment (center/left), approximate spacing rhythm

Document this as a structured section map before touching Figma. Example:

```
Section Map
───────────
[Nav]
  Logo: "Acme"
  Links: Product, Pricing, Docs, Blog
  CTA: Button (primary, default) "Get started"

[Hero]
  Eyebrow: Badge (secondary) "New · v2.0 released"
  Heading: "Ship faster with Acme"
  Subheading: "The platform for teams who move fast."
  CTAs: Button (primary, large) "Start free", Button (outline, large) "View docs"

[Features]  3-column grid
  Card 1: icon, "Fast", "Deploy in seconds..."
  Card 2: icon, "Reliable", "99.99% uptime..."
  Card 3: icon, "Scalable", "Grow without limits..."
```

### Step 2: Discover Design System Components

Search the target Figma file's linked libraries for real components matching the inventory from Step 1.

**Always inspect existing screens first** — if the file already has pages with component instances, walk them to get an authoritative component key map.

Search broadly using `search_design_system` with multiple terms:
- "button", "badge", "card", "avatar", "separator", "input", "nav", "accordion", "tab", "toggle", "checkbox"

For each matched component:
- Record its key, name, and type (COMPONENT vs COMPONENT_SET)
- Note which TEXT properties it exposes (for `setProperties()` overrides)
- Note which variant properties it has (e.g. `variant`, `size`, `state`)

**Never build fake versions of components that exist in the library.** If a component exists, import and instance it. Only build manually when no matching component exists.

### Step 3: Prepare the Figma File

If no target file was provided, create one:
- Call `whoami` to get the plan key
- Call `create_new_file` with an appropriate name (e.g. the page title or domain)

Then create a new page for the URL being replicated:
```js
const page = figma.createPage();
page.name = "url.com/path";  // use the URL path as the page name
await figma.setCurrentPageAsync(page);
```

Load required fonts before any text operations:
```js
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Medium" });
await figma.loadFontAsync({ family: "Inter", style: "SemiBold" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });
// Load whatever font family the design system uses
```

### Step 4: Create the Page Wrapper

One `use_figma` call. Position it clear of existing content. Return its ID.

```js
let maxX = 0;
for (const child of figma.currentPage.children) {
  maxX = Math.max(maxX, child.x + child.width);
}

const wrapper = figma.createFrame();
wrapper.name = "Page — url.com/path";
wrapper.layoutMode = "VERTICAL";
wrapper.primaryAxisAlignItems = "CENTER";
wrapper.counterAxisAlignItems = "CENTER";
wrapper.itemSpacing = 0;
wrapper.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
wrapper.resize(1440, 100);
wrapper.layoutSizingHorizontal = "FIXED";
wrapper.primaryAxisSizingMode = "AUTO";
wrapper.x = maxX + 200;
wrapper.y = 0;

return { success: true, wrapperId: wrapper.id };
```

### Step 5: Build Each Section

One section per `use_figma` call. At the start of each script, fetch the wrapper by ID and append directly to it.

**For every section:**

1. Import the required components by key
2. Create the section frame with correct layout
3. Create instances of real components — never draw fake buttons/badges/cards
4. Override text using `setProperties()` for exposed TEXT properties, or `findOne(n => n.type === "TEXT")` for unexposed ones
5. Select the correct variant (read source defaults carefully — a button with no explicit `variant` prop may default to `"primary"`)
6. Append the section to the wrapper
7. Set `layoutSizingHorizontal = "FILL"` on the section **after** appending

**Text override pattern:**
```js
// Via component property (preferred)
instance.setProperties({ "Label#1530:99": "Get started" });

// Via direct TEXT node (fallback)
const textNode = instance.findOne(n => n.type === "TEXT");
if (textNode) textNode.characters = "Get started";
```

**Content accuracy rule:** The copy in Figma must match the live page exactly. Do not paraphrase, truncate, or invent content. If text is cut off in the fetched HTML, note it as `[...]` rather than guessing.

**Component fidelity rule:** Every button, badge, avatar, separator, and input must be a real component instance from the design system. If no matching component exists, note it in your response rather than faking it with a primitive frame.

### Step 6: Validate Each Section

After each section, call `get_screenshot` on that section's node ID. Check for:
- Clipped or truncated text (line heights cutting off content)
- Overlapping elements
- Placeholder text not overridden ("Button", "Label", "Title")
- Wrong variant selected (Neutral instead of Primary, etc.)
- Missing sections (sections in the source not yet built)

Fix issues with targeted scripts before moving to the next section.

### Step 7: Final Screenshot

After all sections are built, call `get_screenshot` on the wrapper node. Compare against the source URL visually. Note any remaining gaps (components that didn't exist in the library, content that couldn't be extracted from the page).

## Common Section Patterns

### Navigation
```js
const navSection = figma.createFrame();
navSection.layoutMode = "HORIZONTAL";
navSection.primaryAxisAlignItems = "CENTER";
navSection.counterAxisAlignItems = "CENTER";
navSection.primaryAxisSizingMode = "FIXED";
navSection.resize(1440, 10);
navSection.counterAxisSizingMode = "AUTO";
navSection.paddingLeft = 80;
navSection.paddingRight = 80;
navSection.paddingTop = 20;
navSection.paddingBottom = 20;
navSection.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
```

### Hero (centered, full-width)
```js
const heroSection = figma.createFrame();
heroSection.layoutMode = "VERTICAL";
heroSection.primaryAxisAlignItems = "CENTER";
heroSection.counterAxisAlignItems = "CENTER";
heroSection.primaryAxisSizingMode = "AUTO";
heroSection.counterAxisSizingMode = "FIXED";
heroSection.resize(1440, 10);
heroSection.paddingTop = 96;
heroSection.paddingBottom = 96;
heroSection.paddingLeft = 80;
heroSection.paddingRight = 80;
heroSection.itemSpacing = 24;
```

### Card Grid (3-column wrap)
```js
const gridSection = figma.createFrame();
gridSection.layoutMode = "HORIZONTAL";
gridSection.layoutWrap = "WRAP";
gridSection.primaryAxisSizingMode = "FIXED";
gridSection.resize(1200, 10);
gridSection.counterAxisSizingMode = "AUTO";
gridSection.itemSpacing = 24;
gridSection.counterAxisSpacing = 24;
```

### Footer
```js
const footerSection = figma.createFrame();
footerSection.layoutMode = "VERTICAL";
footerSection.primaryAxisAlignItems = "CENTER";
footerSection.counterAxisAlignItems = "CENTER";
footerSection.primaryAxisSizingMode = "AUTO";
footerSection.counterAxisSizingMode = "FIXED";
footerSection.resize(1440, 10);
footerSection.paddingTop = 48;
footerSection.paddingBottom = 48;
footerSection.paddingLeft = 80;
footerSection.paddingRight = 80;
footerSection.itemSpacing = 24;
```

## Error Recovery

If a `use_figma` call fails, **stop immediately**. Failed scripts are atomic — no partial changes are written. Read the error, fix the script, retry. See [figma-use](../figma-use/SKILL.md#6-error-recovery--self-correction) for the full recovery process.

Common errors in this workflow:

| Error | Cause | Fix |
|-------|-------|-----|
| Font not found | Wrong style name (e.g. "Semi Bold" vs "SemiBold") | Call `figma.listAvailableFontsAsync()` to discover exact names |
| Component not found | Key is wrong or library not enabled | Verify library is enabled in the target file; re-run `search_design_system` |
| `FILL` before parent | Set `layoutSizingHorizontal = "FILL"` before `appendChild` | Move sizing after `appendChild` |
| Wrapper node null | Wrong page context | Add `await figma.setCurrentPageAsync(page)` at start of each script |

## What to Report When Done

After completing the build, tell the user:
1. Which sections were built and what components were used
2. Which components were real instances vs manually built (be honest)
3. Any content that couldn't be extracted (behind auth, dynamically loaded, etc.)
4. The Figma file URL if a new file was created
