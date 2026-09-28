--8<-- "snippets/dt-enablement.md"

# Use Case 2 — Node.js CI, Test & SAST

Now that a runner is alive, let's give it something real to build: **kkm-pulse-demo**, a small Express.js app already sitting in this Codespace at `.devcontainer/apps/kkm-pulse-demo`. You'll push it to its own GitLab project, then progressively build a pipeline: install → test → static analysis (SAST) → SonarQube quality gate.

---

## 1. Look at the app

```bash
cd .devcontainer/apps/kkm-pulse-demo
ls
```

```text
kkm-pulse-demo/
├── .gitlab-ci.yaml
├── Dockerfile
├── package.json
├── server.js
├── test/
│   └── server.test.js
└── views/
    └── index.html
```

It's a plain Express app (`server.js`) with two API routes (`/api/status`, `/api/trigger-anomaly`) and a Jest + Supertest test suite (`test/server.test.js`). Try it locally first:

```bash
npm install
npm test
npm start
```

`npm start` prints `KKM Pulse App running hot on port 3000` — stop it with **Ctrl+C** once you've confirmed it works; GitLab CI will run it for real in a moment.

---


## 2. Connect your Codespace to GitLab over SSH

### Generate an SSH key in the Codespace

```bash
ssh-keygen
```

Press **Enter** through the prompts to accept the defaults (no passphrase needed for a throwaway Codespace).

### Copy the public key into GitLab

```bash
cat ~/.ssh/id_ed25519.pub
```

1. In GitLab, click your avatar → **Edit profile** → **SSH Keys** (or go directly to **User Settings → SSH Keys**)
2. Click **Add new key**, paste the output above into **Key**, give it a title, change **Expiry Date** and click **Add key**

---

## 2. Create the project in GitLab and push

!!! example "Step-by-step"
    1. On [gitlab.com](https://gitlab.com), click **Create new... → New project/repository → Create blank project**
    2. Project name: `kkm-pulse-demo`, **do not** initialize with a README (we're pushing existing history)
    3. Click **Create project** and copy the SSH clone URL, e.g. `git@gitlab.com:<your-username>/kkm-pulse-demo.git`

From inside `.devcontainer/apps/kkm-pulse-demo` (it's already its own git repository):

```bash
git remote remove origin 2>/dev/null || true
git remote add origin git@gitlab.com:<your-username>/kkm-pulse-demo.git
git remote set-url origin git@gitlab.com:<your-username>/kkm-pulse-demos.git
git add .
git commit -m "initial commit: kkm-pulse-demo" --allow-empty
git push --set-upstream origin main
```

---

## 3. Register a runner for this project

Runners are registered per-project on GitLab.com, so `kkm-pulse-demo` needs its own registration (the same `gitlab-runner` service from Use Case 1 can hold multiple registrations at once).

1. In the `kkm-pulse-demo` project: **Settings → CI/CD → Runners → New project runner**
2. Tags: `shell`, and Create the Runner 
3. Install the Gitlab Runner

  ```
  # Download the binary for your system
  sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

  # Give it permission to execute
  sudo chmod +x /usr/local/bin/gitlab-runner

  # Create a GitLab Runner user
  sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

  # Install and run as a service
  sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
  sudo gitlab-runner start
  ```

4. Copy and paste the steps number 1 or run below command with the valid `glrt-...` token

  ```bash
  sudo gitlab-runner register \
    --non-interactive \
    --url "https://gitlab.com/" \
    --token "<glrt-...-paste-your-token-here>" \
    --executor "shell" \
    --description "codespace-shell-runner-kkm"
  ```

5. Copy paste and Run step 3 to run the gitlab-runer

```
gitlab-runner run
```

You will see the gitlab runner waiting for job to run on the terminal, you can validate from the Gitlab UI by clicking the View Runner

//TODO to validate and clean
### Give the runner access to Docker and the cluster

Pipeline jobs will run `docker build` and `kubectl apply` as the `gitlab-runner` Linux user — it needs the same Docker group membership and kubeconfig your own `vscode` user already has.

```bash
# Let gitlab-runner talk to the Docker socket
sudo usermod -aG docker gitlab-runner

# Share the kubeconfig so gitlab-runner can reach the k3d-enablement cluster
sudo mkdir -p /home/gitlab-runner/.kube
sudo cp ~/.kube/config /home/gitlab-runner/.kube/config
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab-runner/.kube
```

### Install and start the runner service

```bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

!!! tip "Group membership needs a restart"
    `usermod -aG docker` only takes effect for **new** processes. Since you ran it before `gitlab-runner start`, the service picks it up immediately. If you ever add the group *after* the service is already running, restart it with `sudo gitlab-runner restart`.

---

## 4. Build and test stages

The repo already ships a `.gitlab-ci.yaml`. Replace it with this working version (fixes the placeholder stages and removes the `image:` keys, which the **shell** executor ignores anyway — jobs run directly on the Codespace host where Node.js is already installed):

```yaml title=".gitlab-ci.yaml" linenums="1"
stages:
  - build
  - test

build-job:
  stage: build
  tags:
    - shell
  script:
    - npm ci
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour

test-job:
  stage: test
  tags:
    - shell
  needs:
    - build-job
  script:
    - npm test
```

```bash
git add .gitlab-ci.yaml
git commit -m "ci: build and test stages"
git push
```

Watch it go green under **CI/CD → Pipelines**.

---

## 5. Add SAST

GitLab ships a ready-made SAST template that auto-selects analyzers based on your project's languages.

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
include:
  - template: Jobs/SAST.gitlab-ci.yml

variables:
  # The bundled SAST analyzers ship as Docker images. Our runner uses the
  # shell executor (no container isolation) so it can't run them here —
  # in a production setup you'd point a docker/kubernetes-executor runner
  # at this pipeline instead. We keep the include for visibility but
  # disable actual scanning to avoid stuck/failing jobs on this runner.
  SAST_DISABLED: "true"
```

Push it and look at the **Test** stage in the pipeline graph — you'll see the `semgrep-sast` job appear and pass instantly as a no-op, thanks to `SAST_DISABLED`.

!!! info "Why not just use the docker executor?"
    You could — the Codespace's Docker socket is available — but then every job in the pipeline needs its own container image with Node, kubectl, docker, and helm baked in, and `docker build`/`kubectl` need extra wiring to reach the host's daemon and cluster from inside a container. The shell executor keeps this workshop simple by running jobs directly on the Codespace, which already has everything installed.

---

## 6. Install SonarQube (Code Quality Gate)

SonarQube Community Edition installs into the Kubernetes cluster with a single command using a helper already loaded in your shell.

!!! info "Add New Terminal and leave the GitLab Runner Terminal open"
    Add New Terminal by clicking the `+ New Terminal` on the codespace.

```bash
installSonarqube
```

This will:

1. Add the SonarQube Helm repo and update it
2. Create the `sonarqube` namespace
3. Deploy SonarQube Community Edition via Helm
4. Wait for all pods to be ready
5. Start a port-forward on **port 9000** so the UI is reachable

### Open SonarQube

Port `9000` is already pre-declared in this Codespace (see `devcontainer.json`):

1. Open the **Ports** panel in VS Code
2. Find port `9000` (labeled `SonarQube`)
3. Make the port into **Public**, to make the sonar accessible from gitlab
4. Click the globe icon next to it to **Open in Browser**

Log in with **admin / admin** and set a new password when prompted.


### Configure GitLab and Sonar Token

1. In Sonar Home Page (Projects) Create your pavourite project from DevOps Platform, Click **Setup (Import from Gitlab) → Create Configuration**
2. Under **Generate Tokens**, name it `kkm-pulse-demo`, type **Gitlab API URL**, paste the **Gitlab Personal Access Token**, and **Save Configuration**
3. Paste the **Gitlab Persona Access token** again to provide access to the repo
4. On the Gitlab Project onboarding, select `kkm-pulse-demo → Follow the instance Default  → Analyze with Gitlab CI`
2. Generate the token on step 1 **Generate Token**, name it `kkm-pulse-demo`, Select **Global Analysis Token**, click **Generate**
3. Copy the token — it's shown only once


### Configure GitLab CI/CD variables

In the `kkm-pulse-demo` project: **Settings → CI/CD → Variables → Add variable**

| Key | Value | Mask? |
|---|---|---|
| `SONAR_HOST_URL` | `http://localhost:9000` (the runner and SonarQube share the same Codespace host) | No |
| `SONAR_TOKEN` | the token you generated above | Yes |


### Install the SonarScanner CLI on the runner host

The scanner is a small Java CLI. Install it once, directly on the Codespace, so every pipeline run reuses it instead of re-downloading it:

```bash
sudo apt-get update
sudo apt-get install -y openjdk-21-jre-headless unzip || sudo apt-get install -y default-jre-headless unzip

curl -sSLo /tmp/sonar-scanner.zip \
  https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-8.1.0.6389-linux-x64.zip
sudo unzip -q -o /tmp/sonar-scanner.zip -d /opt
sudo ln -sf /opt/sonar-scanner-8.1.0.6389-linux-x64/bin/sonar-scanner /usr/local/bin/sonar-scanner

sonar-scanner -v
```

### Add the SonarQube stage

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality

sonarqube-check:
  stage: code_quality
  tags:
    - shell
  variables:
    GIT_DEPTH: "0"  # SonarQube wants full git history for blame/new-code analysis
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
  cache:
    key: "sonar-cache-${CI_COMMIT_REF_SLUG}"
    paths:
      - .sonar/cache
  script:
    - sonar-scanner
      -Dsonar.projectKey=kkm-pulse-demo
      -Dsonar.sources=.
      -Dsonar.host.url="${SONAR_HOST_URL}"
      -Dsonar.token="${SONAR_TOKEN}"
  allow_failure: true
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    - if: $CI_COMMIT_BRANCH == 'main'
```

```bash
git add .gitlab-ci.yaml
git commit -m "ci: add sonarqube quality gate"
git push
```

### Validate

1. Watch `sonarqube-check` run in **CI/CD → Pipelines**
2. Back in the SonarQube UI, open **Projects → kkm-pulse-demo** and confirm the analysis landed with a Quality Gate result

---

## Knowledge Check

### 1. Why does `allow_failure: true` exist — and when should you remove it?

The `sonarqube-check` job has `allow_failure: true`, which means the pipeline always shows green even when SonarQube finds problems.

??? question "Show Answer"

    **Why it's there by default:** When you first wire up SonarQube, the existing codebase almost always has findings (code smells, duplication, low coverage). Setting `allow_failure: true` lets the team see the results without immediately blocking every pipeline and merge request. You get visibility first, then tighten the gate once the baseline is clean.

    **When to remove it:** Once your Quality Gate reflects agreed-upon standards and the existing issues are resolved (or marked "Won't Fix"), flip the flag off so a failing gate actually stops the pipeline.

    **How to enforce it on MRs but stay informational on `main`:**

    ```yaml
    sonarqube-check:
      stage: code_quality
      tags:
        - shell
      rules:
        - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
          allow_failure: false   # <-- blocks the MR if quality gate fails
        - if: $CI_COMMIT_BRANCH == 'main'
          allow_failure: true    # <-- informational only on main
    ```

    This pattern is the recommended progression: observe → baseline → enforce on new code (MRs) → eventually enforce on `main` too.

---

### 2. Hands-on: Introduce a bug and watch SonarQube catch it

In `server.js`, add the following deliberately insecure snippet just before the `module.exports` line — this simulates a developer accidentally hardcoding a secret:

```javascript
// TODO: move this to env vars later
const DB_PASSWORD = "super_secret_123";
```

Push the change, let the pipeline run, then check the SonarQube UI.

??? question "Show Answer: What SonarQube flags and how to fix it"

    **What you'll see in SonarQube:**

    SonarQube's built-in rules flag hardcoded credentials as a **Security Hotspot** (rule `javascript:S2068` — *"Hard-coded credentials are security-sensitive"*). The finding will appear under **Projects → kkm-pulse-demo → Security Hotspots**.

    **How to resolve it properly:**

    1. Delete the hardcoded line from `server.js`.
    2. Read the value from an environment variable instead:

    ```javascript
    const DB_PASSWORD = process.env.DB_PASSWORD;
    ```

    3. In GitLab, add `DB_PASSWORD` as a **masked CI/CD variable** (**Settings → CI/CD → Variables**) so the runner injects it at runtime without it ever appearing in source code or logs.

    4. Push the fix. The next pipeline scan will resolve the hotspot automatically and your Quality Gate should return to green.

    **Key takeaway:** SonarQube doesn't just count bugs — it surfaces security-sensitive patterns that code reviewers commonly miss. Pair it with a strict Quality Gate (no new hotspots unreviewed) to catch these before they reach `main`.

---

<div class="grid cards" markdown>
- [Continue to Use Case 3 — Docker Build & Deploy to K8s :octicons-arrow-right-24:](usecase3-deployk8s.md)
</div>
