# 🚀 CI/CD Pipeline in GitLab

> Deep dive into `.gitlab-ci.yml` — stages, jobs, rules, artifacts, caching, environments, and production-ready templates.

---

## Pipeline Anatomy

A GitLab pipeline consists of **stages** (phases) and **jobs** (tasks within a phase):

```
   Pipeline
   ├── Stage: build
   │   └── Job: build_app
   ├── Stage: test
   │   ├── Job: unit_tests
   │   └── Job: lint_check
   ├── Stage: push
   │   └── Job: push_to_registry
   └── Stage: deploy
       └── Job: deploy_production
```

---

## Minimal Pipeline Example

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

build_job:
  stage: build
  script:
    - echo "Building the app..."

test_job:
  stage: test
  script:
    - echo "Running tests..."

deploy_job:
  stage: deploy
  script:
    - echo "Deploying..."
```

---

## Pipeline Triggers

| Trigger               | Description                              |
| --------------------- | ---------------------------------------- |
| `git push`            | Runs on every push to any branch         |
| Merge Request         | Triggered when MR is created/updated     |
| Manual                | Click "Run Pipeline" in GitLab UI        |
| Scheduled             | Cron-based schedules                     |
| API                   | Trigger via REST API                     |
| Downstream / Upstream | Cross-project or child pipeline triggers |

---

## Core Keywords Reference

### `stages`

Define the order of execution:

```yaml
stages:
  - build
  - test
  - security
  - push
  - deploy
```

### `script`

Commands to execute in the job:

```yaml
build_job:
  stage: build
  script:
    - npm install
    - npm run build
```

### `before_script` / `after_script`

Run commands before/after every job (or globally):

```yaml
default:
  before_script:
    - echo "Starting job at $(date)"
  after_script:
    - echo "Job finished at $(date)"
```

### `tags`

Route jobs to specific runners:

```yaml
build_job:
  tags:
    - dev
    - docker
```

### `only` / `except` (Legacy)

```yaml
deploy_job:
  only:
    - main # Run only on main branch
  except:
    - schedules # Skip scheduled pipelines
```

### `rules` (Modern — Preferred)

```yaml
deploy_job:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - when: never # Default: don't run
```

### `when`

| Value        | Behavior                                       |
| ------------ | ---------------------------------------------- |
| `on_success` | Run only if previous stage succeeded (default) |
| `on_failure` | Run only if previous stage failed              |
| `always`     | Always run regardless of status                |
| `manual`     | Requires manual click to trigger               |
| `delayed`    | Run after a delay (`start_in: 30 minutes`)     |
| `never`      | Never run (useful in `rules:`)                 |

### `needs` (DAG — Directed Acyclic Graph)

Skip waiting for the entire stage:

```yaml
test_job:
  stage: test
  needs: [build_job] # Start immediately after build_job, don't wait for full stage
  script:
    - npm test
```

### `dependencies`

Control which job artifacts are downloaded:

```yaml
deploy_job:
  stage: deploy
  dependencies:
    - build_job # Only download build_job's artifacts
  script:
    - ls build/
```

---

## Artifacts

Pass files between jobs across stages:

```yaml
build_job:
  stage: build
  script:
    - npm run build
  artifacts:
    paths:
      - build/
      - dist/
    expire_in: 1 hour # Auto-cleanup

test_job:
  stage: test
  script:
    - ls build/ # Available from build_job
```

### Artifact Reports

```yaml
test_job:
  artifacts:
    reports:
      junit: test-results.xml # Test report in GitLab UI
      dotenv: build.env # Pass vars to downstream jobs
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
```

---

## Caching

Cache dependencies to speed up subsequent runs:

```yaml
# Node.js example
build_job:
  stage: build
  cache:
    key: ${CI_COMMIT_REF_SLUG} # Cache per branch
    paths:
      - node_modules/
  script:
    - npm install
    - npm run build

# Python example
test_job:
  cache:
    key: pip-cache
    paths:
      - .pip-cache/
  script:
    - pip install --cache-dir=.pip-cache -r requirements.txt
    - pytest
```

### Cache vs Artifacts

| Feature    | Cache                     | Artifacts                       |
| ---------- | ------------------------- | ------------------------------- |
| Purpose    | Speed up builds           | Pass files between jobs         |
| Scope      | Same job across pipelines | Different jobs in same pipeline |
| Guaranteed | No (best effort)          | Yes (always available)          |
| Storage    | Runner-local or S3        | GitLab server                   |

---

## Environments & Deployments

```yaml
deploy_staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./deploy.sh staging

deploy_production:
  stage: deploy
  environment:
    name: production
    url: https://app.example.com
  script:
    - ./deploy.sh production
  when: manual
  only:
    - main
```

View deployments: GitLab → **Operate** → **Environments**

---

## Parallel & Matrix Jobs

```yaml
test:
  stage: test
  parallel:
    matrix:
      - NODE_VERSION: ["16", "18", "20"]
        OS: ["ubuntu", "alpine"]
  image: node:${NODE_VERSION}-${OS}
  script:
    - node --version
    - npm test
```

This creates 6 parallel jobs (3 versions × 2 OS variants).

---

## Multi-Project & Child Pipelines

### Trigger a Downstream Pipeline

```yaml
trigger_deploy:
  stage: deploy
  trigger:
    project: my-group/deploy-scripts
    branch: main
    strategy: depend # Wait for downstream to finish
```

### Child Pipeline

```yaml
trigger_child:
  stage: test
  trigger:
    include: path/to/child-pipeline.yml
    strategy: depend
```

---

## Production-Ready Template: Docker Build → Push → Deploy

```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  APP_NAME: "my-app"
  DOCKER_IMAGE: "$DOCKERHUB_USER/$APP_NAME"

# ──── Build ────
build_job:
  stage: build
  script:
    - docker build -t $APP_NAME:$CI_COMMIT_SHORT_SHA .
    - docker tag $APP_NAME:$CI_COMMIT_SHORT_SHA $APP_NAME:latest
  tags:
    - dev

# ──── Test ────
test_job:
  stage: test
  script:
    - docker run --rm -d --name test-container -p 5001:5000 $APP_NAME:latest
    - sleep 5
    - curl -f http://localhost:5001/health || exit 1
    - docker stop test-container
  tags:
    - dev

# ──── Push to Docker Hub ────
push_job:
  stage: push
  script:
    - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS
    - docker tag $APP_NAME:latest $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker tag $APP_NAME:latest $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_IMAGE:latest
  tags:
    - dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ──── Deploy ────
deploy_job:
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
  tags:
    - dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## Validate Your Pipeline

```bash
# Via GitLab UI
# Go to: Build → Pipeline editor → Validate

# Via API
curl --header "Content-Type: application/json" \
     --header "PRIVATE-TOKEN: <PAT>" \
     --data '{"content": "'"$(cat .gitlab-ci.yml)"'"}' \
     "https://gitlab.com/api/v4/ci/lint"
```

---

## 📌 Quick Reference Keywords

| Keyword          | Purpose                             |
| ---------------- | ----------------------------------- |
| `stages`         | Define pipeline phases              |
| `script`         | Commands to run                     |
| `tags`           | Route to specific runners           |
| `rules`          | Conditional job execution           |
| `only`/`except`  | Legacy branch filtering             |
| `when`           | Manual, delayed, on_failure, always |
| `needs`          | DAG — skip stage wait               |
| `artifacts`      | Pass files between jobs             |
| `cache`          | Speed up repeated builds            |
| `environment`    | Track deployments                   |
| `trigger`        | Multi-project / child pipelines     |
| `parallel`       | Matrix / parallel execution         |
| `variables`      | Inline variable definitions         |
| `allow_failure`  | Don't fail pipeline if job fails    |
| `retry`          | Auto-retry failed jobs (max 2)      |
| `timeout`        | Job timeout (e.g., `30 minutes`)    |
| `interruptible`  | Cancel job if newer pipeline starts |
| `resource_group` | Prevent concurrent deploys          |

---

> **Next:** [05 - Docker Integration →](./05-docker-integration.md)
