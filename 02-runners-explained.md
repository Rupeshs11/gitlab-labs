# 🏃 GitLab Runners Explained

> Understand what runners are, the types available, and how to set up self-hosted runners on EC2.

---

## What is a GitLab Runner?

A **GitLab Runner** is an agent that picks up and executes CI/CD jobs defined in `.gitlab-ci.yml`. Without a runner, pipelines won't execute.

```
┌─────────────────┐       ┌──────────────────┐       ┌───────────────┐
│  .gitlab-ci.yml │ ───►  │   GitLab Server  │ ───►  │  GitLab Runner│
│  (Pipeline Def) │       │  (Coordinator)   │       │  (Executes Jobs)
└─────────────────┘       └──────────────────┘       └───────────────┘
```

---

## Types of Runners

| Type               | Scope                         | Use Case                           |
| ------------------ | ----------------------------- | ---------------------------------- |
| **Shared Runner**  | Available to all projects     | GitLab.com provides these for free |
| **Group Runner**   | Available to a group/subgroup | Team-wide CI/CD                    |
| **Project Runner** | Specific to one project       | Custom environments, security      |

### Shared vs Self-Hosted

| Feature       | Shared (GitLab.com)       | Self-Hosted                        |
| ------------- | ------------------------- | ---------------------------------- |
| Setup         | Zero — already available  | Manual install + register          |
| Cost          | 400 min/month (free tier) | Your own infra cost                |
| Customization | Limited                   | Full control (tools, Docker, etc.) |
| Speed         | Variable (shared queue)   | Dedicated performance              |
| Security      | Multi-tenant              | Isolated to your infra             |

---

## Executor Types

The executor determines **how** jobs run on the runner machine:

| Executor         | Description                               | Best For                      |
| ---------------- | ----------------------------------------- | ----------------------------- |
| `shell`          | Runs commands directly on the host        | Simple setups, direct access  |
| `docker`         | Runs each job in a fresh Docker container | Isolated, reproducible builds |
| `docker+machine` | Auto-scales Docker hosts                  | Large-scale CI/CD             |
| `kubernetes`     | Runs jobs as Kubernetes pods              | Cloud-native environments     |
| `ssh`            | Runs commands on a remote machine via SSH | Legacy systems                |
| `virtualbox`     | Runs jobs in VirtualBox VMs               | Heavy isolation               |

> **Recommendation:** Use `shell` for simplicity when the runner already has Docker installed. Use `docker` executor for fully isolated builds.

---

## Install GitLab Runner on EC2 (Ubuntu/Debian)

### Step 1: Install the Package

```bash
# Add GitLab Runner repo
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash

# Install
sudo apt install gitlab-runner -y

# Verify
gitlab-runner --version
```

### Step 2: Get Registration Token

1. Go to your GitLab project
2. **Settings** → **CI/CD** → **Runners** → Expand
3. Click **New project runner**
4. Set a description and tags (e.g., `dev`, `build`, `deploy`)
5. Click **Create runner**
6. Copy the `glrt-...` token

### Step 3: Register the Runner

```bash
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token <PASTE_YOUR_glrt_TOKEN> \
  --description "ec2-runner" \
  --executor shell \
  --non-interactive
```

> ⚠️ **Important:** With new `glrt-` tokens, tags are configured in the **GitLab UI only**. Do NOT pass `--tag-list` in the register command — it will error.

### Step 4: Grant Docker Access

```bash
# Add gitlab-runner user to docker group
sudo usermod -aG docker gitlab-runner

# Restart the runner
sudo gitlab-runner restart
```

### Step 5: Verify

```bash
# List registered runners
sudo gitlab-runner list

# Check runner status
sudo gitlab-runner status

# Run a single job manually (for debugging)
sudo gitlab-runner run
```

---

## Runner Configuration File

The main config file lives at `/etc/gitlab-runner/config.toml`:

```toml
concurrent = 1          # Max jobs to run simultaneously
check_interval = 0      # How often to check for new jobs (seconds, 0 = default 3s)

[[runners]]
  name = "ec2-runner"
  url = "https://gitlab.com"
  token = "glrt-xxxxxxxxxxxx"
  executor = "shell"

  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```

### Key Configuration Options

| Setting          | Description                       | Example           |
| ---------------- | --------------------------------- | ----------------- |
| `concurrent`     | Max parallel jobs                 | `4`               |
| `check_interval` | Polling interval for new jobs     | `5`               |
| `executor`       | How jobs execute                  | `shell`, `docker` |
| `limit`          | Max jobs for this specific runner | `2`               |
| `output_limit`   | Max job log size in KB            | `4096`            |

---

## Runner Tags & Job Matching

Tags ensure jobs run on the **correct runner**:

```yaml
# .gitlab-ci.yml
build_job:
  stage: build
  script:
    - echo "Building..."
  tags:
    - dev # ← Only runs on runners tagged 'dev'

deploy_job:
  stage: deploy
  script:
    - echo "Deploying..."
  tags:
    - production # ← Only runs on runners tagged 'production'
```

**Set tags in GitLab UI:**  
GitLab → Settings → CI/CD → Runners → Edit runner → Tags

---

## Managing Multiple Runners

```bash
# List all registered runners
sudo gitlab-runner list

# Verify runner connectivity
sudo gitlab-runner verify

# Unregister a specific runner
sudo gitlab-runner unregister --name "old-runner"

# Unregister all runners
sudo gitlab-runner unregister --all-runners

# Restart after config changes
sudo gitlab-runner restart
```

---

## Runner as a Systemd Service

```bash
# Check service status
sudo systemctl status gitlab-runner

# Start / Stop / Restart
sudo systemctl start gitlab-runner
sudo systemctl stop gitlab-runner
sudo systemctl restart gitlab-runner

# Enable on boot
sudo systemctl enable gitlab-runner
```

---

## 🔒 Security Best Practices

1. **Use project runners** for sensitive repos (not shared)
2. **Protect runner tokens** — never commit them to code
3. **Use tags** to control which runners execute which jobs
4. **Limit concurrent jobs** to prevent resource exhaustion
5. **Run runners in Docker**executor for isolation
6. **Restrict network access** on runner machines
7. **Rotate tokens** periodically

---

## 📌 Quick Reference Commands

```bash
# Install
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y

# Register
sudo gitlab-runner register --url https://gitlab.com --token <TOKEN> --executor shell --non-interactive

# Docker access
sudo usermod -aG docker gitlab-runner && sudo gitlab-runner restart

# Status & management
sudo gitlab-runner list
sudo gitlab-runner status
sudo gitlab-runner verify
sudo gitlab-runner restart
sudo gitlab-runner unregister --name "<runner-name>"
```

---

> **Next:** [03 - Variables & Secrets →](./03-variables-and-secrets.md)
