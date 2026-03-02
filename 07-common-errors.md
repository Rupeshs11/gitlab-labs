# 🐛 Common Errors & Troubleshooting

> Solutions to the most frequently encountered GitLab CI/CD issues.

---

## Runner Issues

### ❌ Pipeline Stuck on "Waiting for Runner"

**Symptoms:** Pipeline shows "This job is stuck because you don't have any active runners that can run this job."

**Fixes:**

1. **Check if runner is registered:**

   ```bash
   sudo gitlab-runner list
   ```

   If empty → register a runner (see [02-runners-explained.md](./02-runners-explained.md))

2. **Check runner status:**

   ```bash
   sudo gitlab-runner status
   sudo systemctl status gitlab-runner
   ```

   If inactive → restart:

   ```bash
   sudo gitlab-runner restart
   ```

3. **Tag mismatch:** Your job specifies tags that no runner has

   ```yaml
   # Job requires 'production' tag
   deploy_job:
     tags:
       - production # ← No runner has this tag!
   ```

   **Fix:** Edit runner in GitLab UI → add the required tag, or update `.gitlab-ci.yml` tags.

4. **Runner is locked to another project:**
   GitLab → Settings → CI/CD → Runners → Edit → Uncheck "Lock to current projects"

---

### ❌ Runner Offline (Gray Circle)

```bash
# Restart the runner service
sudo gitlab-runner restart

# Or restart the systemd service
sudo systemctl restart gitlab-runner

# Verify connectivity
sudo gitlab-runner verify
```

If still offline → check firewall rules, ensure EC2 can reach `https://gitlab.com`.

---

### ❌ Token Invalid / Registration Failed

**`ERROR: Registering runner... failed: 403 Forbidden`**

- Ensure you're using the **full** `glrt-...` token on one line, no spaces
- Token may have expired → generate a new one in GitLab UI
- Don't include `--tag-list` with new `glrt-` tokens (tags are set in UI)

```bash
# Correct — no --tag-list
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token glrt-xxxxxxxxxxxxxxxxxxxx \
  --executor shell \
  --non-interactive
```

---

## Docker Issues

### ❌ Permission Denied for Docker

**`Got permission denied while trying to connect to the Docker daemon socket`**

```bash
# Add gitlab-runner to docker group
sudo usermod -aG docker gitlab-runner

# Restart runner
sudo gitlab-runner restart

# Verify
sudo -u gitlab-runner docker ps
```

> ⚠️ If using the `docker` executor, ensure Docker-in-Docker (DinD) is configured.

---

### ❌ Docker Daemon Not Running

**`Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`**

```bash
# Start Docker
sudo systemctl start docker

# Enable on boot
sudo systemctl enable docker

# Check status
sudo systemctl status docker
```

---

### ❌ Disk Space Full on Runner

```bash
# Check disk usage
df -h

# Clean up Docker
docker system prune -af --volumes
docker image prune -af --filter "until=72h"

# Check specific Docker disk usage
docker system df
```

**Prevention:** Add a cleanup job to your pipeline:

```yaml
cleanup:
  stage: .post
  script:
    - docker system prune -af --volumes
  when: always
```

---

### ❌ Docker Build Cache Issues

**Stale build:** Image not reflecting code changes.

```bash
# Force rebuild without cache
docker build --no-cache -t myapp:latest .
```

In CI/CD:

```yaml
build_job:
  script:
    - docker build --no-cache -t $APP_NAME:latest .
```

---

## SSH & Deployment Issues

### ❌ SSH Connection Refused

1. **Check EC2 Security Group** → Port 22 must allow the runner's IP
2. **Verify SSH key:**
   ```bash
   ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no ubuntu@$DEPLOY_HOST "echo connected"
   ```
3. **Key format issues:** Ensure `SSH_PRIVATE_KEY` variable has:
   - Full content including `-----BEGIN ... PRIVATE KEY-----` and `-----END ... PRIVATE KEY-----`
   - No extra spaces or newlines at the end
   - Correct permissions (set in pipeline):
     ```yaml
     before_script:
       - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
       - chmod 600 ~/.ssh/id_rsa
     ```

---

### ❌ Host Key Verification Failed

```bash
# The following error:
# Host key verification failed.

# Fix: Add host to known_hosts before SSH
ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts 2>/dev/null

# Or use StrictHostKeyChecking=no (less secure)
ssh -o StrictHostKeyChecking=no ubuntu@$DEPLOY_HOST
```

---

### ❌ Remote Command Exited with Error

If SSH connects but the remote command fails:

```yaml
deploy_job:
  script:
    # Use set -e to stop on first error and see which command fails
    - ssh ubuntu@$DEPLOY_HOST "
      set -e
      echo 'Pulling image...'
      docker pull $DOCKER_IMAGE:latest
      echo 'Stopping old container...'
      docker stop $APP_NAME || true
      docker rm $APP_NAME || true
      echo 'Starting new container...'
      docker run -d --name $APP_NAME -p 80:5000 $DOCKER_IMAGE:latest
      echo 'Done!'
      "
```

---

## Pipeline & YAML Issues

### ❌ YAML Syntax Error

**`(.gitlab-ci.yml): did not find expected key while parsing a block mapping`**

Common causes:

- **Incorrect indentation** (YAML uses spaces, not tabs)
- **Missing quotes** around special characters
- **Missing colons** after keys

**Validate your YAML:**

- GitLab → **Build** → **Pipeline editor** → Validates automatically
- Online: [YAML Lint](https://www.yamllint.com/)

---

### ❌ Job Not Running / Being Skipped

| Cause                                      | Fix                                |
| ------------------------------------------ | ---------------------------------- |
| `rules:` condition not matching            | Check branch name, pipeline source |
| `only:` wrong branch                       | Verify branch name matches exactly |
| `when: manual`                             | Click the play button in GitLab UI |
| Tags don't match                           | Ensure runner has required tags    |
| Protected variable on non-protected branch | Unprotect the variable or branch   |

**Debug rules:**

```yaml
debug_job:
  script:
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Pipeline Source: $CI_PIPELINE_SOURCE"
    - echo "Tag: $CI_COMMIT_TAG"
  rules:
    - when: always # ← Always run to debug
```

---

### ❌ Artifacts Not Being Passed

```yaml
build:
  artifacts:
    paths:
      - build/
    expire_in: 1 hour # ← Ensure not expired

test:
  dependencies:
    - build # ← Explicitly specify dependency
  script:
    - ls build/ # Verify artifact exists
```

---

### ❌ Cache Not Working

```yaml
build:
  cache:
    key: ${CI_COMMIT_REF_SLUG} # ← Must be same key in jobs sharing cache
    paths:
      - node_modules/
    policy: pull-push # ← push creates cache, pull reads it
```

**Common causes:**

- Different `cache.key` in producer and consumer jobs
- Cache cleared by GitLab (not guaranteed to persist)
- Using shared runners (cache is per runner → use S3/GCS distributed cache)

---

## Variable Issues

### ❌ Variable Not Available in Job

| Symptom                     | Cause                                        | Fix                                      |
| --------------------------- | -------------------------------------------- | ---------------------------------------- |
| Variable is empty           | Protected variable + non-protected branch    | Unprotect variable or protect the branch |
| Variable not masked in logs | Value doesn't meet masking rules (< 8 chars) | Make value longer or use `File` type     |
| Expansion not working       | `$VAR` inside single quotes                  | Use double quotes: `"$VAR"` not `'$VAR'` |

### Debugging Variables

```yaml
debug_vars:
  script:
    - echo "DOCKERHUB_USER=$DOCKERHUB_USER"
    - echo "APP_NAME=$APP_NAME"
    - echo "CI_COMMIT_BRANCH=$CI_COMMIT_BRANCH"
    # Never echo masked variables!
    - if [ -z "$SSH_PRIVATE_KEY" ]; then echo "SSH key NOT SET"; else echo "SSH key is set"; fi
```

---

## Container Registry Issues

### ❌ Docker Login Failed to GitLab Registry

```bash
# Error: unauthorized: HTTP Basic: Access denied

# Ensure using correct predefined variables:
docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

# For manual login:
docker login registry.gitlab.com
# Username: your_gitlab_username
# Password: your_personal_access_token (with read_registry + write_registry scopes)
```

---

## Network & Connectivity Issues

### ❌ `curl: (7) Failed to connect`

In health checks or API calls:

```yaml
test_job:
  script:
    - docker run -d --name test-app -p 5001:5000 $APP_NAME:latest
    - sleep 10 # ← Increase wait time
    - curl -f --retry 3 --retry-delay 5 http://localhost:5001/health || exit 1
    - docker stop test-app
```

### ❌ `Could not resolve host`

```bash
# Check DNS on runner
nslookup gitlab.com
dig gitlab.com

# Ensure runner has internet access
curl -I https://gitlab.com
```

---

## ⚡ Quick Diagnosis Checklist

```bash
# 1. Is the runner registered and running?
sudo gitlab-runner list
sudo gitlab-runner status

# 2. Does the runner have Docker access?
sudo -u gitlab-runner docker ps

# 3. Is the runner connected to GitLab?
sudo gitlab-runner verify

# 4. Is the disk full?
df -h
docker system df

# 5. Are the CI/CD variables set?
# Check in: Settings → CI/CD → Variables

# 6. Is the .gitlab-ci.yml valid?
# Check in: Build → Pipeline editor

# 7. Runner logs
sudo journalctl -u gitlab-runner -f
```

---

## 📌 Error Quick Reference Table

| Error                         | Root Cause                         | Quick Fix                                    |
| ----------------------------- | ---------------------------------- | -------------------------------------------- |
| Stuck on "Waiting for runner" | No runner / tag mismatch           | Register runner, fix tags                    |
| Runner offline                | Service stopped                    | `sudo gitlab-runner restart`                 |
| Token invalid (403)           | Expired / malformed token          | Generate new token in GitLab UI              |
| `--tag-list` error            | Using old syntax                   | Remove `--tag-list`, set tags in UI          |
| Docker permission denied      | gitlab-runner not in group         | `sudo usermod -aG docker gitlab-runner`      |
| Docker daemon not running     | Docker service stopped             | `sudo systemctl start docker`                |
| SSH connection refused        | Security group / key issue         | Check SG port 22, verify key                 |
| Host key verification failed  | Missing known_hosts                | `ssh-keyscan -H $HOST >> ~/.ssh/known_hosts` |
| YAML syntax error             | Bad indentation / format           | Use Pipeline editor to validate              |
| Variable empty                | Protected var + unprotected branch | Unprotect variable                           |
| Disk space full               | Docker images piling up            | `docker system prune -af`                    |

---

> **← Back to:** [README →](./README.md)
