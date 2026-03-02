# 🚀 GitLab DevOps Playbook

> A complete, reusable reference guide for GitLab CI/CD — from basics to production deployments.
> Use this as your **go-to playbook** whenever you set up GitLab pipelines, runners, Docker builds, or EC2 deployments.

---

## 📚 Playbook Chapters

| #   | Chapter                                                   | What You'll Learn                                                  |
| --- | --------------------------------------------------------- | ------------------------------------------------------------------ |
| 📘  | [01 - GitLab Basics](./01-gitlab-basics.md)               | Project setup, Git commands, authentication, GitLab vs GitHub      |
| 🏃  | [02 - Runners Explained](./02-runners-explained.md)       | Runner types, executors, installation, registration, config        |
| 🔐  | [03 - Variables & Secrets](./03-variables-and-secrets.md) | CI/CD variables, masking, protection, predefined vars, scoping     |
| 🚀  | [04 - CI/CD Pipeline](./04-ci-cd-pipeline.md)             | Stages, jobs, rules, artifacts, caching, environments, DAG         |
| 🐳  | [05 - Docker Integration](./05-docker-integration.md)     | Docker builds, registries, Compose, multi-stage, security scanning |
| 🖥️  | [06 - Deployment to EC2](./06-deployment-ec2.md)          | SSH deploy, Docker Compose deploy, blue-green, rollback strategies |
| 🐛  | [07 - Common Errors](./07-common-errors.md)               | Troubleshooting runner, Docker, SSH, YAML, and variable issues     |

### 📄 Quick Setup Guide

| Resource                  | Description                                             |
| ------------------------- | ------------------------------------------------------- |
| [Setup Guide](./guide.md) | Step-by-step walkthrough: Runner install → Pipeline run |

---

## ⚡ Essential Commands Cheat Sheet

### Git & GitLab

```bash
# Push to GitLab
git remote add gitlab https://gitlab.com/<user>/<repo>.git
git push -u gitlab main

# Push to GitHub + GitLab simultaneously
git remote set-url --add origin https://gitlab.com/<user>/<repo>.git
git push origin main

# Create Merge Request via push
git push -o merge_request.create -o merge_request.target=main origin feature-branch
```

### GitLab Runner

```bash
# Install runner (Ubuntu/Debian)
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y

# Register runner (new glrt- token)
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token <glrt-TOKEN> \
  --executor shell \
  --non-interactive

# Grant Docker access
sudo usermod -aG docker gitlab-runner
sudo gitlab-runner restart

# Management
sudo gitlab-runner list          # List runners
sudo gitlab-runner status        # Check status
sudo gitlab-runner verify        # Verify connectivity
sudo gitlab-runner restart       # Restart service
sudo gitlab-runner unregister --name "<name>"   # Remove runner
```

### Docker

```bash
# Build & tag
docker build -t myapp:latest .
docker tag myapp:latest user/myapp:latest

# Push to Docker Hub
docker login -u <user> -p <token>
docker push user/myapp:latest

# Push to GitLab Registry
docker login registry.gitlab.com
docker tag myapp:latest registry.gitlab.com/<user>/<project>:latest
docker push registry.gitlab.com/<user>/<project>:latest

# Run & manage
docker run -d --name myapp -p 80:5000 --restart unless-stopped myapp:latest
docker ps -a
docker logs -f myapp
docker stop myapp && docker rm myapp

# Cleanup
docker system prune -af --volumes
```

### SSH Deployment

```bash
# Setup SSH key in pipeline
mkdir -p ~/.ssh
echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa
ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts

# SSH deploy
ssh -o StrictHostKeyChecking=no ubuntu@$DEPLOY_HOST "docker pull user/myapp:latest && docker stop myapp || true && docker rm myapp || true && docker run -d --name myapp -p 80:5000 user/myapp:latest"
```

---

## 🔧 Minimal `.gitlab-ci.yml` Template

```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  APP_NAME: "my-app"

build:
  stage: build
  script:
    - docker build -t $APP_NAME:latest .
  tags: [dev]

test:
  stage: test
  script:
    - docker run --rm -d --name test-$APP_NAME -p 5001:5000 $APP_NAME:latest
    - sleep 5
    - curl -f http://localhost:5001/health || exit 1
    - docker stop test-$APP_NAME
  tags: [dev]

push:
  stage: push
  script:
    - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS
    - docker tag $APP_NAME:latest $DOCKERHUB_USER/$APP_NAME:latest
    - docker push $DOCKERHUB_USER/$APP_NAME:latest
  tags: [dev]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy:
  stage: deploy
  script:
    - docker pull $DOCKERHUB_USER/$APP_NAME:latest
    - docker stop $APP_NAME || true
    - docker rm $APP_NAME || true
    - docker run -d --name $APP_NAME -p 80:5000 --restart unless-stopped $DOCKERHUB_USER/$APP_NAME:latest
  tags: [dev]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## 📋 CI/CD Variables Checklist

Set these in GitLab → **Settings** → **CI/CD** → **Variables**:

| Variable                | Example                 | Masked | Protected | Required For  |
| ----------------------- | ----------------------- | ------ | --------- | ------------- |
| `DOCKERHUB_USER`        | `yourusername`          | ❌     | ❌        | Docker push   |
| `DOCKERHUB_PASS`        | `dckr_pat_xxx`          | ✅     | ❌        | Docker push   |
| `SSH_PRIVATE_KEY`       | Contents of `.pem` file | ✅     | ✅        | SSH deploy    |
| `DEPLOY_HOST`           | `54.123.45.67`          | ❌     | ✅        | SSH deploy    |
| `APP_NAME`              | `my-app`                | ❌     | ❌        | All stages    |
| `AWS_ACCESS_KEY_ID`     | `AKIA...`               | ✅     | ✅        | AWS/Terraform |
| `AWS_SECRET_ACCESS_KEY` | `wJal...`               | ✅     | ✅        | AWS/Terraform |

---

## 🔑 Important `.gitlab-ci.yml` Keywords

| Keyword           | Purpose                          | Example                             |
| ----------------- | -------------------------------- | ----------------------------------- |
| `stages`          | Define pipeline phases           | `stages: [build, test, deploy]`     |
| `script`          | Commands to execute              | `script: - echo "hello"`            |
| `tags`            | Route to specific runner         | `tags: [dev]`                       |
| `rules`           | Conditional execution (modern)   | `if: '$CI_COMMIT_BRANCH == "main"'` |
| `only` / `except` | Branch filtering (legacy)        | `only: [main]`                      |
| `when`            | Manual, delayed, always          | `when: manual`                      |
| `needs`           | DAG — skip stage wait            | `needs: [build_job]`                |
| `artifacts`       | Pass files between jobs          | `paths: [build/]`                   |
| `cache`           | Speed up builds                  | `paths: [node_modules/]`            |
| `environment`     | Track deployments                | `name: production`                  |
| `variables`       | Inline variables                 | `APP: "my-app"`                     |
| `before_script`   | Pre-job commands                 | SSH key setup                       |
| `allow_failure`   | Don't fail pipeline if job fails | `allow_failure: true`               |
| `retry`           | Auto-retry failed jobs           | `retry: 2`                          |
| `timeout`         | Job time limit                   | `timeout: 30 minutes`               |
| `parallel`        | Matrix / parallel jobs           | `matrix: [{NODE: ["16","18"]}]`     |
| `trigger`         | Downstream / child pipelines     | `project: group/other-project`      |
| `resource_group`  | Prevent concurrent deployments   | `resource_group: production`        |
| `interruptible`   | Cancel if newer pipeline starts  | `interruptible: true`               |

---

## 🔗 Useful Links

| Resource                   | URL                                                               |
| -------------------------- | ----------------------------------------------------------------- |
| GitLab CI/CD Docs          | https://docs.gitlab.com/ee/ci/                                    |
| `.gitlab-ci.yml` Reference | https://docs.gitlab.com/ee/ci/yaml/                               |
| Predefined Variables List  | https://docs.gitlab.com/ee/ci/variables/predefined_variables.html |
| GitLab Runner Docs         | https://docs.gitlab.com/runner/                                   |
| Container Registry Docs    | https://docs.gitlab.com/ee/user/packages/container_registry/      |
| Pipeline Editor            | `https://gitlab.com/<user>/<repo>/-/ci/editor`                    |
| GitLab API Docs            | https://docs.gitlab.com/ee/api/                                   |

---

## 📁 Repository Structure

```
gitlab-devops-playbook/
│
├── README.md                       ← You are here
├── guide.md                        ← Quick Setup Guide (step-by-step)
│
├── 01-gitlab-basics.md             ← Project setup, auth, Git commands
├── 02-runners-explained.md         ← Runner types, install, register
├── 03-variables-and-secrets.md     ← CI/CD variables, masking, scoping
├── 04-ci-cd-pipeline.md            ← Stages, jobs, rules, templates
├── 05-docker-integration.md        ← Docker builds, registries, scanning
├── 06-deployment-ec2.md            ← EC2 deploy strategies, rollback
└── 07-common-errors.md             ← Troubleshooting & fixes
```

---

## 🧭 How to Use This Playbook

1. **New to GitLab?** Start with [01 - GitLab Basics](./01-gitlab-basics.md)
2. **Setting up CI/CD?** Follow the [Setup Guide](./guide.md) → then read [04 - CI/CD Pipeline](./04-ci-cd-pipeline.md)
3. **Deploying to EC2?** Jump to [06 - Deployment to EC2](./06-deployment-ec2.md)
4. **Something broke?** Check [07 - Common Errors](./07-common-errors.md)
5. **Need a quick command?** Use the cheat sheet above ☝️

---
