# Git Branching Strategy
## Member 1 Deliverable — Repository Management

---

## Branch Structure

```
main ──────────────────────────────────────────── Production-ready code
  │
  └── develop ─────────────────────────────────── Integration branch
        │
        ├── feature/m1-docker-compose ──────────── Member 1 work
        ├── feature/m2-rag-pipeline ────────────── Member 2 work
        ├── feature/m3-guardrails ──────────────── Member 3 work
        ├── feature/m4-analytics ───────────────── Member 4 work
        └── feature/m5-frontend ────────────────── Member 5 work
```

---

## Rules

### `main` Branch
- **Protected**: no direct pushes, only merge via Pull Request
- Requires at least 1 review approval
- Must pass CI checks (GitHub Actions)
- Always deployable — if it's on `main`, it works

### `develop` Branch
- Integration branch where all features merge first
- May have temporary issues as features integrate
- Weekly sync with `main` when stable

### `feature/*` Branches
- One per member per task: `feature/m2-rag-pipeline`, `feature/m3-guardrails`
- Branch from `develop`, merge back into `develop`
- Delete after merge

---

## Daily Workflow

```bash
# 1. Start your day — sync with develop
git checkout develop
git pull origin develop

# 2. Create or switch to your feature branch
git checkout -b feature/m2-rag-pipeline  # first time
git checkout feature/m2-rag-pipeline      # subsequent times

# 3. Work, commit frequently
git add .
git commit -m "feat(m2): add relevance threshold check to query pipeline"

# 4. Push your branch
git push origin feature/m2-rag-pipeline

# 5. When ready — create a Pull Request on GitHub
#    develop ← feature/m2-rag-pipeline
```

---

## Commit Message Format

```
type(scope): short description

Examples:
  feat(m1): add health checks to docker-compose
  feat(m2): implement Qdrant vector search in query pipeline
  fix(m3): regex false positive on 'system' in academic queries
  docs(m4): add privacy compliance documentation
  style(m5): improve mobile responsiveness of chat UI
  test(m3): add 10 new adversarial test cases
  chore(m1): update .gitignore for IDE files
```

Types: `feat`, `fix`, `docs`, `style`, `test`, `chore`, `refactor`

---

## Conflict Resolution

When multiple members edit overlapping files:

1. The person merging pulls the latest `develop`
2. If conflicts arise, resolve locally and test before pushing
3. `docker-compose.yml` conflicts: coordinate with Member 1
4. n8n workflow JSON conflicts: re-export the workflow from n8n (don't manually merge JSON)

---

## Release Process

1. All features merged into `develop`
2. Full test suite passes (`test_guardrails.sh` + `test_e2e.sh`)
3. Create PR: `main ← develop`
4. At least 2 team members review and approve
5. Merge and tag: `git tag v1.0.0`
