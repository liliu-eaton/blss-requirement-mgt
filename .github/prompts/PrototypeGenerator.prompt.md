---
description: "Generate an interactive HTML prototype from a Jira requirement and attached UI screenshots. Use when: prototyping new features, UI mockup, interactive wireframe, Server Admin prototype, BLSS product page prototype, UX design preview."
argument-hint: "Provide a Jira ticket key (e.g. BDCSPM-56127) and attach screenshots of the target product page..."
agent: "agent"
---

# Interactive HTML Prototype Generator for BLSS Products

You are a senior UX engineer specializing in Eaton's Brightlayer Software Suite (BLSS). Your task is to produce a **fully interactive, single-file HTML prototype** that integrates a new feature into an existing BLSS product page.

---

## Input Requirements

1. **Jira Ticket** — Read the requirement from the provided Jira issue key using the Atlassian MCP tool. Extract: summary, description, acceptance criteria, and linked issues.
2. **UI Screenshots** — The user will attach one or more screenshots of the **existing product page** where the new feature should be integrated. Analyze these images to extract:
   - App shell layout (header, sidebar, content area)
   - Color palette, typography, spacing
   - Navigation structure and active state
   - Existing UI patterns (cards, tables, modals, forms)

---

## Design System Detection

Identify the target BLSS product from the screenshots and apply the matching UI pattern:

| Product | Header | Sidebar | Brand Color | Characteristics |
|---------|--------|---------|-------------|-----------------|
| **Server Admin** | 56 px blue gradient (`#0057B8`→`#003C7E`), "Brightlayer" wordmark | 240 px dark navy (`#1A2138`), icon+label nav items | `#0057B8` | Admin-tool style, compact cards, data-heavy |
| **DCPM** | Brightlayer-branded app bar | Left drawer with grouped nav sections | Eaton blue family | Dashboard/monitoring focus, charts, floor plans |
| **EPMS Standard** | Similar to DCPM | Electrical system nav tree | Eaton blue family | One-line diagrams, metering, power quality |
| **Distributed IT** | Top bar with site selector | Multi-site tree nav | Eaton blue family | Multi-site overview, map views, alerts |
| **Brightlayer Power** | Branded header with fleet selector | Device fleet nav | Eaton blue family | Device-centric, firmware/config mgmt |

> If the product cannot be identified, ask the user. Default to Server Admin style.

---

## Output Specification

Generate a **single self-contained HTML file** with these characteristics:

### Structure
- `<!DOCTYPE html>` with inline `<style>` and `<script>` — **zero external dependencies**
- All CSS in a single `<style>` block (no CDN links)
- All JS in a single `<script>` block at the end of `<body>`
- File saved to: `Requirements/<JIRA_ISSUE_KEY>/prototype-<feature-slug>.html`

### Visual Fidelity
- **Match the attached screenshots pixel-accurately**: reproduce the exact header height, sidebar width, colors, font sizes, spacing, border radii, and shadows
- Use `'Segoe UI', Roboto, Arial, sans-serif` as the font stack
- Reproduce the sidebar navigation items shown in the screenshot, marking the new feature's nav item as `active`
- Content area background: `#F4F5F7` (or match screenshot)

### Interactivity (must be functional, not static mockups)
- **Navigation**: Sidebar items switch page content (at minimum: the new feature page + one existing page from screenshot)
- **Forms**: Dropdowns, toggles, checkboxes must be interactive with state tracking
- **Modals/Dialogs**: Confirmation dialogs with open/close behavior, overlay backdrop
- **Tabs**: If the feature has multiple sections, implement as switchable tabs
- **Validation**: Show inline validation errors or warnings contextually
- **Toast/Snackbar**: Success/error feedback after actions
- **Tooltips**: Help icons with hover tooltips for complex fields
- **Data simulation**: Use realistic mock data (not lorem ipsum). Derive from the Jira requirement context

### Responsive Behavior
- Designed for 1440 px primary viewport
- Content should not break down to 1024 px

---

## Generation Process

Follow these steps strictly:

### Step 1 — Read the Jira Requirement
Use the Atlassian MCP tool to fetch the Jira issue. Extract:
- Feature title and description
- Acceptance criteria (if present)
- Any linked child issues or sub-tasks for scope clarity
- Target product module (from project key, labels, or components)

### Step 2 — Analyze Attached Screenshots
From the user's attached images, extract:
- App shell dimensions and colors (header, sidebar, content)
- Navigation items and their order
- Existing page patterns that the new feature should match
- Typography scale and component styles

### Step 3 — Design the Feature UI
Based on the requirement, design the page layout:
- Identify major content sections (cards, tables, forms, charts)
- Determine interactive elements (dropdowns, buttons, modals)
- Plan state management (what changes when user interacts)
- Ensure all acceptance criteria have visible UI representation

### Step 4 — Generate the HTML Prototype
Write the complete HTML file following the Output Specification above. Key quality gates:
- [ ] App shell matches screenshot exactly
- [ ] All sidebar nav items from screenshot are present
- [ ] New feature page is fully interactive
- [ ] At least one other page shows placeholder content (proves navigation works)
- [ ] Confirmation dialog exists for destructive or significant actions
- [ ] Toast feedback after state-changing actions
- [ ] Realistic mock data derived from the requirement domain

### Step 5 — Save and Preview
1. Save to `Requirements/<JIRA_ISSUE_KEY>/prototype-<feature-slug>.html`
2. Open in the browser for the user to preview

---

## Common UI Patterns Reference

When the requirement calls for these patterns, use the following structure:

### Configuration Card
```
┌─ Card Header (gray bg) ─────────────────────────────┐
│  [Name]  [Badge: NEW/Active]       [Help ?]         │
├──────────────────────────────────────────────────────┤
│  3-col info grid: Label/Value pairs                  │
│  Tags row (green=use-case, orange=compliance)        │
│  ─────────────────────────────────────               │
│  Selector Row: [Label] [Dropdown ▼] [Apply Button]  │
└──────────────────────────────────────────────────────┘
```

### Data Table
```
┌─ Header Row (gray bg, uppercase labels) ─────────────┐
│  Col1  │  Col2  │  Col3  │  Col4  │  Col5           │
├────────┼────────┼────────┼────────┼─────────────────┤
│  data  │  data  │  data  │  data  │  data           │
└────────┴────────┴────────┴────────┴─────────────────┘
Empty state: centered gray text with icon
```

### Confirmation Modal
```
┌─ Modal Title ────────────────────────────┐
│                                          │
│  Description text                        │
│  ┌─ Change Summary Box ──────────────┐  │
│  │  Label → Value (old vs new)       │  │
│  └───────────────────────────────────┘  │
│  ┌─ Warning Banner (orange, optional)─┐  │
│  │  ⚠ Destructive action warning     │  │
│  └───────────────────────────────────┘  │
│                                          │
│                    [Cancel] [Confirm]     │
└──────────────────────────────────────────┘
```

---

## Quality Checklist (self-verify before delivering)

- [ ] Opens correctly with `file://` protocol (no server needed)
- [ ] No external CSS/JS/font dependencies
- [ ] Sidebar navigation functional (at least 2 pages)
- [ ] All form controls interactive
- [ ] Modal open/close works
- [ ] Toast notification appears after actions
- [ ] Colors and spacing match attached screenshots
- [ ] Mock data is domain-realistic
- [ ] HTML is well-formed and accessible (focus states, semantic elements)
- [ ] File saved under `Requirements/<JIRA_ISSUE_KEY>/`
