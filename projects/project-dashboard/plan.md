# Project Dashboard

**Status:** 🟢 Active  
**Tags:** #web #dashboard #tooling  
**Started:** 2026-04-18  
**Last Updated:** 2026-04-18  

---

## Goal

Build a web dashboard that reads project folders from this planning repo, displays project statuses visually, and lets you update a project's status — which writes back to the markdown files and commits to the git repo. Makes managing projects faster without manually editing markdown every time.

---

## Tasks

### To Do
- [ ] Decide tech stack (frontend-only static vs. lightweight backend)
- [ ] Design the UI layout — project cards with status badges
- [ ] Parse project plan markdown files to extract status/metadata
- [ ] Build status-update flow (click to change → writes to file → git commit)
- [ ] Sync DASHBOARD.md automatically when status changes
- [ ] Set up in the coding repo (or as a standalone repo)
- [ ] Wire up submodule so dashboard reads from project_plans

### In Progress
- [ ] 

### Done
- [x] Created project plan

---

## Timeline

| Milestone | Target Date | Done? |
|-----------|-------------|-------|
| Tech stack decided + repo scaffolded | 2026-04-25 | [ ] |
| Read-only dashboard (display only) | 2026-05-02 | [ ] |
| Status update + git commit working | 2026-05-10 | [ ] |
| MVP shipped | 2026-05-15 | [ ] |

---

## Tech Stack / Tools

_To be decided. Options:_

| Option | Pros | Cons |
|--------|------|------|
| **Node.js + Express + plain HTML** | Simple, git integration easy via `simple-git` npm package | Need to run a server |
| **Python + Flask** | Easy file I/O, `gitpython` library, clean | Extra runtime dependency |
| **Next.js (full-stack)** | Modern, API routes handle git ops, good UI ecosystem | More setup overhead for a small tool |

---

## Key Features (MVP)

1. **Project list view** — cards showing project name, status emoji, last updated date
2. **Status change** — click a status badge → dropdown → select new status → saves to `plan.md` + updates `DASHBOARD.md`
3. **Git auto-commit** — on status change, auto-commits the updated files with a message like `chore: update planning-dashboard status → Active`
4. **Submodule aware** — works whether run from the planning repo or from the coding repo via submodule

---

## Notes

- Dashboard will run locally (localhost) — not a deployed service
- All changes go through git, so history is always preserved
- Status field format in markdown: `**Status:** 🟢 Active` — parse this line to read/write status
- DASHBOARD.md stats block needs to be kept in sync too

---

## Resources & Links

- [DASHBOARD.md](../../DASHBOARD.md) — the file this app will manage
- [templates/project-plan.md](../../templates/project-plan.md) — format the parser needs to handle
- `simple-git` npm: https://github.com/steveukx/git-js
- `gitpython`: https://gitpython.readthedocs.io

---

## Decisions Log

| Date | Decision | Reason |
|------|----------|--------|
| 2026-04-18 | Store status in markdown frontmatter-style bold fields | Keeps files human-readable without requiring YAML frontmatter parser |
| 2026-04-18 | Auto-commit on status change | Keeps git history as the audit trail — no separate database needed |

---

## Risks

- Parsing markdown status line could break if format drifts across files — need a strict convention
- Git conflicts if dashboard and manual edits happen simultaneously — low risk for solo use
- Auto-commit could be annoying if it creates too many tiny commits — may batch changes

---

## Blockers / Open Questions

- Which tech stack? (decide before coding)
- Should status changes require a confirmation step before committing?
- Run as a CLI tool or always-on local server?
