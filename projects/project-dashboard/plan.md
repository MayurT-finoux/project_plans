# Project Dashboard

**Status:** 🔵 Planning
**Tags:** #web #dashboard #nextjs #vercel #tooling  
**Started:** 2026-04-18  
**Last Updated:** 2026-04-21

---

## Goal

Build a local Next.js web dashboard that reads project folders from the `project_plans` repo, displays all projects as cards with status badges, and lets you change a project's status via a dropdown — which writes back to `plan.md`, updates `DASHBOARD.md`, and auto-commits to git. Replaces manual markdown editing for day-to-day status management.

---

## Folder Structure

```
project-dashboard/
├── .env.local                          ← PLANS_REPO_PATH=/workspaces/project_plans
├── next.config.ts                      ← serverExternalPackages: ['simple-git']
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── src/
    ├── app/
    │   ├── layout.tsx                  ← root layout, global styles
    │   ├── page.tsx                    ← main page, renders ProjectGrid
    │   ├── globals.css                 ← Tailwind base + status badge colors
    │   └── api/
    │       ├── projects/
    │       │   └── route.ts            ← GET: returns all parsed projects
    │       └── projects/[slug]/
    │           └── status/
    │               └── route.ts        ← PATCH: update status + git commit
    ├── lib/
    │   ├── config.ts                   ← resolves PLANS_REPO_PATH to absolute path
    │   ├── markdown.ts                 ← parse/write bold-field metadata from plan.md
    │   ├── dashboard.ts                ← read/write DASHBOARD.md table + stats block
    │   └── git.ts                      ← simple-git wrapper: stage 2 files + commit
    ├── types/
    │   └── project.ts                  ← Project, Status, ProjectMeta interfaces
    └── components/
        ├── ProjectGrid.tsx             ← grid of cards, holds state, calls API
        ├── ProjectCard.tsx             ← card: name, description, dates, status
        ├── StatusBadge.tsx             ← colored pill badge, opens dropdown on click
        └── StatusDropdown.tsx          ← 5-option menu, calls PATCH on select
```

---

## Tasks

### To Do
- [ ] Scaffold with `create-next-app` + install `simple-git`
- [ ] Write `src/lib/config.ts` — resolve `PLANS_REPO_PATH`
- [ ] Write `src/lib/markdown.ts` — `parsePlanMeta()` + `writeStatusToPlan()`
- [ ] Write `GET /api/projects` — scan `projects/` dir, return `Project[]`
- [ ] Build read-only UI — `ProjectGrid` + `ProjectCard` + `StatusBadge`
- [ ] Write `src/lib/dashboard.ts` — `updateDashboardRow()` + `rebuildStats()`
- [ ] Write `src/lib/git.ts` — `commitStatusChange(slug, status)`
- [ ] Write `PATCH /api/projects/[slug]/status` — wire all three steps
- [ ] Build `StatusDropdown` — connect to PATCH route
- [ ] Polish — loading skeletons, inline error states

### In Progress
- [ ] 

### Done
- [x] Created project plan
- [x] Decided tech stack: Next.js + TypeScript + Tailwind + simple-git

---

## Timeline

| Milestone | Target Date | Done? |
|-----------|-------------|-------|
| Scaffold + lib/config.ts working | 2026-04-25 | [ ] |
| GET /api/projects + read-only UI | 2026-05-02 | [ ] |
| PATCH /status + git commit working | 2026-05-10 | [ ] |
| MVP shipped (full interactive dashboard) | 2026-05-15 | [ ] |

---

## Tech Stack

| Package | Purpose |
|---------|---------|
| Next.js 14+ (App Router) | Full-stack framework — API routes + React UI |
| TypeScript | Type safety across lib, routes, components |
| Tailwind CSS | Styling — status badge colors, card layout |
| `@octokit/rest` | GitHub API — read/write files, auto-commits (replaces simple-git) |
| `next-auth` | GitHub OAuth — only repo owner can access the deployed app |

---

## API Routes

### `GET /api/projects`
- Reads `projects/` directory, finds all `plan.md` files
- Calls `parsePlanMeta()` on each, extracts H1 name + Goal description
- Returns `Project[]` sorted by `lastUpdated` descending

### `PATCH /api/projects/[slug]/status`
- Body: `{ "status": "Paused" }`
- Step 1: validate status is one of 5 allowed values
- Step 2: `writeStatusToPlan(planPath, status)` — updates plan.md
- Step 3: `updateDashboardRow(slug, status, today)` — updates DASHBOARD.md + Stats
- Step 4: `commitStatusChange(slug, status)` — stages 2 files, commits
- Returns `{ ok: true, slug, status }` on success

---

## Key Implementation Details

### Architecture: GitHub API (not simple-git)
Vercel serverless functions have a read-only filesystem. All reads/writes use **`@octokit/rest`** — every write auto-commits to the repo. No local git, no submodule, works on Vercel and localhost.

```
GITHUB_TOKEN=ghp_...        ← PAT with repo scope
GITHUB_OWNER=MayurT-finoux
GITHUB_REPO=project_plans
NEXTAUTH_SECRET=...          ← for GitHub OAuth
```

### `src/lib/github.ts` — GitHub API wrapper
```ts
getFile(path)                         → { content: string, sha: string }
writeFile(path, content, message, sha?) → creates commit in project_plans
listDir(path)                         → [{ name, type, path }]
```

### `src/lib/markdown.ts` — pure functions (no file I/O)
```ts
// Read status (handles pipe-separated template line too)
/^\*\*Status:\*\*\s+(.+?)(?:\s*\|.*)?$/m

// Write status
content.replace(/^(\*\*Status:\*\*\s+).+$/m, `$1${STATUS_MAP[status]}`)

// Write last updated
content.replace(/^(\*\*Last Updated:\*\*\s+).+$/m, `$1${today}`)
```

### Status map
```ts
const STATUS_MAP = {
  Planning:  '🔵 Planning',
  Active:    '🟢 Active',
  Paused:    '🟡 Paused',
  Complete:  '✅ Complete',
  Abandoned: '❌ Abandoned',
}
```

### `src/types/project.ts`
```ts
export type Status = 'Planning' | 'Active' | 'Paused' | 'Complete' | 'Abandoned'

export interface Project {
  slug: string        // directory name under projects/
  name: string        // H1 from plan.md
  description: string // first paragraph under ## Goal
  meta: {
    status: Status
    tags: string[]
    started: string       // YYYY-MM-DD
    lastUpdated: string   // YYYY-MM-DD
  }
}
```

---

## Component Behaviour

| Component | Responsibility |
|-----------|---------------|
| `ProjectGrid` | Fetches projects, holds state, passes `onStatusChange` to cards |
| `ProjectCard` | Renders name, description, dates, status badge — no internal state |
| `StatusBadge` | Colored pill button — green/blue/yellow/gray/red by status |
| `StatusDropdown` | Positioned menu over badge — calls `onSelect(newStatus)` on pick, closes on outside click |

**Optimistic update:** `ProjectGrid` updates local state immediately on status change, reverts on API error. Error shown as inline banner on the card, auto-dismisses after 4s.

---

## Notes

- Dashboard runs locally at `localhost:3000` — not a deployed service
- All changes go through git — history is the audit trail, no database needed
- Status field always written as a single value (e.g. `🟢 Active`), never the pipe-separated template format
- `PLANS_REPO_PATH` in `.env.local` points to wherever `project_plans` lives — works both standalone and via submodule at `docs/plans/`
- `simple-git` must be in `serverExternalPackages` in `next.config.ts` to avoid client bundle errors

---

## Resources & Links

- [DASHBOARD.md](../../DASHBOARD.md) — real format the dashboard.ts parser must handle
- [templates/project-plan.md](../../templates/project-plan.md) — pipe-separated Status line that markdown.ts must normalize
- [simple-git docs](https://github.com/steveukx/git-js)
- [Next.js App Router docs](https://nextjs.org/docs/app)

---

## Decisions Log

| Date | Decision | Reason |
|------|----------|--------|
| 2026-04-18 | Next.js + TypeScript + Tailwind | Full-stack in one project, great UI ecosystem, easy to extend |
| 2026-04-18 | Status stored as bold markdown fields, not YAML | Human-readable without a frontmatter parser |
| 2026-04-18 | Auto-commit on status change, no push | Git is the audit trail; push stays manual for control |
| 2026-04-18 | Stage exactly 2 files per commit | Prevents accidental commits of unrelated changes |
| 2026-04-18 | No third-party dropdown or date library | Format is simple enough for regex + native Date |

---

## Risks

- Markdown status regex breaks if bold-field format drifts across files — enforce convention strictly
- Git commit fails if `user.email`/`user.name` not configured in the plans repo — must check on setup
- Optimistic UI update could show stale state if git commit silently fails — handle error path carefully

---

## Blockers / Open Questions

- None — ready to build

---

## AI Agents

| Agent | Role | Instruction File | Status |
|-------|------|-----------------|--------|
| | | | |
