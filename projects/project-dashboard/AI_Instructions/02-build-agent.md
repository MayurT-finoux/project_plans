---
name: Build Agent Instructions
description: Complete step-by-step instructions for building the project-dashboard Next.js app
type: reference
---

# Build Agent — Full Implementation Instructions

**Before starting:** Read `01-context.md` in this folder. It covers the submodule path, markdown formats, and git rules this app depends on.

**Reference files to read via submodule:**
- `docs/plans/projects/project-dashboard/plan.md` — full implementation spec
- `docs/plans/DASHBOARD.md` — exact DASHBOARD.md format
- `docs/plans/README.md` — exact README.md format
- `docs/plans/templates/project-plan.md` — template for new project creation

---

## Step 1 — Scaffold the App

```bash
# From the app repo root
npx create-next-app@latest . \
  --typescript --tailwind --app --src-dir --no-eslint --no-import-alias
```

Then install `simple-git`:
```bash
npm install simple-git
```

---

## Step 2 — Configure

### `next.config.ts`
```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  serverExternalPackages: ['simple-git'],
}

export default nextConfig
```

### `.env.local` (create if not exists)
```
PLANS_REPO_PATH=./docs/plans
```

### `.gitignore` (add to existing)
```
.env.local
```

---

## Step 3 — Types (`src/types/project.ts`)

```ts
export type Status = 'Planning' | 'Active' | 'Paused' | 'Complete' | 'Abandoned'

export interface ProjectMeta {
  status: Status
  tags: string[]
  started: string
  lastUpdated: string
}

export interface Project {
  slug: string
  name: string
  description: string
  meta: ProjectMeta
}
```

---

## Step 4 — Config (`src/lib/config.ts`)

```ts
import path from 'path'

export const REPO_ROOT = path.resolve(
  process.env.PLANS_REPO_PATH ?? path.join(process.cwd(), 'docs/plans')
)
export const PROJECTS_DIR = path.join(REPO_ROOT, 'projects')
export const DASHBOARD_PATH = path.join(REPO_ROOT, 'DASHBOARD.md')
export const README_PATH = path.join(REPO_ROOT, 'README.md')
export const TEMPLATE_PATH = path.join(REPO_ROOT, 'templates', 'project-plan.md')
```

---

## Step 5 — Markdown Lib (`src/lib/markdown.ts`)

### Status map
```ts
import { Status } from '@/types/project'

export const STATUS_MAP: Record<Status, string> = {
  Planning:  '🔵 Planning',
  Active:    '🟢 Active',
  Paused:    '🟡 Paused',
  Complete:  '✅ Complete',
  Abandoned: '❌ Abandoned',
}

export const STATUS_KEYS = Object.keys(STATUS_MAP) as Status[]
```

### `parsePlanMeta(filePath)` — reads a plan.md, returns parsed metadata
Key regex patterns:
```ts
// Extract field value from "**Key:** Value" lines
const fieldRegex = (key: string) => new RegExp(`^\\*\\*${key}:\\*\\*\\s+(.+?)\\s*$`, 'm')

// Status: strip emoji, match last word to Status type
// Tags: split on spaces, filter items starting with #
// Dates: direct string capture

// Project name: first H1
const nameMatch = content.match(/^#\s+(.+)$/m)

// Goal description: paragraph after "## Goal" heading, before next "---"
const goalMatch = content.match(/## Goal\s+(?:_[^_]+_\s+)?([\s\S]+?)(?=\n---|\n##)/)
```

### `writeStatusToPlan(filePath, status)` — updates Status + Last Updated lines
```ts
// Replace the entire Status line value (handles both single and pipe-separated)
content = content.replace(
  /^(\*\*Status:\*\*\s+).+$/m,
  `$1${STATUS_MAP[status]}`
)
// Update Last Updated
const today = new Date().toISOString().slice(0, 10)
content = content.replace(
  /^(\*\*Last Updated:\*\*\s+).+$/m,
  `$1${today}`
)
fs.writeFileSync(filePath, content, 'utf8')
```

### `createPlanFromTemplate(slug, name, description, tags, status, today)` — creates new plan.md
```ts
// Read template, replace placeholders:
// "# Project Name"                         → "# {name}"
// "**Status:** 🔵 Planning | ..."          → "**Status:** {STATUS_MAP[status]}"
// "**Tags:** #tag1 #tag2"                  → "**Tags:** {tags}" (space-separated)
// "**Started:** YYYY-MM-DD"                → "**Started:** {today}"
// "**Last Updated:** YYYY-MM-DD"           → "**Last Updated:** {today}"
// placeholder text under ## Goal          → description

// Write to projects/{slug}/plan.md
```

---

## Step 6 — Dashboard Lib (`src/lib/dashboard.ts`)

### `updateDashboardRow(slug, status, today)` — update existing row
- Read DASHBOARD.md
- Find the row containing `[{slug}]` using regex
- Replace status emoji cell (column 1) and Last Updated cell (last column)
- Rebuild Stats block
- Write file

### `addDashboardRow(slug, name, description, status, today)` — append new row
- Read DASHBOARD.md
- Insert new row after the table header separator line under `## All Projects`
- Row format: `| {emoji} | [{slug}](projects/{slug}/plan.md) | {description} | {today} | {today} |`
- Rebuild Stats block
- Write file

### `rebuildStats(content)` — pure function, returns updated content
Count occurrences of each status emoji in table rows, replace the `## Stats` block:
```ts
// Stats block is always at the end after the last ---
// Replace from "## Stats" to end of file
const activeCount = (tableSection.match(/\| 🟢 \|/g) ?? []).length
// ... repeat for each status
```

---

## Step 7 — README Lib (`src/lib/readme.ts`)

### `addProjectToReadme(slug, name, description, status, today)` — append row to Projects table
- Read README.md
- Find `## Projects` table, insert new row after the separator line
- Row format: `| [{name}](projects/{slug}/plan.md) | {description} | {emoji} {label} | {today} |`
- Write file

---

## Step 8 — Git Lib (`src/lib/git.ts`)

```ts
import simpleGit from 'simple-git'
import { REPO_ROOT } from './config'
import { STATUS_MAP, Status } from './markdown'

const git = simpleGit(REPO_ROOT)

export async function commitStatusChange(slug: string, status: Status) {
  await git.add([`projects/${slug}/plan.md`, 'DASHBOARD.md'])
  const label = status  // text only, no emoji
  await git.commit(`chore: update ${slug} status → ${label}`)
}

export async function commitNewProject(slug: string, name: string) {
  await git.add([`projects/${slug}`, 'DASHBOARD.md', 'README.md'])
  await git.commit(`feat: add project ${name}`)
}
```

---

## Step 9 — API Routes

### `GET /api/projects` (`src/app/api/projects/route.ts`)
```ts
// 1. fs.readdirSync(PROJECTS_DIR, { withFileTypes: true })
//    filter entries that are directories
// 2. For each dir: check plan.md exists, call parsePlanMeta()
// 3. Return NextResponse.json(projects) sorted by lastUpdated desc
// 4. On error: return NextResponse.json({ error }, { status: 500 })
```

### `PATCH /api/projects/[slug]/status` (`src/app/api/projects/[slug]/status/route.ts`)
```ts
// Body: { status: string }
// 1. Validate status is in STATUS_KEYS → 400 if not
// 2. Check plan.md exists → 404 if not
// 3. writeStatusToPlan(planPath, status)
// 4. updateDashboardRow(slug, status, today)
// 5. await commitStatusChange(slug, status)
// 6. Return NextResponse.json({ ok: true, slug, status })
// On git error: return NextResponse.json({ error }, { status: 500 })
```

### `POST /api/projects/create` (`src/app/api/projects/create/route.ts`)
```ts
// Body: { name: string, description: string, tags: string, status: Status }
// 1. Derive slug: name.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '')
// 2. Check projects/{slug} doesn't exist → 409 if taken
// 3. fs.mkdirSync(projects/{slug}/AI_Instructions, { recursive: true })
// 4. fs.writeFileSync(projects/{slug}/AI_Instructions/.gitkeep, '')
// 5. createPlanFromTemplate(slug, name, description, tags, status, today)
// 6. addDashboardRow(slug, name, description, status, today)
// 7. addProjectToReadme(slug, name, description, status, today)
// 8. await commitNewProject(slug, name)
// 9. Return NextResponse.json({ ok: true, slug })
```

---

## Step 10 — UI Components

### `src/app/page.tsx`
```tsx
import ProjectGrid from '@/components/ProjectGrid'

export default function Home() {
  return (
    <main className="min-h-screen bg-gray-50 p-8">
      <div className="max-w-6xl mx-auto">
        <div className="flex items-center justify-between mb-8">
          <h1 className="text-2xl font-bold text-gray-900">Project Dashboard</h1>
          {/* AddProjectButton triggers modal in ProjectGrid */}
        </div>
        <ProjectGrid />
      </div>
    </main>
  )
}
```

### `src/components/ProjectGrid.tsx` — Client Component
```tsx
'use client'
// State: projects[], isLoading, error, showAddModal
// On mount: fetch GET /api/projects
// Renders: grid of ProjectCard + AddProjectModal
// onStatusChange(slug, status): 
//   optimistic update local state immediately
//   call PATCH /api/projects/{slug}/status
//   revert + show error on failure
// onProjectCreated(project):
//   append to projects state, close modal
```

### `src/components/ProjectCard.tsx`
```tsx
// Props: project: Project, onStatusChange: (slug, status) => void
// Renders:
//   - Project name (bold)
//   - Description (2-line truncate)
//   - Tags as small gray pills
//   - StatusBadge (opens StatusDropdown on click)
//   - Started / Last Updated dates (small text)
//   - Inline error banner (auto-dismiss 4s) if status change fails
```

### `src/components/StatusBadge.tsx`
```tsx
// Props: status: Status, onClick: () => void
// Color map:
//   Planning  → blue  (bg-blue-100 text-blue-700)
//   Active    → green (bg-green-100 text-green-700)
//   Paused    → yellow (bg-yellow-100 text-yellow-700)
//   Complete  → gray  (bg-gray-100 text-gray-600)
//   Abandoned → red   (bg-red-100 text-red-700)
// Renders: <button> pill with emoji + label
```

### `src/components/StatusDropdown.tsx`
```tsx
// Props: currentStatus, isOpen, onSelect, onClose
// Renders: positioned <ul> with 5 status options
// Current status gets a ✓ indicator
// useEffect: mousedown listener on document to close on outside click
// Positioned absolutely below/above the StatusBadge
```

### `src/components/AddProjectModal.tsx`
```tsx
// Props: isOpen, onClose, onCreated: (project: Project) => void
// Form fields:
//   - Project Name (text, required)
//   - Description (textarea, required)
//   - Tags (text, comma or space separated, e.g. "#web #api")
//   - Status (select, default "Planning")
// On submit: POST /api/projects/create
// On success: call onCreated with new project, close modal
// Backdrop click closes modal
// Show loading state on submit button while awaiting
```

---

## Build Sequence

Build in this order — each step is independently testable:

1. **Scaffold** — `create-next-app`, install `simple-git`, configure `next.config.ts`, create `.env.local`
2. **Types** — `src/types/project.ts`
3. **Config** — `src/lib/config.ts`, verify `REPO_ROOT` resolves to correct absolute path
4. **Markdown lib** — `src/lib/markdown.ts`, test `parsePlanMeta` against real plan.md manually
5. **GET /api/projects** — verify with `curl http://localhost:3000/api/projects` returns the project-dashboard project
6. **Read-only UI** — `ProjectGrid` + `ProjectCard` + `StatusBadge`, no interactions yet. Run `npm run dev` and confirm the card displays correctly.
7. **Dashboard + README + Git libs** — `lib/dashboard.ts`, `lib/readme.ts`, `lib/git.ts`
8. **PATCH /api/projects/[slug]/status** — test with curl:
   ```bash
   curl -X PATCH http://localhost:3000/api/projects/project-dashboard/status \
     -H "Content-Type: application/json" \
     -d '{"status":"Paused"}'
   # Verify DASHBOARD.md updated + git log shows commit
   ```
9. **StatusDropdown** — wire interactivity, test full status change flow in browser
10. **POST /api/projects/create + AddProjectModal** — test with curl first, then via modal in browser

---

## Verification Checklist

- [ ] `npm run dev` starts without errors
- [ ] `localhost:3000` shows project-dashboard card with 🟢 Active status
- [ ] Change status to Paused → card updates → `docs/plans/DASHBOARD.md` shows 🟡 Paused → `git log` in `docs/plans` shows commit
- [ ] Change back to Active → same flow confirms reverse works
- [ ] Click "+ Add Project" → fill form → new card appears → `docs/plans/projects/{slug}/` folder exists with plan.md → git log shows commit
- [ ] New project appears in `docs/plans/DASHBOARD.md` and `docs/plans/README.md`
- [ ] Refreshing the page shows all projects correctly (data from files, not memory)
