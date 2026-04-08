---
name: figma-mcp-guide
description: "Use when a project has Figma designs or when setting up Figma MCP for the first time — guides design token extraction, component hierarchy mapping, and visual baseline generation via Figma MCP tools"
---

# Figma MCP Guide

Extract design data from Figma for testing and implementation using cursor-talk-to-figma-mcp.

## When to Use

Invoked during brainstorming when the user confirms they have Figma designs (or wants to set up Figma for a greenfield project).

## Step 0: Check Setup

Is Figma MCP configured?

```bash
# Look for Figma MCP in .mcp.json
cat .mcp.json 2>/dev/null | jq '.mcpServers["TalkToFigma"]'
```

**If not configured**, guide the user:

1. Install the Figma plugin: search "Cursor Talk To Figma MCP" in Figma Community
2. Add to `.mcp.json`:
   ```json
   {
     "mcpServers": {
       "TalkToFigma": {
         "command": "npx",
         "args": ["-y", "cursor-talk-to-figma-mcp@latest"]
       }
     }
   }
   ```
3. Open Figma, run the plugin, note the channel name
4. Verify: `join_channel` with the channel name

**If already configured**, proceed to Step 1.

## Step 1: Connect

```
join_channel({ channel: "<channel-name-from-figma-plugin>" })
get_document_info()  → understand file structure, pages, top-level frames
```

Ask the user which page/frame contains the designs to extract.

## Step 2: Extract Design Tokens

```
get_styles()
→ Returns: {
    colors: [{ id, name, key, paint }],
    texts:  [{ id, name, key, fontSize, fontName }],
    effects: [{ id, name, key }],
    grids:  [{ id, name, key }]
  }
```

Transform into `design-tokens.json`:
```json
{
  "colors": {
    "primary": "#3b82f6",
    "secondary": "#6b7280",
    "error": "#ef4444",
    "surface": "#ffffff",
    "nav-bg": "#1c1b1b",
    "ticker-accent": "#972500"
  },
  "typography": {
    "heading1": { "fontFamily": "Inter", "fontSize": "32px", "fontWeight": "700", "lineHeight": "40px" },
    "heading2": { "fontFamily": "Inter", "fontSize": "24px", "fontWeight": "600", "lineHeight": "32px" },
    "body": { "fontFamily": "Inter", "fontSize": "16px", "fontWeight": "400", "lineHeight": "24px" },
    "caption": { "fontFamily": "Inter", "fontSize": "12px", "fontWeight": "400", "lineHeight": "16px" }
  },
  "spacing": { "xs": "4px", "sm": "8px", "md": "16px", "lg": "24px", "xl": "32px" },
  "radii": { "sm": "4px", "md": "8px", "lg": "16px", "full": "9999px" }
}
```

Save to: `e2e/fixtures/design-tokens.json`

## Step 3: Map Component Hierarchy

```
scan_nodes_by_types({
  nodeId: "<page-or-frame-id>",
  types: ["FRAME", "COMPONENT", "INSTANCE"]
})
→ Returns: [{ id, name, type, bbox: { x, y, width, height } }]
```

For each key component, get detailed properties:
```
get_node_info({ nodeId: "<component-id>" })
→ Returns: {
    id, name, type,
    fills: [{ type, color (hex), opacity }],
    fontFamily, fontSize, fontWeight, lineHeightPx,
    cornerRadius,
    absoluteBoundingBox: { x, y, width, height }
  }
```

Transform into `component-map.json`:
```json
{
  "Header": {
    "selector": "header",
    "children": ["Logo", "Nav", "Search"],
    "tokens": {
      "background-color": "#1c1b1b",
      "height": "64px"
    }
  },
  "ArticleCard": {
    "selector": ".article-card",
    "children": ["Image", "Title", "Meta", "Tags"],
    "tokens": {
      "border-radius": "8px",
      "padding": "16px"
    }
  }
}
```

Save to: `e2e/fixtures/component-map.json`

## Step 4: Generate Visual Baselines (Optional)

```
export_node_as_image({
  nodeId: "<screen-id>",
  format: "png",
  scale: 1
})
→ Returns: base64-encoded PNG
```

Save to: `e2e/__screenshots__/<screen-name>.png`

Use these as Playwright `toHaveScreenshot()` baselines. Note: Figma renders may differ slightly from browser renders — update baselines after first Playwright run.

## Step 5: Mapping Rules

Use this table when writing Playwright test assertions:

| Figma Property | CSS Property | Playwright Assertion |
|---|---|---|
| `fills[0].color` | `background-color` | `getComputedStyle(el).backgroundColor` |
| `fontSize` | `font-size` | `getComputedStyle(el).fontSize` |
| `fontFamily` | `font-family` | `getComputedStyle(el).fontFamily` |
| `fontWeight` | `font-weight` | `getComputedStyle(el).fontWeight` |
| `lineHeightPx` | `line-height` | `getComputedStyle(el).lineHeight` |
| `cornerRadius` | `border-radius` | `getComputedStyle(el).borderRadius` |
| `itemSpacing` | `gap` | `getComputedStyle(el).gap` |
| `padding` | `padding` | `getComputedStyle(el).padding` |

**Color format conversion:** Figma returns hex (`#3b82f6`). `getComputedStyle()` returns rgb (`rgb(59, 130, 246)`). Convert hex to rgb for assertions or use a helper:

```typescript
function hexToRgb(hex: string): string {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return `rgb(${r}, ${g}, ${b})`;
}
```

## MCP Tools Reference

| Tool | Purpose | Use When |
|---|---|---|
| `join_channel` | Connect to Figma plugin | Always first |
| `get_document_info` | File structure overview | Understand pages/frames |
| `get_styles` | Color, text, effect, grid styles | Extract design tokens |
| `scan_nodes_by_types` | Find components by type recursively | Map component hierarchy |
| `get_node_info` | Detailed node properties (fills, fonts, spacing) | Get exact values per component |
| `scan_text_nodes` | All text with font properties | Typography audit |
| `export_node_as_image` | PNG/SVG export | Visual regression baselines |
| `get_selection` | Currently selected nodes | Interactive exploration |

## Worked Example: auto-blog-verify-gate

The auto-blog-verify-gate project uses this pattern in `e2e/tests/homepage.spec.ts`:
- Nav background: `#1c1b1b` (dark) → asserted via `getComputedStyle().backgroundColor`
- Ticker accent: `#972500` (rust) → asserted via `getComputedStyle().color`
- Design tokens extracted from Figma, hardcoded in test file
- With this skill, tokens would instead come from `e2e/fixtures/design-tokens.json`
