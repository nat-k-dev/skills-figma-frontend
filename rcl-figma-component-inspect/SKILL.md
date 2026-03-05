---
name: rcl-figma-component-inspect
description: Use this skill when the user wants to analyze a Figma component before implementing it in code. Extracts variants, colors (with full token chains), spacing, border-radius, and typography. Results in a structured spec ready for implementation.
version: 1.0.0
last_updated: 2026-03-05
project: "@junioreinsteinbv/react-component-library"
---

# Figma Component Inspection

**This skill only applies to the project with `"name": "@junioreinsteinbv/react-component-library"` in `package.json`.** Verify this before proceeding.

Produces a complete implementation spec from a Figma component: variants/properties, pixel-perfect colors (with token chains), spacing, border-radius, and typography. Always run this before `create-component`.

## CRITICAL: Never use Figma REST API

Always use `figma_execute` via Desktop Bridge. Never use `figma_get_file_data`, `figma_get_component`, or any REST-based tool — they cannot resolve variable alias chains or detect per-fill visibility.

---

## Step 0 — Check connection

Call `figma_get_status`. If `setup.valid === false`, stop and ask the user to open the Desktop Bridge plugin in Figma Desktop.

---

## Step 1 — Find the COMPONENT_SET node

**Critical:** The node ID in the Figma URL is often NOT the component set — it is usually a parent frame. You must traverse the page to find the actual `COMPONENT_SET`.

```javascript
await figma.loadAllPagesAsync();
const page = figma.root.children.find(p => p.name.includes('TARGET_PAGE_NAME'));
// Traverse children to find COMPONENT_SET
function findComponentSet(node) {
  if (node.type === 'COMPONENT_SET') return node;
  if (node.children) {
    for (const c of node.children) {
      const found = findComponentSet(c);
      if (found) return found;
    }
  }
  return null;
}
const componentSet = findComponentSet(page);
```

If there are multiple component sets on the page, identify the correct one by name.

---

## Step 2 — Extract variants and component properties

```javascript
return {
  name: componentSet.name,
  componentProperties: componentSet.componentPropertyDefinitions,
  variants: componentSet.children.map(c => c.name),
};
```

Note which properties exist, their types (VARIANT, BOOLEAN, TEXT, INSTANCE_SWAP), default values, and all options.

**Key observation:** If `Shape` only has one value (e.g., `Rounded`), it is not worth exposing as a prop — hardcode it.

---

## Step 3 — Extract colors for every variant

Use the full CSS color extraction script from the `figma-css-colors` skill on the component set node. This resolves full variable alias chains.

Run with `timeout: 30000`.

### Tailwind mapping rules

When you see a Figma primitive token name, map it directly to a Tailwind class:

| Figma primitive token | Tailwind class |
|-----------------------|----------------|
| `Colors/Gray/800` | `gray-800` |
| `Colors/Sky/600` | `sky-600` |
| `Colors/Blue/100` | `blue-100` |
| `Colors/White/100%` | `white` |
| `Colors/White/10%` | `white/10` (opacity modifier) |
| `Colors/Yellow/800` | `yellow-800` |

If a token resolves through an alias with a non-standard name (e.g., `Text/Brand Color`), follow the chain to the primitive and use the primitive's name. If still unclear, ask the user explicitly.

If the hex value is hardcoded (no token binding), find the closest Tailwind color by hex. Do not silently guess — if unsure about a specific combination, ask.

### Border handling

- **Solid type**: bg/text only, no visible border → `border-transparent`
- **Soft type**: bg/text + colored border → set in compound variants
- **White type**: always the same combination → put all styles directly in the `type` variant, no compound variants needed

---

## Step 4 — Extract spacing, padding, border-radius

Inspect the first variant's `_inner` instance (the actual styled layer, not the component wrapper):

```javascript
const variant = componentSet.children[0];
const inner = variant.children[0]; // usually named _alert, _badge, etc.
return {
  padding: { top: inner.paddingTop, right: inner.paddingRight, bottom: inner.paddingBottom, left: inner.paddingLeft },
  itemSpacing: inner.itemSpacing,       // gap between icon and content
  cornerRadius: inner.cornerRadius,
  strokeWeight: inner.strokeWeight,
  // also inspect children for nested gaps
  contentGap: inner.children[0]?.itemSpacing, // e.g. gap between title and description
};
```

Map to Tailwind:
- padding 16px → `p-4`, padding 8px → `p-2`, etc.
- gap 20px → `gap-5`, gap 4px → `gap-1`
- cornerRadius 12px → `rounded-xl`, 8px → `rounded-lg`
- strokeWeight 1 → `border`

Also check if there is extra padding on internal frames (e.g., `pr-36` on the title row = space reserved for a dismiss button → `pr-9` on root when action slot is present).

---

## Step 5 — Extract typography from EXAMPLE INSTANCES

**Critical:** Font sizes and weights on component set variants are often placeholder defaults. Always check real example instances on the examples page for accurate values.

Find the examples frame (usually named `*Examples*` alongside the main docs frame):

```javascript
const examplesFrame = page.children.find(c => c.name.includes('Examples') || c.name.includes('example'));
// Find an Alert instance that has both title and description
function findById(node, id) { ... }
const exampleAlert = findById(examplesFrame, 'KNOWN_INSTANCE_ID');

function findTextNodes(node, results = []) {
  if (node.type === 'TEXT') {
    results.push({
      name: node.name,
      characters: node.characters,
      fontSize: node.fontSize,
      fontWeight: node.fontWeight,
      fontStyle: node.fontName?.style,
      lineHeight: node.lineHeight,
    });
  }
  if (node.children) node.children.forEach(c => findTextNodes(c, results));
  return results;
}
return findTextNodes(exampleAlert);
```

Map font weights: 400=`font-normal`, 500=`font-medium`, 600=`font-semibold`, 700=`font-bold`.
Map font sizes: 12px=`text-xs`, 14px=`text-sm`, 16px=`text-base`, 18px=`text-lg`, 20px=`text-xl`.

---

## Step 6 — Identify sub-components and layout strategy

From the internal structure, identify:
- Is there an icon slot? (optional SVG child)
- Is there a title slot?
- Is there a description/body slot?
- Is there an action slot? (dismiss button, absolute positioned)

For compound components with icon + title + description, use **CSS grid** with `has-[>svg]` detection instead of requiring a wrapper component. This matches the shadcn pattern:

```
'has-[>svg]:grid-cols-[auto_1fr]'
'*:[svg]:row-span-2 *:[svg]:self-start *:[svg]:mt-0.5 *:[svg]:shrink-0'
```

This eliminates the need for a `<Content>` wrapper sub-component.

---

## Step 7 — Output a structured spec

Summarize findings as a spec table:

```
COMPONENT: Alert
COMPONENT SET NODE: 4325:59048

VARIANTS (Figma props):
  Type:  solid | soft | white
  Color: dark | gray | green | blue | red | yellow | light
  Shape: rounded (hardcode — one value only)

LAYOUT (base):
  rounded-xl  p-4  border  gap-x-5  gap-y-1
  grid layout: has-[>svg]:grid-cols-[auto_1fr]
  action padding: has-[[data-slot=alert-action]]:pr-9

TYPOGRAPHY:
  Title:       text-lg font-semibold
  Description: text-base font-normal

COLORS:
  solid/dark:   bg-gray-800   text-white         (no border)
  solid/gray:   bg-gray-500   text-white         (no border)
  ...
  soft/green:   bg-green-100  text-green-800  border-green-200
  ...
  white:        bg-white      text-gray-800   border-gray-200

SUB-COMPONENTS:
  Alert        — root div, role="alert", data-slot="alert"
  AlertTitle   — div, data-slot="alert-title"
  AlertDescription — div, data-slot="alert-description"
  AlertAction  — div, data-slot="alert-action", absolute top-2.5 right-3
```

This spec is the input to the `create-component` skill.
