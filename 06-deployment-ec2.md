# 🖥️ Deployment to EC2 with GitLab CI/CD

> Automate deploying your Dockerized application to AWS EC2 using GitLab pipelines.

---

## Architecture Overview

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Developer   │────►│   GitLab     │────►│  Docker Hub  │────►│   AWS EC2    │
│  git push    │     │  Pipeline    │     │  (Registry)  │     │  (Deploy)    │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

**Flow:**

1. Developer pushes code to GitLab
2. Pipeline builds & pushes Docker image
3. Pipeline SSHs into EC2 and runs the new container

---

## Prerequisites

### AWS EC2 Instance Setup

```bash
# Launch Ubuntu 22.04 EC2 instance with:
# - Instance type: t2.micro (free tier) or t2.small
# - Security Group: Allow ports 22 (SSH), 80 (HTTP), 443 (HTTPS)
# - Key pair: Download .pem file

# SSH into EC2
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

# Install Docker
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### GitLab CI/CD Variables Required

| Variable          | Value                        | Masked | Protected |
| ----------------- | ---------------------------- | ------ | --------- |
| `DOCKERHUB_USER`  | Your Docker Hub username     | ❌     | ❌        |
| `DOCKERHUB_PASS`  | Docker Hub access token      | ✅     | ❌        |
| `SSH_PRIVATE_KEY` | Full contents of `.pem` file | ✅     | ✅        |
| `DEPLOY_HOST`     | EC2 public IP or domain      | ❌     | ✅        |
| `APP_NAME`        | Application name             | ❌     | ❌        |

---

## Deployment Strategy 1: Direct Docker (Runner on EC2)

When the GitLab Runner **is installed on the same EC2** that runs the app:

```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  APP_NAME: "my-app"

build_job:
  stage: build
  script:
    - docker build -t $APP_NAME:latest .
  tags:
    - dev

test_job:
  stage: test
  script:
    - docker run --rm -d --name test-app -p 5001:5000 $APP_NAME:latest
    - sleep 5
    - curl -f http://localhost:5001/health || exit 1
    - docker stop test-app
  tags:
    - dev

push_job:
  stage: push
  script:
    - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS
    - docker tag $APP_NAME:latest $DOCKERHUB_USER/$APP_NAME:latest
    - docker push $DOCKERHUB_USER/$APP_NAME:latest
  tags:
    - dev

deploy_job:
  stage: deploy
  script:
    - docker pull $DOCKERHUB_USER/$APP_NAME:latest
    - docker stop $APP_NAME || true
    - docker rm $APP_NAME || true
    - docker run -d --name $APP_NAME
      -p 80:5000
      --restart unless-stopped
      $DOCKERHUB_USER/$APP_NAME:latest
  tags:
    - dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## Deployment Strategy 2: SSH Deploy (Runner ≠ EC2)

When the runner is on a **different machine** than the deployment target:

```yaml
deploy_job:
  stage: deploy
  before_script:
    - apt-get update && apt-get install -y openssh-client
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts 2>/dev/null
  script:
    - ssh -o StrictHostKeyChecking=no ubuntu@$DEPLOY_HOST "
      docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS &&
      docker pull $DOCKERHUB_USER/$APP_NAME:latest &&
      docker stop $APP_NAME || true &&
      docker rm $APP_NAME || true &&
      docker run -d --name $APP_NAME
      -p 80:5000
      --restart unless-stopped
      $DOCKERHUB_USER/$APP_NAME:latest
      "
  environment:
    name: production
    url: http://$DEPLOY_HOST
  tags:
    - dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## Deployment Strategy 3: Docker Compose via SSH

For multi-container deployments:

```yaml
deploy_job:
  stage: deploy
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts 2>/dev/null
  script:
    # Copy compose file to EC2
    - scp docker-compose.prod.yml ubuntu@$DEPLOY_HOST:~/app/docker-compose.yml

    # Deploy on EC2
    - ssh ubuntu@$DEPLOY_HOST "
      cd ~/app &&
      export DOCKERHUB_USER=$DOCKERHUB_USER &&
      export APP_NAME=$APP_NAME &&
      docker compose pull &&
      docker compose down &&
      docker compose up -d
      "
  tags:
    - dev
```

### `docker-compose.prod.yml` Example

```yaml
version: "3.8"

services:
  app:
    image: ${DOCKERHUB_USER}/${APP_NAME}:latest
    ports:
      - "80:5000"
    restart: unless-stopped
    environment:
      - FLASK_ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
    restart: unless-stopped
```

---

## Blue-Green Deployment

Zero-downtime deployment using two containers:

```yaml
deploy_blue_green:
  stage: deploy
  script:
    - ssh ubuntu@$DEPLOY_HOST "
        # Pull new image
        docker pull $DOCKERHUB_USER/$APP_NAME:latest

        # Start new container (green) on different port
        docker run -d --name ${APP_NAME}-green
          -p 5001:5000
          --restart unless-stopped
          $DOCKERHUB_USER/$APP_NAME:latest

        # Health check on green container
        sleep 5
        curl -f http://localhost:5001/health || {
          docker stop ${APP_NAME}-green && docker rm ${APP_NAME}-green
          echo 'Green deployment failed health check!'
          exit 1
        }

        # Switch traffic: stop blue, remap green to port 80
        docker stop ${APP_NAME}-blue || true
        docker rm ${APP_NAME}-blue || true
        docker stop ${APP_NAME}-green
        docker rm ${APP_NAME}-green
        docker run -d --name ${APP_NAME}-blue
          -p 80:5000
          --restart unless-stopped
          $DOCKERHUB_USER/$APP_NAME:latest
      "
  tags:
    - dev
```

---

## Rollback Strategy

```yaml
# Manual rollback job
rollback_job:
  stage: deploy
  script:
    - ssh ubuntu@$DEPLOY_HOST "
      docker stop $APP_NAME || true &&
      docker rm $APP_NAME || true &&
      docker run -d --name $APP_NAME
      -p 80:5000
      --restart unless-stopped
      $DOCKERHUB_USER/$APP_NAME:$ROLLBACK_TAG
      "
  variables:
    ROLLBACK_TAG: "" # Set when triggering manually
  when: manual
  tags:
    - dev
```

---

## EC2 Security Group Configuration

| Rule Type | Protocol | Port Range | Source     | Description      |
| --------- | -------- | ---------- | ---------- | ---------------- |
| Inbound   | TCP      | 22         | Your IP/32 | SSH access       |
| Inbound   | TCP      | 80         | 0.0.0.0/0  | HTTP traffic     |
| Inbound   | TCP      | 443        | 0.0.0.0/0  | HTTPS traffic    |
| Inbound   | TCP      | 5000       | Your IP/32 | Dev/testing only |
| Outbound  | All      | All        | 0.0.0.0/0  | Internet access  |

---

## Post-Deployment Monitoring

```bash
# Check running containers
docker ps

# View logs
docker logs -f my-app

# Check resource usage
docker stats

# View container health
docker inspect --format='{{.State.Health.Status}}' my-app
```

---

## Complete Pipeline: Build → Test → Push → Deploy to EC2

```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  APP_NAME: "my-app"
  DOCKER_IMAGE: "$DOCKERHUB_USER/$APP_NAME"

build:
  stage: build
  script:
    - docker build -t $APP_NAME:$CI_COMMIT_SHORT_SHA .
    - docker tag $APP_NAME:$CI_COMMIT_SHORT_SHA $APP_NAME:latest
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
    - docker tag $APP_NAME:latest $DOCKER_IMAGE:latest
    - docker tag $APP_NAME:latest $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
  tags: [dev]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy:
  stage: deploy
  before_script:
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts 2>/dev/null
  script:
    - ssh -o StrictHostKeyChecking=no ubuntu@$DEPLOY_HOST "
      docker pull $DOCKER_IMAGE:latest &&
      docker stop $APP_NAME || true &&
      docker rm $APP_NAME || true &&
      docker run -d --name $APP_NAME
      -p 80:5000
      --restart unless-stopped
      $DOCKER_IMAGE:latest
      "
  environment:
    name: production
    url: http://$DEPLOY_HOST
  tags: [dev]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## 📌 Checklist: EC2 Deployment

- [ ] EC2 instance launched (Ubuntu, Docker installed)
- [ ] Security group allows ports 22, 80, 443
- [ ] `.pem` key downloaded and stored securely
- [ ] `SSH_PRIVATE_KEY` added as CI/CD variable (masked, protected)
- [ ] `DEPLOY_HOST` added as CI/CD variable
- [ ] Docker Hub credentials added as CI/CD variables
- [ ] `.gitlab-ci.yml` has SSH deploy job
- [ ] Pipeline tested and deployment verified ✅

---

> **Next:** [07 - Common Errors & Troubleshooting →](./07-common-errors.md)
