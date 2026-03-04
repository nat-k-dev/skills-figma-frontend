---
name: figma-css-colors
description: Use this skill when the user asks to extract CSS color styles from a selected Figma item, inspect colors or tokens on a selected node, show token chains, or asks "what colors does this have", "extract css colors", "show me the token chain for colors", or similar Figma color inspection tasks.
version: 1.0.0
---

# Figma CSS Color Extraction

This skill extracts accurate, complete CSS color information from the currently selected Figma node, including full design token alias chains, while filtering out invisible noise.

## CRITICAL: Never use Figma REST API

**Never use `figma_get_file_data`, `figma_get_component`, `figma_get_styles`, or any other tool that calls the Figma REST API** to inspect design data when a user provides a Figma link or asks about colors/tokens. The REST API cannot resolve variable alias chains, cannot detect per-fill visibility, and returns stale cloud data.

Always use `figma_execute` (via the Desktop Bridge) for all color and token extraction.

## Step 0 — Guard: check Desktop Bridge connection

Before doing anything, call `figma_get_status` and verify `setup.valid === true`.

If the Desktop Bridge is **not connected** (`setup.valid === false`):
- **Stop immediately.** Do not attempt any Figma tool calls.
- Tell the user: "The Figma Desktop Bridge plugin is not connected. Please open Figma Desktop, go to **Plugins → Development → Figma Desktop Bridge**, and run the plugin until you see 'MCP ready'. Then try again."
- Do not proceed until the user confirms the plugin is running.

## Step 1 — Get the selection

```
figma_get_selection → extract node id
```

## Step 2 — Run figma_execute with the extraction script below

Use `timeout: 30000` because variable resolution is async and may be slow on deep trees.

### Complete extraction script

```javascript
const node = figma.currentPage.findOne(n => n.id === 'NODE_ID_HERE');
if (!node) return { error: 'Node not found' };

function toHex(r, g, b) {
  return '#' + [r, g, b].map(v => Math.round(v * 255).toString(16).padStart(2, '0')).join('');
}

async function resolveChain(variableId) {
  const chain = [];
  let currentId = variableId;
  while (currentId) {
    const variable = await figma.variables.getVariableByIdAsync(currentId);
    if (!variable) break;
    const collection = await figma.variables.getVariableCollectionByIdAsync(variable.variableCollectionId);
    const modeId = collection?.defaultModeId;
    const value = modeId ? variable.valuesByMode[modeId] : null;
    const entry = { name: variable.name, collection: collection?.name };
    if (value && typeof value === 'object' && value.type === 'VARIABLE_ALIAS') {
      entry.value = '→ alias';
      chain.push(entry);
      currentId = value.id;
    } else {
      entry.value = (value && typeof value === 'object' && 'r' in value)
        ? toHex(value.r, value.g, value.b)
        : value;
      chain.push(entry);
      break;
    }
  }
  return chain;
}

async function extractVisible(n) {
  // Skip the entire layer if it's hidden
  if (n.visible === false) return null;

  const result = { name: n.name, type: n.type, colors: {} };
  const bound = n.boundVariables || {};

  // Fills → background-color (or color for TEXT nodes)
  if ('fills' in n && Array.isArray(n.fills)) {
    for (let i = 0; i < n.fills.length; i++) {
      const f = n.fills[i];
      // Skip invisible fills — Figma allows per-fill visibility toggle
      if (f.visible === false || f.type !== 'SOLID') continue;
      const cssKey = n.type === 'TEXT' ? 'color' : 'background-color';
      const varId = bound.fills?.[i]?.id;
      result.colors[cssKey] = {
        hex: toHex(f.color.r, f.color.g, f.color.b),
        tokenChain: varId ? await resolveChain(varId) : null
      };
    }
  }

  // Strokes → border-color
  if ('strokes' in n && Array.isArray(n.strokes)) {
    for (let i = 0; i < n.strokes.length; i++) {
      const s = n.strokes[i];
      if (s.visible === false || s.type !== 'SOLID') continue;
      const varId = bound.strokes?.[i]?.id;
      result.colors['border-color'] = {
        hex: toHex(s.color.r, s.color.g, s.color.b),
        tokenChain: varId ? await resolveChain(varId) : null
      };
    }
  }

  // Recurse into children
  if ('children' in n) {
    const children = [];
    for (const child of n.children) {
      const c = await extractVisible(child);
      if (c) children.push(c);
    }
    if (children.length) result.children = children;
  }

  return result;
}

return await extractVisible(node);
```

## Step 3 — Format the output

Present results as a tree with CSS properties and token chains indented below each node. Use this format:

```
NodeName  (TYPE)
  background-color: #hex
    → Collection › Token/Path/Name
      → Collection › Token/Path/Name
        → Primitives › Colors/Palette/500
          → #hex (resolved)
  border-color: #hex
    → ...

└── ChildName  (TYPE)
      color: #hex
        → ...
```

If `tokenChain` is null, note it as **no token binding (hardcoded)**.

## Key rules for correctness

### Visibility filtering

1. **Layer-level**: Skip any node where `n.visible === false` — entire subtree is invisible.
2. **Fill/stroke-level**: Skip individual paint entries where `f.visible === false`. In Figma, each fill in the fills array has its own eye toggle independent of the layer. This is a common source of "ghost" colors.

### CSS property mapping

| Figma node type | Paint type | CSS property |
|---|---|---|
| Any except TEXT | fills | `background-color` |
| TEXT | fills | `color` |
| Any | strokes | `border-color` |

### Token chain resolution

- Always use `getVariableByIdAsync` (NOT `getVariableById`) — sync version throws in `dynamic-page` document access mode.
- A variable value of `{ type: 'VARIABLE_ALIAS', id: '...' }` means it points to another variable — keep following the chain until you reach a raw value (hex color, number, string).
- Use `collection.defaultModeId` to read values — this gives the base/light mode value.
- Each step in the chain has: `name` (token path), `collection` (collection name), `value` (hex or "→ alias").

### Identifying swappable wrapper layers (noise vs. signal)

An INSTANCE layer is a **swappable wrapper** (not a direct color contributor) when ALL of these are true:
- `type === 'INSTANCE'`
- All fills have `visible: false`
- `blendMode === 'PASS_THROUGH'` (transparent to parent)
- No strokes
- No `boundVariables` for fills

These layers exist for component reuse and icon-swap flexibility — their invisible fills are default component fills that were switched off. Report them in the tree but note they have no visible color contribution. The actual colors live in their `VECTOR` children.

### What VECTOR strokes represent

For icon/SVG layers (`type === 'VECTOR'`), strokes are the drawn line color — treat as the icon's foreground `color` in CSS terms, even though Figma calls it `border-color`. You may annotate this in the output: `border-color (icon stroke): #ffffff`.
