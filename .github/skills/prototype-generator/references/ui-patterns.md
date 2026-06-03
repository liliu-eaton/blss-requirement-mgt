# UI Patterns Reference

When the requirement calls for these patterns, use the following structure:

## Configuration Card

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

### CSS Pattern
- Card container: `border-radius: 8px; box-shadow: 0 1px 3px rgba(0,0,0,0.12); background: #fff;`
- Card header: `background: #F4F5F7; padding: 16px 20px; border-bottom: 1px solid #E0E0E0;`
- Info grid: `display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; padding: 20px;`
- Tags: `display: inline-block; padding: 4px 8px; border-radius: 4px; font-size: 12px;`

## Data Table

```
┌─ Header Row (gray bg, uppercase labels) ─────────────┐
│  Col1  │  Col2  │  Col3  │  Col4  │  Col5           │
├────────┼────────┼────────┼────────┼─────────────────┤
│  data  │  data  │  data  │  data  │  data           │
└────────┴────────┴────────┴────────┴─────────────────┘
Empty state: centered gray text with icon
```

### CSS Pattern
- Table: `width: 100%; border-collapse: collapse;`
- Header: `background: #F4F5F7; text-transform: uppercase; font-size: 11px; font-weight: 600; color: #666; padding: 12px 16px;`
- Row: `border-bottom: 1px solid #E8E8E8; padding: 12px 16px;`
- Row hover: `background: #F8F9FA;`
- Empty state: `text-align: center; padding: 48px; color: #999;`

## Confirmation Modal

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

### CSS Pattern
- Overlay: `position: fixed; inset: 0; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 1000;`
- Modal: `background: #fff; border-radius: 8px; padding: 24px; max-width: 480px; width: 90%;`
- Warning banner: `background: #FFF3E0; border-left: 4px solid #FF9800; padding: 12px 16px; border-radius: 4px;`
- Button row: `display: flex; justify-content: flex-end; gap: 12px; margin-top: 24px;`
- Cancel button: `padding: 8px 16px; border: 1px solid #ccc; border-radius: 4px; background: #fff;`
- Confirm button: `padding: 8px 16px; border: none; border-radius: 4px; background: #0057B8; color: #fff;`

## Toast/Snackbar

```
┌──────────────────────────────────────────┐
│  ✓ Success message text          [✕]    │
└──────────────────────────────────────────┘
```

### CSS Pattern
- Container: `position: fixed; bottom: 24px; left: 50%; transform: translateX(-50%); z-index: 2000;`
- Toast: `padding: 12px 20px; border-radius: 4px; color: #fff; display: flex; align-items: center; gap: 8px;`
- Success: `background: #2E7D32;`
- Error: `background: #C62828;`
- Warning: `background: #E65100;`
- Auto-dismiss: 4 seconds with fade-out animation

## Form Controls

### Toggle Switch
```css
.toggle { width: 40px; height: 20px; border-radius: 10px; background: #ccc; position: relative; cursor: pointer; }
.toggle.active { background: #0057B8; }
.toggle::after { content: ''; width: 16px; height: 16px; border-radius: 50%; background: #fff; position: absolute; top: 2px; left: 2px; transition: 0.2s; }
.toggle.active::after { left: 22px; }
```

### Dropdown
```css
.dropdown { padding: 8px 12px; border: 1px solid #ccc; border-radius: 4px; background: #fff; min-width: 200px; appearance: none; background-image: url("data:image/svg+xml,...chevron..."); background-repeat: no-repeat; background-position: right 12px center; }
.dropdown:focus { border-color: #0057B8; outline: none; box-shadow: 0 0 0 2px rgba(0,87,184,0.2); }
```

### Text Input
```css
.input { padding: 8px 12px; border: 1px solid #ccc; border-radius: 4px; width: 100%; }
.input:focus { border-color: #0057B8; outline: none; box-shadow: 0 0 0 2px rgba(0,87,184,0.2); }
.input.error { border-color: #C62828; }
.input-error-text { color: #C62828; font-size: 12px; margin-top: 4px; }
```

## Tooltip

```css
.tooltip-trigger { position: relative; cursor: help; }
.tooltip-trigger:hover .tooltip-content { display: block; }
.tooltip-content { display: none; position: absolute; bottom: 100%; left: 50%; transform: translateX(-50%); background: #333; color: #fff; padding: 8px 12px; border-radius: 4px; font-size: 12px; white-space: nowrap; margin-bottom: 8px; }
```
