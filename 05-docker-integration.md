# 🐳 Docker Integration with GitLab CI/CD

> Build, test, push, and deploy Docker containers using GitLab pipelines.

---

## Why Docker + GitLab?

Docker is the backbone of modern CI/CD:

- **Consistent environments** — same container in dev, test, and prod
- **Isolated builds** — no dependency conflicts
- **Easy rollbacks** — just pull a previous image tag
- **Registry integration** — GitLab has a built-in container registry

---

## Docker in GitLab Pipelines

### Prerequisites

Your GitLab Runner must have Docker installed:

```bash
# Install Docker (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y docker.io

# Start & enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Grant gitlab-runner user Docker access
sudo usermod -aG docker gitlab-runner
sudo gitlab-runner restart
```

---

## Building Docker Images in CI/CD

### Basic Build & Push

```yaml
stages:
  - build
  - push

variables:
  APP_NAME: "my-app"

build_job:
  stage: build
  script:
    - docker build -t $APP_NAME:latest .
    - docker build -t $APP_NAME:$CI_COMMIT_SHORT_SHA .
  tags:
    - dev

push_job:
  stage: push
  script:
    - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS
    - docker tag $APP_NAME:latest $DOCKERHUB_USER/$APP_NAME:latest
    - docker tag $APP_NAME:$CI_COMMIT_SHORT_SHA $DOCKERHUB_USER/$APP_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKERHUB_USER/$APP_NAME:latest
    - docker push $DOCKERHUB_USER/$APP_NAME:$CI_COMMIT_SHORT_SHA
  tags:
    - dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## GitLab Container Registry

GitLab provides a **free, built-in** container registry for every project.

### Login to GitLab Registry

```yaml
push_to_gitlab_registry:
  stage: push
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

### Predefined Registry Variables

| Variable               | Value                                  |
| ---------------------- | -------------------------------------- |
| `CI_REGISTRY`          | `registry.gitlab.com`                  |
| `CI_REGISTRY_USER`     | `gitlab-ci-token`                      |
| `CI_REGISTRY_PASSWORD` | Auto-generated job token               |
| `CI_REGISTRY_IMAGE`    | `registry.gitlab.com/<user>/<project>` |

### Pull from GitLab Registry

```bash
# Login
docker login registry.gitlab.com

# Pull
docker pull registry.gitlab.com/<user>/<project>:latest
```

---

## Docker Compose in CI/CD

### Testing with Docker Compose

```yaml
integration_test:
  stage: test
  script:
    - docker compose -f docker-compose.test.yml up -d
    - sleep 10
    - curl -f http://localhost:5000/health || exit 1
    - docker compose -f docker-compose.test.yml down
  tags:
    - dev
```

### Docker Compose Deploy

```yaml
deploy_job:
  stage: deploy
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts 2>/dev/null
  script:
    - scp docker-compose.yml ubuntu@$DEPLOY_HOST:~/app/
    - ssh ubuntu@$DEPLOY_HOST "
      cd ~/app &&
      docker compose pull &&
      docker compose down &&
      docker compose up -d
      "
  tags:
    - dev
```

---

## Multi-Stage Docker Builds

Optimize image size with multi-stage builds:

```dockerfile
# ── Stage 1: Build ──
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ── Stage 2: Production ──
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Python Flask Example

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

---

## Docker Image Tagging Strategy

| Tag Format                   | Use Case                  |
| ---------------------------- | ------------------------- |
| `latest`                     | Most recent build         |
| `$CI_COMMIT_SHORT_SHA`       | Traceable to exact commit |
| `$CI_COMMIT_TAG` (e.g. v1.0) | Release versions          |
| `$CI_COMMIT_BRANCH`          | Branch-specific builds    |
| `$(date +%Y%m%d)`            | Date-based builds         |

```yaml
build_job:
  stage: build
  script:
    - docker build
      -t $DOCKER_IMAGE:latest
      -t $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
      -t $DOCKER_IMAGE:$(date +%Y%m%d)
      .
```

---

## Docker Cleanup in CI/CD

Runners can fill up disk fast. Add periodic cleanup:

```yaml
cleanup_job:
  stage: .post
  script:
    - docker system prune -af --volumes
    - docker image prune -af --filter "until=72h"
  when: always
  tags:
    - dev
```

Or on the runner machine via cron:

```bash
# Add to crontab
0 3 * * * docker system prune -af --volumes >> /var/log/docker-cleanup.log 2>&1
```

---

## Docker Security Scanning

### Using Trivy (Open Source)

```yaml
security_scan:
  stage: test
  script:
    - docker run --rm
      -v /var/run/docker.sock:/var/run/docker.sock
      aquasec/trivy image $APP_NAME:latest
  allow_failure: true
  tags:
    - dev
```

### GitLab Built-in Container Scanning

```yaml
include:
  - template: Security/Container-Scanning.gitlab-ci.yml

container_scanning:
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

---

## `.dockerignore` Best Practices

```dockerignore
.git
.gitignore
.gitlab-ci.yml
README.md
docker-compose*.yml
node_modules
__pycache__
*.pyc
.env
.vscode
.idea
tests/
docs/
```

---

## 📌 Quick Reference Commands

```bash
# Build
docker build -t myapp:latest .

# Run
docker run -d --name myapp -p 80:5000 myapp:latest

# Push to Docker Hub
docker login -u <user> -p <token>
docker tag myapp:latest <user>/myapp:latest
docker push <user>/myapp:latest

# Push to GitLab Registry
docker login registry.gitlab.com
docker tag myapp:latest registry.gitlab.com/<user>/<project>:latest
docker push registry.gitlab.com/<user>/<project>:latest

# Cleanup
docker system prune -af
docker image prune -af
docker volume prune -f

# Inspect
docker images
docker ps -a
docker logs <container>
docker exec -it <container> /bin/sh
```

---

> **Next:** [06 - Deployment to EC2 →](./06-deployment-ec2.md)
