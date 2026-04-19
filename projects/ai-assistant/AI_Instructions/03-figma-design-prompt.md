# Figma Design Prompt — Project Dashboard

**Project:** project-dashboard  
**Created:** 2026-04-19  
**Type:** UI Design Specification for Figma  

---

## Overview

Design a clean, minimal web dashboard for a personal project tracker. The app reads projects from a GitHub repo and lets the user view and manage project statuses. It runs at `localhost:3000` (also deployable on Vercel). Single-user, no multi-tenant complexity.

---

## Design Style

- **Aesthetic:** Clean developer tool — think Linear, Vercel dashboard, or Raycast. Not corporate, not playful.
- **Color palette:** Light mode primary. Neutral grays for backgrounds, white cards, colored status badges only.
- **Typography:** Inter or Geist (system font). No decorative fonts.
- **Density:** Medium — not too spacious, not cramped. Cards should feel scannable.
- **Radius:** Subtle — 8px cards, 999px pills for badges.
- **Shadows:** Very light card shadow (`0 1px 3px rgba(0,0,0,0.08)`). No heavy drop shadows.

---

## Color Tokens

| Token | Value | Usage |
|-------|-------|-------|
| `bg-page` | `#F9FAFB` | Page background |
| `bg-card` | `#FFFFFF` | Card background |
| `border` | `#E5E7EB` | Card border, dividers |
| `text-primary` | `#111827` | Project name, headings |
| `text-secondary` | `#6B7280` | Description, dates, labels |
| `text-muted` | `#9CA3AF` | Placeholder, empty states |

### Status Badge Colors

| Status | Background | Text | Emoji |
|--------|-----------|------|-------|
| 🟢 Active | `#DCFCE7` | `#15803D` | 🟢 |
| 🔵 Planning | `#DBEAFE` | `#1D4ED8` | 🔵 |
| 🟡 Paused | `#FEF9C3` | `#A16207` | 🟡 |
| ✅ Complete | `#F3F4F6` | `#374151` | ✅ |
| ❌ Abandoned | `#FEE2E2` | `#B91C1C` | ❌ |

---

## Screens to Design

### Screen 1 — Main Dashboard (Default State)

**URL:** `/`

**Layout:**
- Full-width page, max content width `1152px`, centered, `32px` horizontal padding
- Top navbar (fixed or static)
- Stats bar below navbar
- Project card grid below stats

**Navbar:**
- Left: App name `Project Dashboard` (bold, `text-primary`)
- Right: `+ Add Project` button (primary filled button, dark background `#111827`, white text)
- Height: `64px`, white background, bottom border `1px solid #E5E7EB`

**Stats Bar:**
- Sits between navbar and grid
- 4 stat chips in a row: `Active: 1`, `Paused: 0`, `Complete: 0`, `Abandoned: 0`
- Each chip: small label in `text-secondary`, count in `text-primary` bold
- Separated by thin vertical dividers or just spacing

**Project Card Grid:**
- CSS Grid: 3 columns on desktop, 2 on tablet, 1 on mobile
- Gap: `20px`
- Cards sorted by `lastUpdated` descending

**Project Card anatomy:**
```
┌─────────────────────────────────────┐
│  [StatusBadge]              [⋯ menu] │  ← top row
│                                     │
│  Project Name                       │  ← bold, text-primary, 16px
│  Description text (2 lines max,     │  ← text-secondary, 14px
│  truncated with ellipsis)           │
│                                     │
│  #tag1  #tag2  #tag3                │  ← gray pills, 12px
│                                     │
│  Started: 2026-04-18                │  ← text-muted, 12px
│  Updated: 2026-04-18                │
└─────────────────────────────────────┘
```

- Card padding: `20px`
- Card border: `1px solid #E5E7EB`
- Card border-radius: `8px`
- Card background: white
- Hover state: border color shifts to `#D1D5DB`, very subtle lift shadow

**StatusBadge:**
- Pill shape (border-radius: `999px`)
- Padding: `4px 10px`
- Font: `12px`, medium weight
- Shows emoji + label: e.g. `🟢 Active`
- Clickable — opens StatusDropdown on click

**Tag pills:**
- Background: `#F3F4F6`
- Text: `#6B7280`
- Font: `12px`
- Padding: `2px 8px`
- Border-radius: `999px`

---

### Screen 2 — Status Dropdown (Open State)

Triggered by clicking the StatusBadge on any card.

**Dropdown:**
- Positioned absolutely below the StatusBadge
- Width: `180px`
- Background: white
- Border: `1px solid #E5E7EB`
- Border-radius: `8px`
- Box shadow: `0 4px 12px rgba(0,0,0,0.1)`
- 5 items, one per status

**Each dropdown item:**
- Height: `36px`
- Padding: `0 12px`
- Shows emoji + label: `🟢 Active`
- Current status has a `✓` checkmark on the right
- Hover: `bg-gray-50` (`#F9FAFB`)
- Font: `14px`, `text-primary`

---

### Screen 3 — Add Project Modal

Triggered by clicking `+ Add Project` in the navbar.

**Modal:**
- Centered overlay with dark backdrop (`rgba(0,0,0,0.4)`)
- Modal width: `480px`
- Background: white
- Border-radius: `12px`
- Padding: `28px`
- Box shadow: `0 20px 40px rgba(0,0,0,0.15)`

**Modal header:**
- Title: `New Project` (bold, `18px`, `text-primary`)
- Close `×` button top-right

**Form fields (top to bottom):**

1. **Project Name** — text input, required, placeholder: `e.g. my-api-service`
2. **Description** — textarea (3 rows), required, placeholder: `What does this project do? What does "done" look like?`
3. **Tags** — text input, placeholder: `#web #api #python (space separated)`
4. **Status** — select dropdown, options: Planning (default), Active, Paused, Complete, Abandoned

**Form field style:**
- Label: `13px`, `text-secondary`, `font-medium`, `4px` margin-bottom
- Input: `height: 40px`, border `1px solid #E5E7EB`, border-radius `6px`, padding `0 12px`, font `14px`
- Focus state: border `#111827`, outline none
- Textarea: same border style, `padding: 10px 12px`, resize vertical

**Modal footer:**
- Right-aligned buttons: `Cancel` (ghost/outline) + `Create Project` (primary filled, dark)
- `Create Project` shows loading spinner while API call is in progress

---

### Screen 4 — Loading State

Shown on initial page load while `GET /api/projects` is in flight.

- Replace each project card with a skeleton card
- Same card dimensions as real cards
- Skeleton elements: animated shimmer (`bg-gray-200` with shimmer animation)
  - Top-left: short wide bar (status badge placeholder)
  - Middle: two lines (title + description)
  - Bottom: three short bars (tags)
- Show 3 skeleton cards in the grid

---

### Screen 5 — Error State (Inline Card Error)

Shown when a status change PATCH fails.

- A red error banner appears at the bottom of the affected card
- Banner: `bg-red-50`, `border-t border-red-200`, `text-red-700`, `12px`
- Text: `Failed to update status. Changes reverted.`
- Auto-dismisses after 4 seconds (show dismiss `×` button too)
- Card status badge reverts to previous value immediately

---

### Screen 6 — Empty State

Shown when no projects exist yet (fresh install).

- Centered in the grid area
- Icon: a simple folder or grid icon (outline style, `48px`, `text-muted`)
- Heading: `No projects yet` (`text-primary`, `16px`, bold)
- Subtext: `Click "+ Add Project" to create your first project.` (`text-secondary`, `14px`)

---

## Component Inventory (for Figma Components Panel)

| Component | Variants |
|-----------|----------|
| `StatusBadge` | Planning, Active, Paused, Complete, Abandoned |
| `StatusDropdown` | Open (each status as current) |
| `ProjectCard` | Default, Hover, Error state, Loading skeleton |
| `TagPill` | Default |
| `Button` | Primary (dark), Ghost/Outline, Loading state |
| `TextInput` | Default, Focus, Error |
| `Textarea` | Default, Focus |
| `SelectInput` | Default, Open |
| `Modal` | Add Project (empty, filled, loading) |
| `Navbar` | Default |
| `StatsBar` | Default |
| `EmptyState` | Default |
| `SkeletonCard` | Default |
| `ErrorBanner` | Default |

---

## Responsive Breakpoints

| Breakpoint | Grid Columns | Notes |
|------------|-------------|-------|
| Desktop `≥1024px` | 3 columns | Full layout |
| Tablet `768–1023px` | 2 columns | Navbar stays same |
| Mobile `<768px` | 1 column | `+ Add Project` becomes icon-only or moves |

---

## Figma File Structure (Suggested Pages)

1. **🎨 Design System** — colors, typography, spacing tokens, all components
2. **📱 Desktop — Main** — Screen 1 (default), Screen 2 (dropdown open), Screen 4 (loading), Screen 6 (empty)
3. **📱 Desktop — Modal** — Screen 3 (empty form, filled form, loading state)
4. **📱 Desktop — Error** — Screen 5 (inline card error)
5. **📱 Mobile** — Mobile layout of main screen + modal

---

## Interaction Notes (for Figma Prototyping)

- StatusBadge click → opens StatusDropdown (overlay component)
- StatusDropdown item click → closes dropdown, badge updates to new status
- `+ Add Project` click → opens modal overlay
- Modal backdrop click or `×` click → closes modal
- Modal `Create Project` click → shows loading state on button
- Card hover → subtle border + shadow change

---

## What NOT to Design

- No sidebar navigation (single page app)
- No dark mode (out of scope for MVP)
- No project detail page (Phase 2)
- No filter/search bar (Phase 2)
- No authentication/login screen (handled by NextAuth, not in scope for UI design)
