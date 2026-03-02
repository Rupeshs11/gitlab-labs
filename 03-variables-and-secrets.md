# 🔐 Variables & Secrets in GitLab CI/CD

> How to securely manage environment variables, secrets, and credentials in your pipelines.

---

## Why Use CI/CD Variables?

Hardcoding secrets (passwords, API keys, tokens) in `.gitlab-ci.yml` or your codebase is a **critical security risk**. GitLab CI/CD variables let you:

- Store sensitive data **outside** your code
- **Mask** values in job logs
- **Protect** variables to run only on protected branches
- Scope variables to **specific environments**

---

## Types of Variables

| Type                     | Scope                    | Defined In                    |
| ------------------------ | ------------------------ | ----------------------------- |
| **Project Variables**    | Single project           | Settings → CI/CD → Variables  |
| **Group Variables**      | All projects in a group  | Group → Settings → CI/CD      |
| **Instance Variables**   | All projects on instance | Admin → Settings → CI/CD      |
| **Predefined Variables** | Auto-available in jobs   | GitLab provides automatically |
| **Pipeline Variables**   | Single pipeline run      | Run Pipeline → Add variable   |
| **Job Variables**        | Single job in `.yml`     | `variables:` keyword in YAML  |

---

## Setting Up Project Variables

### Via GitLab UI

1. Go to your project → **Settings** → **CI/CD**
2. Expand **Variables**
3. Click **Add variable**

| Field         | Description                                             |
| ------------- | ------------------------------------------------------- |
| **Key**       | Variable name (e.g., `DOCKERHUB_PASS`)                  |
| **Value**     | The secret value                                        |
| **Type**      | `Variable` (plain text) or `File` (writes to temp file) |
| **Protected** | Only available on protected branches/tags               |
| **Masked**    | Hidden in job logs (value must meet masking rules)      |
| **Expanded**  | Whether `$VAR` references inside are resolved           |

### Common Variables to Set

| Variable                | Example Value           | Masked | Protected |
| ----------------------- | ----------------------- | ------ | --------- |
| `DOCKERHUB_USER`        | `yourusername`          | ❌     | ❌        |
| `DOCKERHUB_PASS`        | `dckr_pat_xxxxxxxx`     | ✅     | ❌        |
| `SSH_PRIVATE_KEY`       | Contents of `.pem` file | ✅     | ✅        |
| `DEPLOY_HOST`           | `54.123.45.67`          | ❌     | ✅        |
| `AWS_ACCESS_KEY_ID`     | `AKIA...`               | ✅     | ✅        |
| `AWS_SECRET_ACCESS_KEY` | `wJal...`               | ✅     | ✅        |
| `KUBE_CONFIG`           | Base64 kubeconfig       | ✅     | ✅        |

---

## Using Variables in `.gitlab-ci.yml`

### Basic Usage

```yaml
variables:
  APP_NAME: "my-app" # Inline variable
  DEPLOY_ENV: "production"

build_job:
  stage: build
  script:
    - echo "Building $APP_NAME" # Shell expansion
    - docker build -t $APP_NAME:latest .
    - echo "Deploying to $DEPLOY_ENV"
```

### Using Protected CI/CD Variables

```yaml
push_job:
  stage: push
  script:
    - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS # From CI/CD settings
    - docker push $DOCKERHUB_USER/$APP_NAME:latest
  tags:
    - dev
```

### File-Type Variables

When you set a variable as type `File`, GitLab writes the value to a temp file and sets the variable to the **file path**:

```yaml
deploy_job:
  stage: deploy
  script:
    - mkdir -p ~/.ssh
    - cp $SSH_PRIVATE_KEY ~/.ssh/id_rsa # $SSH_PRIVATE_KEY = path to temp file
    - chmod 600 ~/.ssh/id_rsa
    - ssh -i ~/.ssh/id_rsa ubuntu@$DEPLOY_HOST "echo deployed"
```

> **Tip:** Use `File` type for SSH keys, kubeconfig, and certificates.

---

## Variable Precedence (Highest to Lowest)

```
1. Pipeline-level variables (manual run)
2. Job-level variables (in .gitlab-ci.yml)
3. Project-level variables (Settings → CI/CD)
4. Group-level variables
5. Instance-level variables
6. Predefined variables
```

> If the same variable is defined at multiple levels, the **highest** level wins.

---

## Predefined Variables (Auto-Available)

GitLab provides many built-in variables. The most useful ones:

| Variable              | Description                 | Example Value                      |
| --------------------- | --------------------------- | ---------------------------------- |
| `CI_PROJECT_NAME`     | Project name                | `my-app`                           |
| `CI_COMMIT_SHA`       | Full commit hash            | `abc123def456...`                  |
| `CI_COMMIT_SHORT_SHA` | Short commit hash           | `abc123d`                          |
| `CI_COMMIT_BRANCH`    | Branch name                 | `main`                             |
| `CI_COMMIT_TAG`       | Tag name (if tagged)        | `v1.2.0`                           |
| `CI_COMMIT_MESSAGE`   | Commit message              | `fix: resolve bug`                 |
| `CI_PIPELINE_ID`      | Pipeline ID                 | `123456`                           |
| `CI_JOB_ID`           | Job ID                      | `789012`                           |
| `CI_JOB_NAME`         | Job name                    | `build_job`                        |
| `CI_REGISTRY`         | Container registry URL      | `registry.gitlab.com`              |
| `CI_REGISTRY_IMAGE`   | Full image path             | `registry.gitlab.com/user/project` |
| `CI_PROJECT_DIR`      | Repo checkout directory     | `/builds/user/project`             |
| `CI_ENVIRONMENT_NAME` | Environment name            | `production`                       |
| `GITLAB_USER_LOGIN`   | User who triggered pipeline | `john_doe`                         |

### Example: Dynamic Image Tagging

```yaml
build_job:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker build -t $CI_REGISTRY_IMAGE:latest .
  tags:
    - dev
```

---

## Masking & Protecting Variables

### Masking Rules

A variable can be masked only if its value:

- Is at least **8 characters** long
- Contains only characters from: `a-zA-Z0-9`, `@:.~`, and certain special characters
- Doesn't contain newlines

```
Masked variable in logs:
$ echo $DOCKERHUB_PASS
[MASKED]
```

### Protecting Variables

Protected variables are only available to:

- **Protected branches** (e.g., `main`, `release/*`)
- **Protected tags** (e.g., `v*`)

This prevents feature branches from accessing production secrets.

---

## Variable Scopes with Environments

```yaml
variables:
  APP_NAME: "my-app"

deploy_staging:
  stage: deploy
  environment:
    name: staging
  variables:
    DEPLOY_URL: "staging.example.com" # Only in this job
  script:
    - echo "Deploying to $DEPLOY_URL"

deploy_production:
  stage: deploy
  environment:
    name: production
  variables:
    DEPLOY_URL: "app.example.com" # Different value
  script:
    - echo "Deploying to $DEPLOY_URL"
  when: manual
```

### Environment-Scoped Variables (via UI)

When adding a variable in Settings → CI/CD → Variables, you can scope it to a specific environment:

| Variable  | Value            | Environment  |
| --------- | ---------------- | ------------ |
| `DB_HOST` | `staging-db.rds` | `staging`    |
| `DB_HOST` | `prod-db.rds`    | `production` |

---

## Using `.env` Files with Artifacts

Pass variables between jobs using dotenv artifacts:

```yaml
generate_vars:
  stage: build
  script:
    - echo "VERSION=1.2.3" >> build.env
    - echo "BUILD_DATE=$(date +%F)" >> build.env
  artifacts:
    reports:
      dotenv: build.env # ← Makes vars available to downstream jobs

deploy_job:
  stage: deploy
  needs: [generate_vars]
  script:
    - echo "Deploying version $VERSION built on $BUILD_DATE"
```

---

## 📌 Quick Reference

```bash
# List all predefined variables in a job
env | grep CI_

# Debug: print all variables (DON'T do in production)
printenv

# Check if a variable is set
if [ -z "$MY_VAR" ]; then echo "NOT SET"; fi
```

### GitLab API: Manage Variables Programmatically

```bash
# List project variables
curl --header "PRIVATE-TOKEN: <your_pat>" \
  "https://gitlab.com/api/v4/projects/<project_id>/variables"

# Create a variable
curl --request POST \
  --header "PRIVATE-TOKEN: <your_pat>" \
  --form "key=NEW_VAR" \
  --form "value=my_value" \
  --form "masked=true" \
  "https://gitlab.com/api/v4/projects/<project_id>/variables"
```

---

> **Next:** [04 - CI/CD Pipeline →](./04-ci-cd-pipeline.md)
