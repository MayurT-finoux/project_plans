---
name: Project Context
description: Overview of the project-dashboard app — what it does, what repo it reads, file formats, and constraints
type: reference
---

# Project Dashboard — Agent Context

## What This App Does

A local Next.js web dashboard that manages projects stored in the `project_plans` git repo. It:
1. **Displays** all projects as cards (status, tags, description, dates)
2. **Updates status** via a dropdown — writes back to markdown files and auto-commits to git
3. **Creates new projects** via a form modal — generates folder + plan.md from template, updates DASHBOARD.md and README.md, auto-commits to git

Runs locally at `localhost:3000`. Not deployed. No database — git is the audit trail.

---

## How project_plans Is Accessed

The `project_plans` repo is added as a git submodule in this app repo at:
```
docs/plans/
```

So the full path to project files from the app root is:
```
docs/plans/projects/{slug}/plan.md
docs/plans/DASHBOARD.md
docs/plans/README.md
docs/plans/templates/project-plan.md
```

The `.env.local` in the app root must contain:
```
PLANS_REPO_PATH=./docs/plans
```

The `lib/config.ts` module resolves this to an absolute path at runtime.

---

## project_plans Folder Structure

```
docs/plans/                         ← REPO_ROOT (resolved from PLANS_REPO_PATH)
├── projects/
│   └── {slug}/
│       ├── plan.md                 ← project metadata + planning doc
│       └── AI_Instructions/        ← agent instruction files (like this one)
├── templates/
│   └── project-plan.md             ← template used when creating new projects
├── DASHBOARD.md                    ← status table for all projects
└── README.md                       ← project index table
```

---

## Markdown File Formats

### `plan.md` — Bold-field metadata at the top

```markdown
# Project Name

**Status:** 🟢 Active
**Tags:** #web #dashboard #nextjs
**Started:** 2026-04-18
**Last Updated:** 2026-04-18

---

## Goal

Description of what the project does and what "done" looks like.

---
... (rest of planning doc)
```

**Important:** The template file has a pipe-separated Status line:
```
**Status:** 🔵 Planning | 🟢 Active | 🟡 Paused | ✅ Complete | ❌ Abandoned
```
When writing status for the first time, this entire pipe-separated value must be replaced with the single chosen status string.

**Status values and their display strings:**
```
Planning  → "🔵 Planning"
Active    → "🟢 Active"
Paused    → "🟡 Paused"
Complete  → "✅ Complete"
Abandoned → "❌ Abandoned"
```

### `DASHBOARD.md` — All Projects table

```markdown
## All Projects

| Status | Project | Description | Started | Last Updated |
|--------|---------|-------------|---------|--------------|
| 🟢 | [project-dashboard](projects/project-dashboard/plan.md) | Web dashboard to track projects | 2026-04-18 | 2026-04-18 |

---

## Stats

- **Active:** 1
- **Paused:** 0
- **Complete:** 0
- **Abandoned:** 0
```

### `README.md` — Projects table

```markdown
## Projects

| Project | Description | Status | Started |
|---------|-------------|--------|---------|
| [project-dashboard](projects/project-dashboard/plan.md) | Web dashboard to track projects | 🟢 Active | 2026-04-18 |
```

---

## Git Rules

- The `simple-git` instance must be initialized with `REPO_ROOT` (the plans repo path), NOT `process.cwd()` (the app directory)
- `commitStatusChange(slug, status)`: stage only `projects/{slug}/plan.md` and `DASHBOARD.md`
- `commitNewProject(slug, name)`: stage `projects/{slug}/`, `DASHBOARD.md`, `README.md`
- **Never** use `git add -A` or `git add .`
- **No push** for MVP — all commits are local only
- Commit message format:
  - Status change: `chore: update {slug} status → {label}`
  - New project: `feat: add project {name}`

---

## Key Constraints

- All file I/O uses Node.js `fs` module — server-side only (API routes + lib)
- `simple-git` must be in `serverExternalPackages` in `next.config.ts` to avoid client bundle errors
- No YAML frontmatter — metadata uses bold-field markdown format (`**Key:** Value`)
- No third-party markdown parser needed — regex is sufficient for the simple bold-field format
- No date library — use `new Date().toISOString().slice(0, 10)` for YYYY-MM-DD
- TypeScript throughout
- Tailwind CSS for styling
