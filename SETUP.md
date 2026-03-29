# SETUP.md — Push This Repo to GitHub

Step-by-step commands to get this repo live. Run these from the root of the project directory.

---

## Option A — GitHub CLI (fastest)

Requires `gh` CLI installed and authenticated (`gh auth login`).

```bash
# 1. Initialize git
git init
git add .
git commit -m "Initial commit: RAG onboarding tool scaffold"

# 2. Create the repo on GitHub and push in one command
gh repo create rag-onboarding-tool \
  --public \
  --description "RAG-powered role onboarding tool — private AI knowledge system for DevOps environments" \
  --source=. \
  --push
```

Done. The repo will be live at `https://github.com/dolgova/rag-onboarding-tool`.

---

## Option B — Manual (no CLI needed)

```bash
# 1. Initialize git
git init
git add .
git commit -m "Initial commit: RAG onboarding tool scaffold"
```

Then go to https://github.com/new and create a repo named `rag-onboarding-tool`.
Leave "Initialize this repository" unchecked — you already have a commit.

```bash
# 2. Add remote and push (SSH)
git remote add origin git@github.com:dolgova/rag-onboarding-tool.git
git branch -M main
git push -u origin main
```

Or if you use HTTPS:
```bash
git remote add origin https://github.com/dolgova/rag-onboarding-tool.git
git branch -M main
git push -u origin main
```

---

## Verify

After pushing, confirm the repo looks right:

```bash
gh repo view dolgova/rag-onboarding-tool --web
# or just open https://github.com/dolgova/rag-onboarding-tool
```

You should see:
- `README.md` rendered on the front page
- `ARCHITECTURE.md`, `SESSION_GUIDE.md`, `STATUS.md` at root
- `docs/security_compliance_considerations.docx` downloadable
- Four subdirectories: `ingestion/`, `query/`, `api/`, `infra/`

---

## First commit message convention

For subsequent sessions, use descriptive commit messages that match the session:

```bash
# After completing a session deliverable
git add .
git commit -m "Phase 1 Session 1: ingestion/chunker.py — 512/64 token chunking"
git push
```

This keeps the git log as a readable record of the build sequence.
