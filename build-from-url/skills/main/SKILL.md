---
name: figma-build-from-url
description: Build a pixel-perfect 1:1 replica of any web page in Figma, using the exact colours, fonts, images, and content from the live URL. Use this skill when the user provides a URL and asks to "build this in Figma", "create a Figma design from this page", "replicate this site in Figma", "turn this URL into a Figma design", "clone this page into Figma", or "build [URL] in Figma". Always use this skill when a URL is provided alongside any Figma intent. Do NOT use design system components — replicate the page as-is.
---

# Build a 1:1 Figma Replica from a URL

This skill fetches a live URL and builds a pixel-faithful Figma replica using the page's exact colours, fonts, spacing, images, and text content. No design system components are used — the goal is a faithful copy of the source, not an abstraction of it.

**MANDATORY**: Load [figma-use](../figma-use/SKILL.md) before any `use_figma` call. That skill contains the Plugin API rules (return pattern, color range 0–1, font loading, page context reset, FILL after appendChild) that apply to every script you write.

## Inputs

- **URL** (required): The page to replicate
- **Figma file key** (optional): Target file. If not provided, create a new file first via `create_new_file`.

## Required Workflow

Follow these steps in order. Do not skip steps.

---

### Step 1: Fetch and Analyse the Page

Use `WebFetch` to retrieve the URL. Extract the following — be thorough and exact:

#### 1a. Style tokens

Pull exact values from the page's CSS. Look for:

- **Colours**: hex values used for backgrounds, text, borders, buttons, links, accents. Note which element type each applies to.
- **Fonts**: every font family and weight used, with the Google Fonts / Typekit name where possible (e.g. `"Inter"`, `"Geist"`, `"Playfair Display"`). Note which weight styles are needed (Regular, Medium, SemiBold, Bold, etc.)
- **Spacing rhythm**: the base spacing unit (4px, 8px, etc.), section padding, gap between elements
- **Border radii**: button radius, card radius, image radius
- **Font sizes**: heading sizes (h1, h2, h3), body, small/caption

Record these as a style token table before touching Figma:

```
Style Tokens
────────────
Colours
  Background:    #0A0A0A
  Surface:       #141414
  Text primary:  #FFFFFF
  Text muted:    #A1A1AA
  Accent:        #6366F1
  Border:        #27272A

Fonts
  Heading: "Geist" Bold (700) — sizes: 64px, 48px, 32px
  Body:    "Geist" Regular (400) — sizes: 16px, 14px
  UI:      "Geist" Medium (500) — sizes: 14px

Spacing
  Section padding: 96px top/bottom, 80px left/right
  Component gap: 24px
  Element gap: 12px

Radii
  Button: 6px
  Card: 12px
```

#### 1b. Section map

Map every section of the page in document order with its full copy:

```
Section Map
───────────
[Nav]
  Background: #0A0A0A, border-bottom: 1px solid #27272A
  Left: Logo "Acme" (Geist Bold 18px, white)
  Centre: links — Product, Pricing, Docs, Blog (Geist Regular 14px, #A1A1AA)
  Right: "Sign in" (text link, white), "Get started" (button, accent bg #6366F1, white text, 6px radius, 14px 20px padding)

[Hero]
  Background: #0A0A0A
  Centred layout, padding: 96px 80px
  Badge: "New · v2.0 released" (#27272A bg, #A1A1AA text, 6px radius)
  H1: "Ship faster with Acme" (Geist Bold 64px, white)
  Subheading: "The platform for teams who move fast." (Geist Regular 20px, #A1A1AA)
  CTAs: "Start free" (accent button), "View docs" (outline button, #27272A border)
  Gap between badge/h1/sub/ctas: 24px

[Features]  3-column grid, gap 24px
  Card bg: #141414, border: 1px #27272A, radius: 12px, padding: 32px
  Card 1: icon (purple), "Fast" (Geist SemiBold 20px white), "Deploy in seconds..." (Regular 14px muted)
  ...
```

Do not paraphrase or truncate copy. If text is cut off in the HTML, mark it `[...]`.

---

### Step 2: Prepare the Figma File

If no file was provided:
- Call `whoami` to get the plan key
- Call `create_new_file` with the site name (e.g. `"acme.com"`)

Then create a new page:
```js
const page = figma.createPage();
page.name = "acme.com/pricing";  // use actual path
await figma.setCurrentPageAsync(page);
return { pageId: page.id };
```

---

### Step 3: Load Fonts

Load every font family and style extracted in Step 1 before any text operations. If a font is not available in Figma (e.g. a custom or paid typeface), substitute with the closest available match and note it.

```js
// Example — adapt to the actual fonts found on the page
await figma.loadFontAsync({ family: "Geist", style: "Regular" });
await figma.loadFontAsync({ family: "Geist", style: "Medium" });
await figma.loadFontAsync({ family: "Geist", style: "SemiBold" });
await figma.loadFontAsync({ family: "Geist", style: "Bold" });
```

If a font fails to load, call `figma.listAvailableFontsAsync()` to find the exact available style name (e.g. `"Semi Bold"` vs `"SemiBold"`).

---

### Step 4: Create the Page Wrapper

One `use_figma` call. Position it clear of existing content. Return its ID.

```js
let maxX = 0;
for (const child of figma.currentPage.children) {
  maxX = Math.max(maxX, child.x + child.width);
}

const wrapper = figma.createFrame();
wrapper.name = "acme.com — /pricing";
wrapper.layoutMode = "VERTICAL";
wrapper.primaryAxisAlignItems = "CENTER";
wrapper.counterAxisAlignItems = "CENTER";
wrapper.itemSpacing = 0;
wrapper.clipsContent = false;
wrapper.fills = [{ type: "SOLID", color: { r: 0.04, g: 0.04, b: 0.04 } }]; // exact page bg colour
wrapper.resize(1440, 100);
wrapper.layoutSizingHorizontal = "FIXED";
wrapper.primaryAxisSizingMode = "AUTO";
wrapper.x = maxX + 200;
wrapper.y = 0;

return { success: true, wrapperId: wrapper.id };
```

---

### Step 5: Build Each Section

One section per `use_figma` call. At the start of each script, fetch the wrapper by ID and append directly to it. Never build sections as top-level page children and reparent later — `appendChild` across calls silently fails.

**Use exact values from the style token table** — hex colours converted to 0–1 range, exact font families and sizes, exact padding and gap values, exact border radii.

#### Colour conversion

Hex → Figma 0–1 range:
```js
// Helper to use inline
function hex(h) {
  const n = parseInt(h.replace('#',''), 16);
  return { r: ((n>>16)&255)/255, g: ((n>>8)&255)/255, b: (n&255)/255 };
}

// Usage
frame.fills = [{ type: "SOLID", color: hex("#6366F1") }];
text.fills = [{ type: "SOLID", color: hex("#A1A1AA") }];
```

#### Stroke (border) pattern

```js
frame.strokes = [{ type: "SOLID", color: hex("#27272A") }];
frame.strokeWeight = 1;
frame.strokeAlign = "INSIDE";
```

#### Text node pattern

```js
await figma.loadFontAsync({ family: "Geist", style: "Bold" });
const heading = figma.createText();
heading.fontName = { family: "Geist", style: "Bold" };
heading.fontSize = 64;
heading.characters = "Ship faster with Acme";  // exact copy from page
heading.fills = [{ type: "SOLID", color: hex("#FFFFFF") }];
// Set line height if needed
heading.lineHeight = { unit: "PIXELS", value: 72 };
```

#### Image pattern

For images extracted from the page (hero images, logos, avatars, product screenshots), use `figma.createRectangle()` as a placeholder with the correct dimensions and a note label. If the image URL is accessible, use `figma.createImageAsync(url)`:

```js
// Placeholder (always works)
const imgPlaceholder = figma.createRectangle();
imgPlaceholder.resize(600, 400);
imgPlaceholder.fills = [{ type: "SOLID", color: { r: 0.15, g: 0.15, b: 0.15 } }];
imgPlaceholder.cornerRadius = 12;
imgPlaceholder.name = "Image: hero screenshot";

// Actual image (when URL is accessible and not auth-gated)
const image = await figma.createImageAsync("https://acme.com/hero.png");
const imageRect = figma.createRectangle();
imageRect.resize(600, 400);
imageRect.fills = [{ type: "IMAGE", scaleMode: "FILL", imageHash: image.hash }];
```

#### After appending a section
```js
wrapper.appendChild(section);
section.layoutSizingHorizontal = "FILL"; // MUST be after appendChild
```

---

### Step 6: Validate Each Section

After each section, call `get_screenshot` on that section's node ID (not just the full page). Check for:

- **Clipped text**: line heights or frame sizing cutting off descenders or whole lines
- **Wrong colours**: mismatched hex values — compare to the style token table
- **Wrong font**: wrong weight, wrong family, wrong size
- **Overlapping elements**: incorrect sizing or missing auto-layout
- **Missing content**: copy from the source page not yet in Figma
- **Images**: placeholders where the real image could be loaded

Fix issues with targeted scripts before moving to the next section.

---

### Step 7: Final Screenshot

After all sections, call `get_screenshot` on the full wrapper node. Compare against the source URL. Report:

1. Which sections were built
2. Any fonts that weren't available in Figma and what was substituted
3. Any images that couldn't be loaded (auth-gated, CORS-blocked, etc.) and remain as placeholders
4. Any content that couldn't be extracted from the page (dynamic/JS-rendered, behind auth)
5. The Figma file URL if a new file was created

---

## Common Section Patterns

### Navigation bar
```js
const nav = figma.createFrame();
nav.name = "Nav";
nav.layoutMode = "HORIZONTAL";
nav.primaryAxisAlignItems = "CENTER";
nav.counterAxisAlignItems = "CENTER";
nav.primaryAxisSizingMode = "FIXED";
nav.resize(1440, 10);
nav.counterAxisSizingMode = "AUTO";
nav.paddingLeft = 80;
nav.paddingRight = 80;
nav.paddingTop = 20;
nav.paddingBottom = 20;
nav.fills = [{ type: "SOLID", color: hex("#0A0A0A") }];
nav.strokes = [{ type: "SOLID", color: hex("#27272A") }];
nav.strokeWeight = 1;
nav.strokeAlign = "INSIDE";
```

### Hero section (centred, vertical)
```js
const hero = figma.createFrame();
hero.name = "Hero";
hero.layoutMode = "VERTICAL";
hero.primaryAxisAlignItems = "CENTER";
hero.counterAxisAlignItems = "CENTER";
hero.primaryAxisSizingMode = "AUTO";
hero.counterAxisSizingMode = "FIXED";
hero.resize(1440, 10);
hero.paddingTop = 96;
hero.paddingBottom = 96;
hero.paddingLeft = 80;
hero.paddingRight = 80;
hero.itemSpacing = 24;
hero.fills = [{ type: "SOLID", color: hex("#0A0A0A") }];
```

### Button (built from scratch — no component import)
```js
const btn = figma.createFrame();
btn.name = "Button / Primary";
btn.layoutMode = "HORIZONTAL";
btn.primaryAxisAlignItems = "CENTER";
btn.counterAxisAlignItems = "CENTER";
btn.primaryAxisSizingMode = "AUTO";
btn.counterAxisSizingMode = "AUTO";
btn.paddingTop = 10;
btn.paddingBottom = 10;
btn.paddingLeft = 20;
btn.paddingRight = 20;
btn.cornerRadius = 6;
btn.fills = [{ type: "SOLID", color: hex("#6366F1") }];

await figma.loadFontAsync({ family: "Geist", style: "Medium" });
const btnLabel = figma.createText();
btnLabel.fontName = { family: "Geist", style: "Medium" };
btnLabel.fontSize = 14;
btnLabel.characters = "Get started";
btnLabel.fills = [{ type: "SOLID", color: hex("#FFFFFF") }];
btn.appendChild(btnLabel);
```

### Card
```js
const card = figma.createFrame();
card.name = "Card";
card.layoutMode = "VERTICAL";
card.primaryAxisSizingMode = "AUTO";
card.counterAxisSizingMode = "FIXED";
card.resize(380, 10);
card.paddingTop = 32;
card.paddingBottom = 32;
card.paddingLeft = 32;
card.paddingRight = 32;
card.itemSpacing = 16;
card.cornerRadius = 12;
card.fills = [{ type: "SOLID", color: hex("#141414") }];
card.strokes = [{ type: "SOLID", color: hex("#27272A") }];
card.strokeWeight = 1;
card.strokeAlign = "INSIDE";
```

### 3-column grid (wrapping)
```js
const grid = figma.createFrame();
grid.name = "Grid";
grid.layoutMode = "HORIZONTAL";
grid.layoutWrap = "WRAP";
grid.primaryAxisSizingMode = "FIXED";
grid.resize(1200, 10);
grid.counterAxisSizingMode = "AUTO";
grid.itemSpacing = 24;
grid.counterAxisSpacing = 24;
grid.fills = [];
```

---

## Error Recovery

If a `use_figma` call fails, **stop immediately**. Failed scripts are atomic — nothing is written on error. Read the error, fix the script, retry.

| Error | Cause | Fix |
|-------|-------|-----|
| Font not found | Wrong style name | Call `figma.listAvailableFontsAsync()` to find exact name |
| `FILL` before parent | `layoutSizingHorizontal = "FILL"` set before `appendChild` | Move after `appendChild` |
| Wrapper node null | Wrong page context | Add `await figma.setCurrentPageAsync(page)` at top of script |
| Image load fails | Auth-gated or CORS-blocked URL | Use placeholder rectangle instead |
| Text truncated | Frame too narrow / fixed height | Set `primaryAxisSizingMode = "AUTO"` and remove fixed height |
