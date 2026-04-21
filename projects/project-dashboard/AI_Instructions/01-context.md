---
name: Project Context
description: Overview of the project-dashboard app — architecture, GitHub API approach, file formats, constraints
type: reference
---

# Project Dashboard — Agent Context

## What This App Does

A Next.js web dashboard (deployed on Vercel) that manages projects stored in the `project_plans` GitHub repo. It:
1. **Displays** all projects as cards (status, tags, description, dates)
2. **Updates status** via a dropdown — writes directly to GitHub via API, auto-commits
3. **Creates new projects** via a form modal — creates folder + plan.md + updates DASHBOARD.md + README.md, auto-commits
4. **Protected** via GitHub OAuth — only the repo owner can log in and make changes

Deployed on Vercel. No database. GitHub is the single source of truth and audit trail.

---

## Architecture: GitHub API (Not Submodule / simple-git)

**Do NOT use `simple-git` or `fs.writeFileSync`.** Vercel's serverless functions have a read-only ephemeral filesystem. The app must use the **GitHub REST API (Octokit)** to read and write all files.

This also means **no submodule is needed** in the app repo. The app talks directly to the `project_plans` repo via API.

### How it works
```
User action (status change / new project)
  → Next.js API route
    → GitHub API: GET file content + sha
    → modify markdown string in memory
    → GitHub API: PUT file (content + sha + commit message)
      → file updated + commit created in project_plans repo automatically
  → return success to UI
```

Every GitHub API write creates a commit in `project_plans` — no separate git push needed, works from Vercel or localhost.

### Required environment variables
```
GITHUB_TOKEN=ghp_...            ← Personal Access Token with repo scope
GITHUB_OWNER=MayurT-finoux
GITHUB_REPO=project_plans
```

For Vercel: add these in the Vercel project dashboard → Settings → Environment Variables.
For local dev: add to `.env.local` in the app root.

The PAT must have `repo` scope (read + write). When the repo goes private, nothing changes in the app — the token handles auth.

---

## project_plans Repo Structure

```
project_plans/                       ← GitHub repo: MayurT-finoux/project_plans
├── projects/
│   └── {slug}/
│       ├── plan.md                  ← project metadata + planning doc
│       └── AI_Instructions/         ← agent instruction files
├── templates/
│   └── project-plan.md              ← template for new projects
├── DASHBOARD.md                     ← status table for all projects
└── README.md                        ← project index table
```

---

## Markdown File Formats

### `plan.md` — Bold-field metadata at top

```markdown
# Project Name

**Status:** 🟢 Active
**Tags:** #web #dashboard #nextjs
**Started:** 2026-04-18
**Last Updated:** 2026-04-18

---

## Goal

Description of what the project does and "done" looks like.
```

**Status values:**
```
Planning  → "🔵 Planning"
Active    → "🟢 Active"
Paused    → "🟡 Paused"
Complete  → "✅ Complete"
Abandoned → "❌ Abandoned"
```

The template has a pipe-separated Status line — always replace the full value with the single chosen status on first write:
```
**Status:** 🔵 Planning | 🟢 Active | 🟡 Paused | ✅ Complete | ❌ Abandoned
```

### `DASHBOARD.md` — All Projects table

```markdown
## All Projects

| Status | Project | Description | Started | Last Updated |
|--------|---------|-------------|---------|--------------|
| 🟢 | [project-dashboard](projects/project-dashboard/plan.md) | Web dashboard | 2026-04-18 | 2026-04-18 |

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
| [project-dashboard](projects/project-dashboard/plan.md) | Web dashboard | 🟢 Active | 2026-04-18 |
```

---

## GitHub API File Operations

### Reading a file
```ts
const { data } = await octokit.repos.getContent({
  owner: GITHUB_OWNER, repo: GITHUB_REPO, path
})
// data.content is base64 encoded, data.sha is needed for updates
const content = Buffer.from(data.content, 'base64').toString('utf8')
```

### Writing / updating a file
```ts
await octokit.repos.createOrUpdateFileContents({
  owner: GITHUB_OWNER, repo: GITHUB_REPO,
  path,
  message: 'chore: update status',
  content: Buffer.from(newContent).toString('base64'),
  sha: existingSha,   // required for updates, omit for new files
})
```

### Listing a directory
```ts
const { data } = await octokit.repos.getContent({
  owner: GITHUB_OWNER, repo: GITHUB_REPO, path: 'projects'
})
// data is an array of { name, type, path, sha } when path is a directory
```

### Creating a new file (no sha needed)
```ts
await octokit.repos.createOrUpdateFileContents({
  owner: GITHUB_OWNER, repo: GITHUB_REPO,
  path: `projects/${slug}/plan.md`,
  message: `feat: add project ${name}`,
  content: Buffer.from(planContent).toString('base64'),
  // no sha field = create new file
})
```

---

## Key Constraints

- All GitHub API calls are server-side only (API routes + lib) — never in client components
- The `sha` must be fetched fresh before every update to prevent overwrite conflicts
- Batch updates (e.g. new project creates 3 files) should use sequential API calls — GitHub doesn't support batch commits via REST API
- Use `@octokit/rest` package — not `@octokit/core` directly
- Authentication: use NextAuth.js with GitHub provider for deployed app, token-based for API routes
- No `simple-git`, no `fs.writeFileSync`, no `PLANS_REPO_PATH`

---

## Planned Improvements (Phase 2)

These are not MVP but are planned. Build Phase 1 fully first.

### Automation (highest priority Phase 2)
1. **Auto-create GitHub repo** — `POST /api/projects/[slug]/create-repo`
   - `octokit.repos.createForAuthenticatedUser({ name: slug, private: true, auto_init: true })`
   - Store created repo URL back into plan.md as a `**Repo:**` field
   - Show "Create Repo" button on card when no repo exists yet

2. **Auto-create Codespace** — `POST /api/projects/[slug]/create-codespace`
   - `octokit.codespaces.createForRepo({ owner, repo: slug, ref: 'main', ... })`
   - Returns Codespace URL — show "Open in Codespace" button on card
   - Requires `codespace` scope added to PAT

### Progress & Detail
3. **Milestone progress bars** — parse `- [x]` and `- [ ]` in plan.md Tasks section, show `X / Y tasks` + progress bar on each card
4. **Project detail page** — `/projects/[slug]` route, renders full plan.md as HTML, inline task checkbox toggle (calls PATCH to update plan.md)

### Discovery & Sync
5. **Filter/search** — filter cards by status or tag (client-side, no new API needed)
6. **Webhook sync** — GitHub webhook → `POST /api/webhook` → re-fetch projects in UI (keeps dashboard in sync when plan.md is edited directly in GitHub)
7. **GitHub Actions trigger** — on status → Complete, trigger `workflow_dispatch` event via API
