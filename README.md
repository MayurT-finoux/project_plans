# project_plans

A personal planning repo for tracking projects and ideas.  
→ Full status view: [DASHBOARD.md](DASHBOARD.md)  
→ Raw ideas: [IDEAS.md](IDEAS.md)

---

## Projects

| Project | Description | Status | Started |
|---------|-------------|--------|---------|
| [project-dashboard](projects/project-dashboard/plan.md) | Web dashboard to track and manage projects from this repo | 🟢 Active | 2026-04-18 |

---

## Folder Structure

```
project_plans/
├── projects/    ← all projects live here permanently (path never changes)
│   └── your-project/
│       ├── plan.md
│       └── AI_Instructions/
└── templates/   ← copy project-plan.md to start a new project
```

## How to Use

1. **New project:** copy `templates/project-plan.md` → `projects/your-project-name/plan.md`
2. **Status changes:** update the `Status` field in `plan.md` and in [DASHBOARD.md](DASHBOARD.md)
3. **Project folders never move** — status is tracked in the files, not the folder location

## Submodule Usage (from coding repo)

```bash
# Add this repo as a submodule in your coding repo
git submodule add https://github.com/MayurT-finoux/project_plans docs/plans

# Commit plan changes from inside the coding repo
cd docs/plans
git add .
git commit -m "update: ..."
git push
cd ../..
git add docs/plans
git commit -m "chore: update plans submodule"
git push

# Pull latest plan changes into coding repo
git submodule update --remote docs/plans
```
