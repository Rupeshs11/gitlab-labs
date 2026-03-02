# 📘 GitLab Basics

> Everything you need to get started with GitLab — from project setup to daily workflow.

---

## What is GitLab?

GitLab is a complete DevOps platform delivered as a single application. It covers:

- **Source Code Management** (Git repositories)
- **CI/CD Pipelines** (build, test, deploy automation)
- **Container Registry** (store Docker images)
- **Issue Tracking & Boards** (project management)
- **Security Scanning** (SAST, DAST, dependency scanning)
- **Infrastructure as Code** (Terraform integration)

---

## Creating a GitLab Project

### Option 1: Blank Project

1. Go to [gitlab.com](https://gitlab.com) → **New Project** → **Create blank project**
2. Set visibility: `Private`, `Internal`, or `Public`
3. Optionally initialize with a `README.md`

### Option 2: Push Existing Local Repo

```bash
# Add GitLab as a remote
git remote add gitlab https://gitlab.com/<username>/<repo>.git

# Push your code
git push -u gitlab main
```

### Option 3: Mirror from GitHub

GitLab → **New Project** → **Import project** → **GitHub** → Authorize → Select repo

---

## Push to GitHub + GitLab Simultaneously

```bash
# Add a second push URL to your existing 'origin' remote
git remote set-url --add origin https://gitlab.com/<username>/<repo>.git

# Verify both URLs are listed
git remote -v

# Now 'git push origin main' pushes to BOTH
git push origin main
```

---

## GitLab Project Structure

| Item               | Description                                      |
| ------------------ | ------------------------------------------------ |
| **Repository**     | Your source code, branches, tags                 |
| **Issues**         | Bug reports, feature requests                    |
| **Merge Requests** | Code review and merge workflow (like GitHub PRs) |
| **CI/CD**          | Pipeline configuration and execution             |
| **Packages**       | Container registry, package registry             |
| **Wiki**           | Built-in documentation pages                     |
| **Snippets**       | Shareable code snippets                          |

---

## Key Git Commands for GitLab

```bash
# Clone a GitLab repo
git clone https://gitlab.com/<username>/<repo>.git

# Create a new branch
git checkout -b feature/my-feature

# Stage, commit, and push
git add .
git commit -m "feat: add new feature"
git push origin feature/my-feature

# Create a Merge Request (MR) via CLI push option
git push -o merge_request.create \
         -o merge_request.target=main \
         -o merge_request.title="My Feature MR" \
         origin feature/my-feature
```

---

## GitLab vs GitHub — Quick Comparison

| Feature            | GitLab                      | GitHub                           |
| ------------------ | --------------------------- | -------------------------------- |
| CI/CD              | Built-in (`.gitlab-ci.yml`) | GitHub Actions (workflows)       |
| Container Registry | Built-in                    | GitHub Packages (GHCR)           |
| Self-Hosting       | GitLab CE/EE (free)         | GitHub Enterprise (paid)         |
| Merge Requests     | GitLab MRs                  | Pull Requests                    |
| Security Scanning  | Built-in (SAST, DAST, etc.) | Third-party or Advanced Security |
| Issue Boards       | Built-in Kanban             | Projects (beta boards)           |

---

## Access Tokens & Authentication

### Personal Access Token (PAT)

GitLab → **User Settings** → **Access Tokens** → Create with scopes:

| Scope              | Use Case                            |
| ------------------ | ----------------------------------- |
| `read_api`         | Read-only API access                |
| `write_repository` | Push code via HTTPS                 |
| `read_registry`    | Pull images from container registry |
| `api`              | Full API access                     |

```bash
# Clone using PAT
git clone https://oauth2:<YOUR_PAT>@gitlab.com/<username>/<repo>.git
```

### SSH Key Authentication

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy public key
cat ~/.ssh/id_ed25519.pub

# Add to GitLab → User Settings → SSH Keys
```

---

## `.gitignore` for Common Projects

```gitignore
# Python
__pycache__/
*.pyc
.env
venv/

# Node.js
node_modules/
dist/
.env

# Docker
.dockerignore

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db
```

---

## Important GitLab URLs

| Resource           | URL                                                   |
| ------------------ | ----------------------------------------------------- |
| GitLab Dashboard   | `https://gitlab.com/dashboard`                        |
| CI/CD Pipelines    | `https://gitlab.com/<user>/<repo>/-/pipelines`        |
| Container Registry | `https://gitlab.com/<user>/<repo>/container_registry` |
| Runner Settings    | `https://gitlab.com/<user>/<repo>/-/settings/ci_cd`   |
| Variables          | Same as Runner Settings → Expand **Variables**        |
| API Docs           | `https://docs.gitlab.com/ee/api/`                     |

---

## 📌 Quick Reference Commands

```bash
# Check remote URLs
git remote -v

# Switch default branch
git branch -M main

# Force push (use carefully)
git push --force origin main

# View commit history
git log --oneline -10

# Stash changes
git stash
git stash pop
```

---

> **Next:** [02 - Runners Explained →](./02-runners-explained.md)
