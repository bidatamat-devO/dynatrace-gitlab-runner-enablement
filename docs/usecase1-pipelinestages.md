--8<-- "snippets/dt-enablement.md"

# Use Case 1 — Pipeline Stages, Jobs & Artifacts

In this use case you will build a small GitLab CI pipeline **step by step**, entirely in the **GitLab Web IDE**. You will see a job run, watch a test fail on purpose, learn why, and fix it with **stages** and **artifacts**.

You need a **GitLab Runner** to execute the jobs, so you will install one inside your Codespace first. It stays registered and is reused in the following use cases.

---

## 1. Create the project

!!! example "Step-by-step"
    1. Log in to [gitlab.com](https://gitlab.com) (or create a free account)
    2. Click **Create new... → New project/repository → Create blank project**
    3. Project name: `pipelinestages`, visibility: your choice (Private is fine)
    4. Leave "Initialize repository with a README" **checked**, then **Create project**

---

## 2. Deploy a GitLab Runner in the Codespace

A pipeline is only a definition — jobs run on a **runner**. Without one, every pipeline stays **pending** forever. Install the runner as a **shell executor** inside the Codespace:

```bash
# Download the binary for the codespace's architecture (amd64 shown; use arm64 on Apple Silicon)
sudo curl -L --output /usr/local/bin/gitlab-runner \
  https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a dedicated GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```

!!! example "Step-by-step — Create the runner in the GitLab UI"
    1. In the `pipelinestages` project, go to **Settings → CI/CD → Runners**
    2. Click **New project runner**
    3. Check **Run untagged jobs** — the pipeline in this use case does not use tags
    4. Click **Create runner** and copy the `--token glrt-...` value from the registration command

Register it and start it from the Codespace terminal:

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --token "<glrt-...-paste-your-token-here>" \
  --executor "shell" \
  --description "codespace-shell-runner"

gitlab-runner run
```

Keep this terminal open. **Settings → CI/CD → Runners** should show the runner as **online** (green dot).

!!! info "Shell executor and `image:`"
    The jobs below declare `image: alpine`. A shell executor runs commands directly on the Codespace and ignores `image:`, so the pipeline works unchanged. (A Docker executor would pull `alpine` and run each job in a container.)

!!! tip "Want to know more about runners and tags?"
    [Use Case 2](usecase2-gitlabrunner.md) covers runner registration and tag routing in detail.

---

## 3. Step 1 — A single job

!!! example "Step-by-step — in the Web IDE"
    1. In the project, click **Edit → Web IDE** (or press `.` on the repository page)
    2. Create a new file named `.gitlab-ci.yml`
    3. Paste the pipeline below
    4. Open the **Source Control** panel, enter a commit message such as `feat: add build_car job`, and **Commit to main**
    5. Go to **Build → Pipelines** and open the pipeline

```yaml
build_car:
    image: alpine
    stage: build
    script:
        - echo "Hello, $USER!"
        - echo "build the car pipeline"
        - mkdir build
        - touch build/car.txt
        - echo "chassis" > build/car.txt
        - cat build/car.txt
        - echo "Congratulation!!!"
```

The job should turn **green**. Open it and read the log — the `cat` output shows `chassis`.

!!! note "No `stages:` yet"
    GitLab provides default stages (`.pre`, `build`, `test`, `deploy`, `.post`), so `stage: build` is valid even though you haven't declared anything. You will declare them explicitly in Step 3.

---

## 4. Step 2 — Add a test job (expected to fail)

Edit `.gitlab-ci.yml` in the Web IDE and append the `test_car` job:

```yaml
build_car:
    image: alpine
    stage: build
    script:
        - echo "Hello, $USER!"
        - echo "build the car pipeline"
        - mkdir build
        - touch build/car.txt
        - echo "chassis" > build/car.txt
        - cat build/car.txt
        - echo "Congratulation!!!"

test_car:
    image: alpine
    stage: test
    script:
        - echo "oppss!!! the pipeline break"
        - test -f build/car.txt
        - grep "chassis" build/car.txt
```

Commit (`test: add test_car job`) and open the pipeline. `build_car` passes, but **`test_car` fails** on `test -f build/car.txt`.

!!! question "Why does it fail?"
    Every job starts in a **fresh, clean workspace**. The `build/car.txt` file created by `build_car` is thrown away when that job ends, so `test_car` cannot see it. Jobs share files only when you explicitly pass them as **artifacts**.

---

## 5. Step 3 — Declare the stages

Add the `stages:` block at the very top of the file:

```yaml
stages:
    - build
    - test

build_car:
    ...
```

Commit (`ci: declare stages`) and run again. The pipeline view now shows two columns, **build → test**, and the result is still a failure.

!!! tip "What stages do"
    Jobs in the same stage run in **parallel**; stages run **in order**, and a failed stage stops the pipeline before the next one starts. Declaring `stages:` makes that order explicit and lets you add your own stages (for example `deploy`) later.

---

## 6. Step 4 — Add artifacts and get a green pipeline

Add the `artifacts:` block to `build_car` so the `build/` folder is handed to later jobs:

```yaml
stages:
    - build
    - test

build_car:
    image: alpine
    stage: build
    script:
        - echo "Hello, $USER!"
        - echo "build the car pipeline"
        - mkdir build
        - touch build/car.txt
        - echo "chassis" > build/car.txt
        - cat build/car.txt
        - echo "Congratulation!!!"
    artifacts:
        paths:
            - build/

test_car:
    image: alpine
    stage: test
    script:
        - test -f build/car.txt
        - grep "chassis" build/car.txt
        - echo "successfull pipeline build!!"
```

Commit (`fix: pass build output as artifact`). This time both jobs turn **green**. On the `build_car` job page you can **Browse** or **Download** the `build/` artifact.

!!! tip "Key takeaways"
    - A **job** runs a script on a runner; a **stage** groups jobs and defines their order
    - Each job gets a clean workspace — nothing carries over automatically
    - **Artifacts** are how a job hands files to the jobs in later stages
    - A runner must be online, or nothing runs at all

---

<div class="grid cards" markdown>
- [Continue to Use Case 2 — First GitLab Project & Runner :octicons-arrow-right-24:](usecase2-gitlabrunner.md)
</div>
