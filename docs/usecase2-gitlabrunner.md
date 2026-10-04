--8<-- "snippets/dt-enablement.md"

# Use Case 2 — First GitLab Project & Runner

In this use case you will create your first project on **GitLab.com**, connect your Codespace to it over SSH, and install your own **GitLab Runner** directly inside the Codespace so it can execute pipelines against the Kubernetes cluster and Docker daemon already running there.

---

## 1. Create your first GitLab project

!!! example "Step-by-step"
    1. Log in to [gitlab.com](https://gitlab.com) (or create a free account)
    2. Click **Create new... → New project/repository → Create blank project**
    3. Project name: `firstproject`, visibility: your choice (Private is fine)
    4. Leave "Initialize repository with a README" **checked**, then **Create project**

![Create a new blank project in GitLab](img/usecase1-create-project.png)

---

## 2. Create your first pipeline

In your new project, create a `.gitlab-ci.yml` file directly from the GitLab web interface:

!!! example "Step-by-step"
    1. In your project, click **+ (Create new file) → New file**
    2. Name the file `.gitlab-ci.yml`
    3. Paste the pipeline definition below, then click **Commit changes**

```yaml
# My first GitLab CI pipeline
stages:
  - build
  - deploy

build-job:
  stage: build
  tags:
    - shell
  script:
    - echo "Hello, $USER!"
    - echo "compile my first project"
    - echo "compile completed"

deploy-job:
  stage: deploy
  tags:
    - shell
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
```

![Creating .gitlab-ci.yml in the GitLab web editor](img/usecase1-create-pipeline.png)

!!! info "Nothing will run yet"
    Open ** → Pipelines** in GitLab and you'll see the pipeline stuck **pending** — there is no runner registered against this project yet. That's what the rest of this use case fixes.

![Pipeline stuck in pending state — no runner registered](img/usecase1-pipeline-pending.png)

---

## 3. Install GitLab Runner inside the Codespace

The Codespace already has Docker, `kubectl`, `k3d`, and Node.js installed and a k3d cluster running — installing the runner as a **shell executor** lets every pipeline job use those tools directly, with no extra Docker-in-Docker setup.

!!! note "Already installed in Use Case 1?"
    Use Case 1 installs the runner binary for you. If `gitlab-runner --version` already works in your terminal, skip the commands below and continue at section 4 to register a runner for this new project.

```bash
# Download the binary for the codespace's architecture (amd64 shown; use arm64 on Apple Silicon)
sudo curl -L --output /usr/local/bin/gitlab-runner \
  https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a dedicated GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```

![Running the GitLab Runner installation commands in the Codespace terminal](img/usecase1-install-runner.png)

---

## 4. Register the runner against your project

GitLab.com no longer uses the old shared "registration token" — you create the runner in the UI first and get a one-time **authentication token** (`glrt-...`).

!!! example "Step-by-step — Create the runner in GitLab UI"
    1. In your `firstproject` GitLab project, go to **Settings → CI/CD → Runners**
    2. Click **New project runner**
    3. Tags: enter `shell` (matches the tag used by the jobs above); check **"Run untagged jobs"** if you want it to also pick up jobs with no tags
    4. Click **Create runner**

![Settings → CI/CD → Runners → New project runner](img/usecase1-create-runner-ui.png)

!!! example "Step-by-step — Copy the registration token"
    5. After clicking **Create runner**, GitLab shows a one-time registration command — copy the `--token glrt-...` value from it

![Registering the runner in the Codespace terminal](img/usecase1-runner-create.png)

Register it non-interactively from the Codespace terminal:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --token "<glrt-...-paste-your-token-here>" \
  --executor "shell" \
  --description "codespace-shell-runner"
```

Start the runner:

```bash
gitlab-runner run
```

![Registering the runner in the Codespace terminal](img/usecase1-runner-register.png)


Back in GitLab, **Settings → CI/CD → Runners** should now show your runner as **online** (green dot).

---

## 5. Watch the pipeline run

Both `build-job` and `deploy-job` should turn green. Click into a job to see its log — you'll also see the runner picking up jobs in the Codespace terminal where `gitlab-runner run` is executing.

---

## 6. Hands-On Exercise: Understanding Runner Tags

Tags decide which jobs a runner is allowed to pick up. A job only runs on a runner if **every tag the job lists** is also present on the runner (the runner may have *more* tags than the job requires).

| | Runner tags | Job tags | Runs? |
|---|---|---|---|
| Example 1 | `[docker, shell, codespace]` | `[shell]` | ✅ yes — the job's required tag is a subset of the runner's tags |
| Example 2 | `[docker, shell, codespace]` | `[docker, shell, codespace]` | ✅ yes — exact match |
| Example 3 | `[docker, shell, codespace]` | `[docker, shell, k8s]` | ❌ no — `k8s` isn't one of the runner's tags, so it stays **pending** forever |

### Try it yourself — directly in the GitLab UI

You will modify `.gitlab-ci.yml` using the **GitLab web editor**, commit the change, and observe what happens to the pipeline — no terminal or `git push` needed.

---

**Exercise A — Make the pipeline go pending by using a tag the runner doesn't have**

!!! example "Step-by-step"
    1. In your project, open `.gitlab-ci.yml` and click the **Edit** (pencil) icon
    2. Change the `tags:` on **both jobs** to `k8s` (a tag your runner does *not* have):

```yaml
# My first GitLab CI pipeline
stages:
  - build
  - deploy

build-job:
  stage: build
  tags:
    - k8s         # ← changed from shell
  script:
    - echo "Hello, $USER!"
    - echo "compile my first project"
    - echo "compile completed"

deploy-job:
  stage: deploy
  tags:
    - k8s         # ← changed from shell
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
```

    3. Scroll down, enter a commit message such as `test: use wrong runner tag`, and click **Commit changes**
    4. Navigate to **Build → Pipelines** — the jobs should be **stuck pending** immediately

![Pipeline jobs stuck in pending because no runner matches the k8s tag](img/usecase1-tags-pending.png)

---

**Exercise B — Restore the correct tag and watch it recover**

!!! example "Step-by-step"
    1. Open `.gitlab-ci.yml` in the web editor again
    2. Change **both** `k8s` tags back to `shell`:

```yaml
# My first GitLab CI pipeline
stages:
  - build
  - deploy

build-job:
  stage: build
  tags:
    - shell       # ← restored
  script:
    - echo "Hello, $USER!"
    - echo "compile my first project"
    - echo "compile completed"

deploy-job:
  stage: deploy
  tags:
    - shell       # ← restored
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
```

    3. Commit with message `fix: restore shell runner tag`
    4. **CI/CD → Pipelines** — the new pipeline should be picked up immediately and both jobs pass

![Pipeline runs successfully after restoring the correct tag](img/usecase1-tags-success.png)

!!! tip "Key takeaway"
    Runner tags are a routing mechanism — they let you direct jobs to the right environment (e.g. `shell` for a Codespace, `k8s` for a Kubernetes executor, `docker` for a Docker executor). A mismatch silently queues the job forever with no error message.

---

## 7. Bonus Challenge — Branch-Conditional Deployments

??? question "Challenge: Run `deploy-job` only on `main`, skip it on all other branches"

    Real pipelines should not deploy on every commit to every branch. Modify your `.gitlab-ci.yml` so that:

    - `build-job` **always runs** on any branch
    - `deploy-job` **only runs** when the commit is on the `main` branch, and is **skipped** on feature branches

    Edit the file directly in the GitLab UI. Commit to a feature branch first to verify the deploy job is skipped, then merge to `main` and confirm it runs.

??? success "Reveal the answer"

    Use GitLab CI's `rules:` keyword with the built-in `$CI_COMMIT_BRANCH` variable:

    ```yaml
    # My first GitLab CI pipeline
    stages:
      - build
      - deploy

    build-job:
      stage: build
      tags:
        - shell
      script:
        - echo "Hello, $USER!"
        - echo "compile my first project"
        - echo "compile completed"

    deploy-job:
      stage: deploy
      tags:
        - shell
      rules:
        - if: '$CI_COMMIT_BRANCH == "main"'
      script:
        - echo "Deploying application..."
        - echo "Application successfully deployed."
    ```

    **How to test it:**

    1. In the GitLab UI, create a new branch — e.g. `feature/test-rules` — from **Repository → Branches → New branch**
    2. Edit `.gitlab-ci.yml` on that branch to add the `rules:` block above, then commit
    3. Navigate to **CI/CD → Pipelines** — `build-job` should run but `deploy-job` shows as **skipped**
    4. Merge the branch into `main` (or commit the same change directly on `main`)
    5. The next pipeline on `main` should run **both** jobs


    **Why `rules:` instead of `only:`?**
    `rules:` is the modern replacement for the older `only:/except:` syntax. It evaluates conditions top-to-bottom and supports complex logic — multiple `if:`, `changes:`, and `exists:` conditions can be combined in one block. GitLab recommends `rules:` for all new pipelines.

---

<div class="grid cards" markdown>
- [Continue to Use Case 3 — Node.js CI, Test & SAST :octicons-arrow-right-24:](usecase3-nodejs-sast.md)
</div>
