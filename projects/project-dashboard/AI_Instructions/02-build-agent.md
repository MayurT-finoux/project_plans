---
name: Build Agent Instructions
description: Complete step-by-step instructions for building the project-dashboard Next.js app with GitHub API
type: reference
---

# Build Agent — Full Implementation Instructions

**Before starting:** Read `01-context.md` in this folder. It covers the GitHub API approach, markdown formats, and why `simple-git`/`fs` must NOT be used.

---

## Step 1 — Scaffold

```bash
npx create-next-app@latest . \
  --typescript --tailwind --app --src-dir --no-eslint --no-import-alias

npm install @octokit/rest next-auth
```

---

## Step 2 — Environment Variables

### `.env.local` (never commit this)
```
GITHUB_TOKEN=ghp_...
GITHUB_OWNER=MayurT-finoux
GITHUB_REPO=project_plans

# For NextAuth GitHub OAuth
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
NEXTAUTH_SECRET=...
NEXTAUTH_URL=http://localhost:3000
```

### How to create a GitHub Personal Access Token (PAT)

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **"Generate new token (classic)"**
3. Set a name (e.g. `project-dashboard`)
4. Set expiration as needed
5. Check the **`repo`** scope (top-level checkbox — grants full read/write to all repos)
6. Click **"Generate token"** and copy it immediately (shown only once)
7. Paste it as `GITHUB_TOKEN` in `.env.local`

**Scope needed:** `repo` — required for reading files, writing files, and creating commits in `project_plans`.

### `next.config.ts`
```ts
import type { NextConfig } from 'next'
const nextConfig: NextConfig = {}
export default nextConfig
```

---

## Step 3 — Types (`src/types/project.ts`)

```ts
export type Status = 'Planning' | 'Active' | 'Paused' | 'Complete' | 'Abandoned'

export interface ProjectMeta {
  status: Status
  tags: string[]
  started: string      // YYYY-MM-DD
  lastUpdated: string  // YYYY-MM-DD
}

export interface Project {
  slug: string
  name: string
  description: string
  meta: ProjectMeta
}

export interface GitHubFile {
  content: string
  sha: string
}
```

---

## Step 4 — GitHub Client (`src/lib/github.ts`)

Single Octokit wrapper. All GitHub API calls go through here.

```ts
import { Octokit } from '@octokit/rest'

const octokit = new Octokit({ auth: process.env.GITHUB_TOKEN })
const OWNER = process.env.GITHUB_OWNER!
const REPO = process.env.GITHUB_REPO!

// Read a file — returns decoded content + sha
export async function getFile(path: string): Promise<GitHubFile> {
  const { data } = await octokit.repos.getContent({ owner: OWNER, repo: REPO, path })
  if (Array.isArray(data)) throw new Error(`${path} is a directory`)
  const content = Buffer.from((data as any).content, 'base64').toString('utf8')
  return { content, sha: (data as any).sha }
}

// Write/update a file — sha required for updates, omit for new files
export async function writeFile(
  path: string, content: string, message: string, sha?: string
): Promise<void> {
  await octokit.repos.createOrUpdateFileContents({
    owner: OWNER, repo: REPO, path, message,
    content: Buffer.from(content).toString('base64'),
    ...(sha ? { sha } : {}),
  })
}

// List directory — returns array of { name, type, path }
export async function listDir(path: string) {
  const { data } = await octokit.repos.getContent({ owner: OWNER, repo: REPO, path })
  if (!Array.isArray(data)) throw new Error(`${path} is not a directory`)
  return data as Array<{ name: string; type: string; path: string }>
}
```

---

## Step 5 — Markdown Lib (`src/lib/markdown.ts`)

**Pure functions only** — takes strings, returns strings. No file I/O.

### Status map
```ts
export const STATUS_MAP: Record<Status, string> = {
  Planning: '🔵 Planning', Active: '🟢 Active',
  Paused: '🟡 Paused', Complete: '✅ Complete', Abandoned: '❌ Abandoned',
}
export const STATUS_KEYS = Object.keys(STATUS_MAP) as Status[]

// Reverse lookup: "🟢 Active" → "Active"
export function parseStatusString(raw: string): Status {
  return (STATUS_KEYS.find(k => STATUS_MAP[k] === raw.trim()) ?? 'Planning')
}
```

### `parsePlanMeta(content: string): ProjectMeta & { name: string; description: string }`
```ts
// Name: first H1
const name = content.match(/^#\s+(.+)$/m)?.[1]?.trim() ?? 'Unnamed'

// Status field (handles both single value and pipe-separated template line)
const statusRaw = content.match(/^\*\*Status:\*\*\s+(.+?)(?:\s*\|.*)?$/m)?.[1]?.trim() ?? ''
const status = parseStatusString(statusRaw)

// Tags: space-separated tokens starting with #
const tagsRaw = content.match(/^\*\*Tags:\*\*\s+(.+)$/m)?.[1] ?? ''
const tags = tagsRaw.split(/\s+/).filter(t => t.startsWith('#'))

// Dates
const started = content.match(/^\*\*Started:\*\*\s+(.+)$/m)?.[1]?.trim() ?? ''
const lastUpdated = content.match(/^\*\*Last Updated:\*\*\s+(.+)$/m)?.[1]?.trim() ?? ''

// Description: first non-empty, non-italic line under ## Goal
const goalSection = content.match(/## Goal\s+([\s\S]+?)(?=\n---|\n##)/)?.[1] ?? ''
const description = goalSection.split('\n')
  .map(l => l.trim()).filter(l => l && !l.startsWith('_'))[0] ?? ''
```

### `writeStatus(content: string, status: Status, today: string): string`
```ts
// Replaces the entire Status line value (normalizes pipe-separated template line too)
content = content.replace(/^(\*\*Status:\*\*\s+).+$/m, `$1${STATUS_MAP[status]}`)
content = content.replace(/^(\*\*Last Updated:\*\*\s+).+$/m, `$1${today}`)
return content
```

### `buildPlanFromTemplate(template: string, data: { name, description, tags, status, today }): string`
```ts
let plan = template
plan = plan.replace(/^#\s+Project Name$/m, `# ${data.name}`)
plan = plan.replace(/^\*\*Status:\*\*\s+.+$/m, `**Status:** ${STATUS_MAP[data.status]}`)
plan = plan.replace(/^\*\*Tags:\*\*\s+.+$/m, `**Tags:** ${data.tags}`)
plan = plan.replace(/^\*\*Started:\*\*\s+.+$/m, `**Started:** ${data.today}`)
plan = plan.replace(/^\*\*Last Updated:\*\*\s+.+$/m, `**Last Updated:** ${data.today}`)
// Replace goal placeholder with description
plan = plan.replace(
  /## Goal\s+_[^_]+_/,
  `## Goal\n\n${data.description}`
)
return plan
```

---

## Step 6 — Dashboard Lib (`src/lib/dashboard.ts`)

**Pure functions** — takes DASHBOARD.md string, returns updated string.

### `updateDashboardRow(content, slug, status, today): string`
- Find the row where column 2 contains `[{slug}]`
- Replace column 1 (status emoji) and last column (last updated)
- Call `rebuildStats(content)` after

### `addDashboardRow(content, slug, name, description, status, today): string`
- Insert new row after the `|---|` separator line in `## All Projects` table
- Format: `| {emoji} | [{slug}](projects/{slug}/plan.md) | {description} | {today} | {today} |`
- Call `rebuildStats(content)` after

### `rebuildStats(content): string`
```ts
// Count emoji occurrences in table rows
const active = (content.match(/\| 🟢 \|/g) ?? []).length
const paused = (content.match(/\| 🟡 \|/g) ?? []).length
const complete = (content.match(/\| ✅ \|/g) ?? []).length
const abandoned = (content.match(/\| ❌ \|/g) ?? []).length
const planning = (content.match(/\| 🔵 \|/g) ?? []).length

// Replace everything from "## Stats" to end of file
const newStats = `## Stats\n\n- **Active:** ${active}\n- **Paused:** ${paused}\n- **Planning:** ${planning}\n- **Complete:** ${complete}\n- **Abandoned:** ${abandoned}\n`
return content.replace(/## Stats[\s\S]*$/, newStats)
```

---

## Step 7 — README Lib (`src/lib/readme.ts`)

**Pure function** — takes README.md string, returns updated string.

### `addProjectToReadme(content, slug, name, description, status, today): string`
- Find `## Projects` table separator row
- Insert new row after it
- Format: `| [{name}](projects/{slug}/plan.md) | {description} | {emoji} {status} | {today} |`

---

## Step 8 — Auth (`src/app/api/auth/[...nextauth]/route.ts`)

```ts
import NextAuth from 'next-auth'
import GitHub from 'next-auth/providers/github'

const handler = NextAuth({
  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    })
  ],
  callbacks: {
    // Only allow the repo owner to sign in
    async signIn({ profile }) {
      return profile?.login === process.env.GITHUB_OWNER
    }
  }
})

export { handler as GET, handler as POST }
```

Add middleware to protect all routes except `/api/auth/*`:

### `src/middleware.ts`
```ts
export { default } from 'next-auth/middleware'
export const config = { matcher: ['/((?!api/auth|_next).*)'] }
```

---

## Step 9 — API Routes

### `GET /api/projects` (`src/app/api/projects/route.ts`)
```ts
// 1. listDir('projects') → filter type === 'directory'
// 2. For each dir:
//    a. getFile(`projects/${slug}/plan.md`) → { content, sha }
//    b. parsePlanMeta(content) → name, description, meta
// 3. Return NextResponse.json(projects) sorted by lastUpdated desc
// Cache: set revalidate = 0 (always fresh — files change via API)
```

### `PATCH /api/projects/[slug]/status` (`src/app/api/projects/[slug]/status/route.ts`)
```ts
// Body: { status: string }
// 1. Validate status in STATUS_KEYS → 400
// 2. const today = new Date().toISOString().slice(0, 10)
// 3. const { content, sha } = await getFile(`projects/${slug}/plan.md`)
// 4. const updated = writeStatus(content, status, today)
// 5. await writeFile(
//      `projects/${slug}/plan.md`, updated,
//      `chore: update ${slug} status → ${status}`, sha
//    )
// 6. // Update DASHBOARD.md
//    const dash = await getFile('DASHBOARD.md')
//    const updatedDash = updateDashboardRow(dash.content, slug, status, today)
//    await writeFile('DASHBOARD.md', updatedDash,
//      `chore: update ${slug} status → ${status}`, dash.sha)
// 7. Return NextResponse.json({ ok: true })
```

### `POST /api/projects/create` (`src/app/api/projects/create/route.ts`)
```ts
// Body: { name, description, tags, status }
// 1. slug = name.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '')
// 2. Try getFile(`projects/${slug}/plan.md`) → if it exists, return 409
// 3. const today = new Date().toISOString().slice(0, 10)
// 4. const { content: template } = await getFile('templates/project-plan.md')
// 5. const plan = buildPlanFromTemplate(template, { name, description, tags, status, today })
// 6. await writeFile(`projects/${slug}/plan.md`, plan,
//      `feat: add project ${name}`)          ← no sha = create new file
// 7. await writeFile(`projects/${slug}/AI_Instructions/.gitkeep`, '',
//      `feat: add project ${name} (scaffolding)`)
// 8. const dash = await getFile('DASHBOARD.md')
//    const updatedDash = addDashboardRow(dash.content, slug, name, description, status, today)
//    await writeFile('DASHBOARD.md', updatedDash,
//      `feat: add project ${name}`, dash.sha)
// 9. const readme = await getFile('README.md')
//    const updatedReadme = addProjectToReadme(readme.content, slug, name, description, status, today)
//    await writeFile('README.md', updatedReadme,
//      `feat: add project ${name}`, readme.sha)
// 10. Return NextResponse.json({ ok: true, slug })
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
          <h1 className="text-2xl font-bold">Project Dashboard</h1>
          {/* AddProjectButton rendered inside ProjectGrid */}
        </div>
        <ProjectGrid />
      </div>
    </main>
  )
}
```

### `src/components/ProjectGrid.tsx` (Client Component)
```tsx
'use client'
// State: projects[], isLoading, error, showAddModal
// On mount: fetch GET /api/projects
// Renders: "+ Add Project" button + grid of ProjectCards + AddProjectModal
// onStatusChange(slug, status):
//   optimistic: update local state immediately
//   call PATCH /api/projects/{slug}/status
//   on error: revert state + show toast
// onProjectCreated(project):
//   prepend to projects state + close modal
```

### `src/components/ProjectCard.tsx`
```
Props: project: Project, onStatusChange: (slug, status) => void
Renders:
  - Project name (h2, bold)
  - Description (2-line clamp, text-gray-600)
  - Tags as small pills (bg-gray-100, rounded-full)
  - StatusBadge + StatusDropdown
  - Started / Last Updated (text-xs, text-gray-400)
  - Inline error banner (red, auto-dismiss 4s) on PATCH failure
```

### `src/components/StatusBadge.tsx`
```
Props: status: Status, onClick: () => void
Renders: <button> pill with emoji + label
Color classes by status:
  Planning  → bg-blue-100 text-blue-700
  Active    → bg-green-100 text-green-700
  Paused    → bg-yellow-100 text-yellow-700
  Complete  → bg-gray-100 text-gray-500
  Abandoned → bg-red-100 text-red-600
```

### `src/components/StatusDropdown.tsx`
```
Props: currentStatus, isOpen, onSelect, onClose
Renders: absolute-positioned <ul> listing all 5 statuses
Current status shows ✓ indicator
useEffect: document mousedown listener → call onClose on outside click
```

### `src/components/AddProjectModal.tsx`
```
Props: isOpen, onClose, onCreated: (project: Project) => void
Form fields:
  - Project Name     (text input, required)
  - Description      (textarea, required)
  - Tags             (text input, e.g. "#web #api", space or comma separated)
  - Status           (select, default "Planning")
On submit: POST /api/projects/create → call onCreated → close
Loading state on submit button while awaiting
Backdrop div → onClick closes modal
```

---

## Build Sequence

1. **Scaffold** — create-next-app + install packages + `.env.local`
2. **Types** — `src/types/project.ts`
3. **GitHub client** — `src/lib/github.ts`, test `getFile('DASHBOARD.md')` in a temp script
4. **Markdown lib** — `src/lib/markdown.ts` pure functions, unit-test against real plan.md content
5. **GET /api/projects** — curl test: `curl localhost:3000/api/projects`
6. **Read-only UI** — ProjectGrid + ProjectCard + StatusBadge. Verify project-dashboard card appears.
7. **Dashboard + README libs** — `lib/dashboard.ts` + `lib/readme.ts`
8. **PATCH /api/projects/[slug]/status** — curl test:
   ```bash
   curl -X PATCH localhost:3000/api/projects/project-dashboard/status \
     -H "Content-Type: application/json" -d '{"status":"Paused"}'
   # Check GitHub repo — DASHBOARD.md should be updated with new commit
   ```
9. **Auth** — NextAuth + middleware. Verify redirect to GitHub login when logged out.
10. **StatusDropdown** — wire to PATCH, test full flow in browser
11. **POST /api/projects/create** — curl test, then AddProjectModal in browser
12. **Deploy to Vercel** — add env vars in Vercel dashboard, set `NEXTAUTH_URL` to production URL

---

## Verification Checklist

- [ ] `npm run dev` starts without errors
- [ ] Unauthenticated visit → redirected to GitHub OAuth
- [ ] After login → dashboard shows project-dashboard card with 🟢 Active
- [ ] Change status → card updates → **GitHub repo** DASHBOARD.md updated → new commit visible in repo
- [ ] Add project via modal → new card appears → `projects/{slug}/` folder in GitHub repo → DASHBOARD.md + README.md updated
- [ ] Deployed on Vercel → same flows work from production URL
- [ ] GitHub repo goes private → app still works (PAT handles auth)
