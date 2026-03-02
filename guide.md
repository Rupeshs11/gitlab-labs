# GitLab CI/CD Setup Guide — Self-Hosted Runner

> A reusable, step-by-step guide to set up GitLab CI/CD with a self-hosted runner on any project.

---

## 1. Create GitLab Project

1. Go to [gitlab.com](https://gitlab.com) → **New Project** → **Create blank project**
2. Push existing code:
   ```bash
   git remote add gitlab https://gitlab.com/<username>/<repo>.git
   git push gitlab main
   ```

> **Tip:** To push to GitHub and GitLab at once:
>
> ```bash
> git remote set-url --add origin https://gitlab.com/<username>/<repo>.git
> ```

---

## 2. Install GitLab Runner on EC2

```bash
# Install
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt install gitlab-runner -y
```

---

## 3. Register the Runner

### Get Token

GitLab → **Settings** → **CI/CD** → **Runners** → **New project runner** → Set tags → **Create runner** → Copy `glrt-...` token

### Register

```bash
sudo gitlab-runner register \
  --url https://gitlab.com \
  --token <PASTE_TOKEN> \
  --description "my-runner" \
  --executor shell \
  --non-interactive
```

> ⚠️ With new `glrt-` tokens, tags are set in **GitLab UI only**. Do NOT pass `--tag-list` in the command.

### Grant Docker Access

```bash
sudo usermod -aG docker gitlab-runner
sudo gitlab-runner restart
```

### Verify

```bash
sudo gitlab-runner list    # Should show your runner
```

---

## 4. Add CI/CD Variables

GitLab → **Settings** → **CI/CD** → **Variables**

| Variable          | Example                 | Masked |
| ----------------- | ----------------------- | ------ |
| `DOCKERHUB_USER`  | `yourusername`          | ❌     |
| `DOCKERHUB_PASS`  | `dckr_pat_xxx`          | ✅     |
| `SSH_PRIVATE_KEY` | Contents of `.pem` file | ✅     |
| `DEPLOY_HOST`     | `54.123.45.67`          | ❌     |

---

## 5. Create `.gitlab-ci.yml`

### Template: Docker Build + Push + Deploy

```yaml
stages:
  - build
  - test
  - push
  - deploy

build_job:
  stage: build
  script:
    - docker build -t $APP_NAME:latest .
  tags:
    - dev

test_job:
  stage: test
  script:
    - docker run --rm -d --name test-container -p 5001:5000 $APP_NAME:latest
    - sleep 5
    - curl -f http://localhost:5001/health || exit 1
    - docker stop test-container
  tags:
    - dev

push_job:
  stage: push
  script:
    - docker login -u $DOCKERHUB_USER -p $DOCKERHUB_PASS
    - docker image tag $APP_NAME:latest $DOCKERHUB_USER/$APP_NAME:latest
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
```

### Template: SSH Deploy to Remote EC2

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
      docker pull $DOCKERHUB_USER/$APP_NAME:latest &&
      docker stop $APP_NAME || true &&
      docker rm $APP_NAME || true &&
      docker run -d --name $APP_NAME -p 80:5000 --restart unless-stopped $DOCKERHUB_USER/$APP_NAME:latest
      "
  tags:
    - dev
```

---

## 6. Useful Keywords

| Keyword         | Purpose                          | Example                             |
| --------------- | -------------------------------- | ----------------------------------- |
| `only`          | Run only on specific branches    | `only: [main]`                      |
| `except`        | Skip specific branches           | `except: [dev]`                     |
| `when: manual`  | Require manual click to run      | For deploy stages                   |
| `needs`         | Run after specific job           | `needs: [build_job]`                |
| `artifacts`     | Pass files between stages        | `artifacts: { paths: [build/] }`    |
| `cache`         | Cache dependencies               | `cache: { paths: [node_modules/] }` |
| `allow_failure` | Don't fail pipeline if job fails | `allow_failure: true`               |

---

## 7. Troubleshooting

| Problem                                | Fix                                                                       |
| -------------------------------------- | ------------------------------------------------------------------------- |
| Pipeline stuck on "Waiting for runner" | Check `sudo gitlab-runner list` — if empty, register it                   |
| Token invalid error                    | Paste the **full** `glrt-...` token in one line                           |
| `--tag-list` error                     | Remove it — tags are set in GitLab UI with new tokens                     |
| Docker permission denied               | Run `sudo usermod -aG docker gitlab-runner && sudo gitlab-runner restart` |
| Runner offline (gray circle)           | Run `sudo gitlab-runner restart` on EC2                                   |

---

## Quick Checklist

- [ ] GitLab project created
- [ ] Runner installed on EC2
- [ ] Runner registered with token
- [ ] Docker permission granted to `gitlab-runner`
- [ ] CI/CD variables added in GitLab
- [ ] `.gitlab-ci.yml` created in project root
- [ ] Pipeline triggered and running ✅
