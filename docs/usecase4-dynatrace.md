--8<-- "snippets/dt-enablement.md"

# Use Case 4 — Dynatrace Events & Load Testing

`kkm-pulse-demo` now deploys itself to Kubernetes on every pipeline run. Let's make Dynatrace aware of it: install the OneAgent, mark every deployment with a **Dynatrace event**, run a small **load test** from the pipeline, and push the test result back into Dynatrace as another event.

---

## 1. Deploy the Dynatrace Operator and OneAgent

These helper functions are already loaded in your Codespace shell (they read `DT_ENVIRONMENT`, `DT_OPERATOR_TOKEN`, and `DT_INGEST_TOKEN` from the Codespace secrets you set at launch):

```bash
dynatraceDeployOperator
deployApplicationMonitoring
```

!!! info "Why Application Monitoring, not Cloud Native Full Stack?"
    Cloud Native Full Stack needs privileged host-level access that isn't available on the `k3d`-based `enablement` cluster this workshop uses. Application Monitoring injects the OneAgent into your app's container at Pod startup instead — no host access required.

OneAgent injection only happens **at Pod creation**, so restart the Deployment you already have running so the next Pod picks it up:

```bash
kubectl rollout restart deployment/kkm-pulse-demo -n kkm-pulse-dev
kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-dev
```

Give it a minute, then generate some traffic and check Dynatrace:

```bash
curl -H "Host: kkm-pulse-dev.127.0.0.1.sslip.io" http://localhost/api/status
```

In Dynatrace, go to **Applications & Microservices → Processes** (or search `kkm-pulse-demo`) — you should see a new Node.js process group appear within a couple of minutes.

![Dynatrace Applications & Microservices → Processes showing kkm-pulse-demo Node.js process group](img/usecase4-dt-process-group.png)

---

## 2. Give the pipeline Dynatrace credentials

In the `kkm-pulse-demo` project: **Settings → CI/CD → Variables → Add variable**

| Key | Value | Mask? |
|---|---|---|
| `DT_ENVIRONMENT` | same value as your Codespace secret, e.g. `https://abc123.apps.dynatrace.com` | No |
| `DT_INGEST_TOKEN` | same value as your Codespace secret | Yes |

![Adding DT_ENVIRONMENT and DT_INGEST_TOKEN as CI/CD variables in GitLab Settings → CI/CD → Variables](img/usecase4-gitlab-dt-vars.png)

!!! tip "Where do I find the values again?"
    They were provided as Codespaces secrets at launch. From the terminal: `echo $DT_ENVIRONMENT` (don't echo the token to a shared screen — copy it from wherever you originally stored it, or from GitHub's Codespaces secrets settings).

---

## 3. Send a deployment event

The Environment API v2 [events/ingest](https://docs.dynatrace.com/docs/shortlink/api-events-v2-post-event) endpoint takes a `CUSTOM_DEPLOYMENT` event. `DT_ENVIRONMENT` is the `.apps.` SSO URL; the ingest API lives on the `.live.` host, so we rewrite it inline.

The richer the event payload, the more context Davis AI has when it correlates a problem to a deployment. The job below populates the standard Dynatrace deployment fields **plus** a set of traceability properties drawn straight from GitLab CI predefined variables — no extra scripting needed.

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
notify-dynatrace-deploy:
  stage: deploy_dev
  tags:
    - shell
  needs:
    - deploy-dev
  script:
    - DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')
    - |
      PAYLOAD=$(cat <<EOF
      {
        "eventType": "CUSTOM_DEPLOYMENT",
        "title": "kkm-pulse-demo deployed to dev",
        "entitySelector": "type(SERVICE),tag(k8s.namespace.name:kkm-pulse-dev)",
        "properties": {
          "dt.event.deployment.name":    "kkm-pulse-demo",
          "deploymentVersion":           "${CI_COMMIT_SHORT_SHA}",
          "source":                      "GitLab CI",
          "ciBackLink":                  "${CI_JOB_URL}",
          "GitLabUrl":                   "${CI_PROJECT_URL}",
          "GitCommit":                   "${CI_COMMIT_SHA}",
          "Owner":                       "platform-team",
          "Approval":                    "${GITLAB_USER_NAME}",
          "environment":                 "dev",
          "branch":                      "${CI_COMMIT_REF_NAME}",
          "commitAuthor":                "${CI_COMMIT_AUTHOR}",
          "pipelineUrl":                 "${CI_PIPELINE_URL}",
          "projectName":                 "${CI_PROJECT_NAME}",
          "runnerTags":                  "${CI_RUNNER_TAGS}"
        }
      }
      EOF
      )
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest" \
        -H "Authorization: Api-Token ${DT_INGEST_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "$PAYLOAD"
```

### What each property does

| Property | GitLab variable | Purpose |
|---|---|---|
| `deploymentVersion` | `CI_COMMIT_SHORT_SHA` | Identifies the exact build artifact deployed; shows up on the Dynatrace deployment marker tooltip |
| `source` | _(literal)_ | Labels the event origin in the events feed — useful when events arrive from multiple tools |
| `ciBackLink` | `CI_JOB_URL` | Clickable deep-link from the Dynatrace event directly to the GitLab job log |
| `GitLabUrl` | `CI_PROJECT_URL` | Project home — lets SREs navigate to the repo from a Dynatrace problem card |
| `GitCommit` | `CI_COMMIT_SHA` | Full 40-character SHA — unambiguous reference for `git bisect` or release audits |
| `Owner` | _(literal)_ | Team responsible for this service; drives alert routing and runbook assignment |
| `Approval` | `GITLAB_USER_NAME` | GitLab username of the person who triggered the pipeline — the human "approved" this push |
| `branch` | `CI_COMMIT_REF_NAME` | Branch that was deployed; helps distinguish feature-branch deploys from `main` |
| `commitAuthor` | `CI_COMMIT_AUTHOR` | Name + email of the commit author (may differ from the pipeline triggerer) |
| `pipelineUrl` | `CI_PIPELINE_URL` | Link to the full pipeline run — broader context than a single job URL |
| `projectName` | `CI_PROJECT_NAME` | Repository name — useful in multi-project Dynatrace dashboards |
| `runnerTags` | `CI_RUNNER_TAGS` | Which runner pool executed the job — handy for infra-level problem correlation |

!!! tip "Adding a variable for `Owner`"
    Hard-coding `platform-team` works for a workshop, but in production you'd store it as a GitLab CI/CD variable (`DT_OWNER`) so different projects can declare different owners without touching the pipeline template.

Push, run the pipeline, then in Dynatrace open **Notifications & alerting → Events** (or search `deployment.name:kkm-pulse-demo` in the events feed) to see it land — deployment events also draw a marker line on the process's timeline charts. Click the marker to expand the full property list you just sent.

![Dynatrace Events feed showing CUSTOM_DEPLOYMENT event for kkm-pulse-demo with deployment properties](img/usecase4-dt-deployment-event.png)

---

## 4. Run a load test and grade it

No JMeter needed for a small Express API — a short bash/curl loop is enough to produce a real latency/error-rate signal.

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev
  - load_test

load-test:
  stage: load_test
  tags:
    - shell
  needs:
    - notify-dynatrace-deploy
  script:
    - |
      TARGET="http://localhost/api/status"
      HOST_HEADER="kkm-pulse-dev.127.0.0.1.sslip.io"
      TOTAL=50
      FAIL=0
      SUM_MS=0
      for i in $(seq 1 $TOTAL); do
        RESULT=$(curl -s -o /dev/null -H "Host: ${HOST_HEADER}" -w "%{http_code} %{time_total}" "$TARGET")
        CODE=$(echo "$RESULT" | awk '{print $1}')
        SECS=$(echo "$RESULT" | awk '{print $2}')
        MS=$(awk -v s="$SECS" 'BEGIN{printf "%d", s*1000}')
        SUM_MS=$((SUM_MS + MS))
        [ "$CODE" != "200" ] && FAIL=$((FAIL + 1))
      done
      AVG_MS=$((SUM_MS / TOTAL))
      ERROR_RATE=$((FAIL * 100 / TOTAL))
      echo "Requests: $TOTAL  Failures: $FAIL  Error rate: ${ERROR_RATE}%  Avg latency: ${AVG_MS}ms"
      {
        echo "LOADTEST_TOTAL=$TOTAL"
        echo "LOADTEST_FAIL=$FAIL"
        echo "LOADTEST_ERROR_RATE=$ERROR_RATE"
        echo "LOADTEST_AVG_MS=$AVG_MS"
      } >> loadtest.env
  artifacts:
    reports:
      dotenv: loadtest.env

notify-dynatrace-test-result:
  stage: load_test
  tags:
    - shell
  needs:
    - load-test
  script:
    - DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')
    - >
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest"
      -H "Authorization: Api-Token ${DT_INGEST_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"eventType\":\"CUSTOM_INFO\",\"title\":\"kkm-pulse-demo load test result\",\"properties\":{\"requests\":\"${LOADTEST_TOTAL}\",\"failures\":\"${LOADTEST_FAIL}\",\"error_rate_pct\":\"${LOADTEST_ERROR_RATE}\",\"avg_latency_ms\":\"${LOADTEST_AVG_MS}\",\"commit\":\"${CI_COMMIT_SHORT_SHA}\"}}"
    - |
      if [ "$LOADTEST_ERROR_RATE" -gt 10 ]; then
        echo "Error rate ${LOADTEST_ERROR_RATE}% exceeds the 10% budget — failing this job."
        exit 1
      fi
```

`load-test` records its numbers via a [`dotenv` artifact](https://docs.gitlab.com/ee/ci/yaml/artifacts_reports.html#artifactsreportsdotenv), so `notify-dynatrace-test-result` can read `$LOADTEST_ERROR_RATE` etc. straight from the environment — no need to re-run the test or parse logs.

```bash
git add .gitlab-ci.yaml
git commit -m "ci: dynatrace deployment events and load test"
git push
```

---

## 5. See Davis AI catch a real anomaly

Trigger the CPU-spike endpoint a few times while watching Dynatrace:

```bash
for i in 1 2 3; do curl -H "Host: kkm-pulse-dev.127.0.0.1.sslip.io" http://localhost/api/trigger-anomaly; done
```

Open **Problems** in Dynatrace — Davis AI should surface a CPU saturation problem correlated to the `kkm-pulse-demo` process within a few minutes, alongside the deployment and load-test events you just pushed on the same timeline.

![Dynatrace Problems showing Davis AI-detected CPU saturation anomaly correlated to kkm-pulse-demo](img/usecase4-dt-problem.png)

---

## What "stop a bad build" means here

`notify-dynatrace-test-result` exits non-zero when the error budget is blown. Since it's a downstream `needs` dependency, any stage you add **after** `load_test` (like a production deploy) simply won't start if this job fails — the bad build never leaves dev. Use Case 5 builds exactly that gate.

---

## Knowledge Check

### Question 1 — Why does the pipeline rewrite `.apps.` to `.live.`?

The `DT_TENANT` line transforms the `DT_ENVIRONMENT` URL before calling the Events API:

```bash
DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')
```

Explain why this transformation is necessary and what would happen if you passed `$DT_ENVIRONMENT` directly to the `curl` command.

??? question "Show Answer"

    Dynatrace uses **two separate hostnames** for the same tenant:

    | Hostname pattern | Purpose |
    |---|---|
    | `*.apps.dynatrace.com` | The SSO-authenticated web UI (OAuth-based, browser sessions) |
    | `*.live.dynatrace.com` | The classic REST API endpoints (token-based, programmatic access) |

    The Environment API v2 (`/api/v2/events/ingest`) is served on the `.live.` host. If you send the `curl` request to the `.apps.` URL, the request is routed to the UI's authentication layer, which does not accept `Api-Token` auth headers — you'll receive a `401 Unauthorized` or a redirect to the login page.

    **Why store `.apps.` and transform at runtime?**

    The `.apps.` URL is what the Dynatrace UI displays in the address bar and what most users copy as "my environment URL". Storing it as `DT_ENVIRONMENT` and deriving the API URL in-pipeline means you only need one variable — you never have to ask someone to manually find and paste the `.live.` variant.

    **Rule of thumb:** any Dynatrace API call using `Api-Token` authentication goes to `*.live.dynatrace.com`; anything browser-SSO-based goes to `*.apps.dynatrace.com`.

---

### Question 2 — Hands-on: Tighten the error budget gate

The current `notify-dynatrace-test-result` job only fails the pipeline when the error rate exceeds 10%. Your team has agreed on stricter SLOs:

- Error rate **≤ 5%**
- Average latency **≤ 300 ms**

Modify the gate section of `notify-dynatrace-test-result` so that the job fails if **either** condition is violated, and prints a clear message identifying which threshold was breached.

??? question "Show Answer"

    Replace the existing gate block at the bottom of `notify-dynatrace-test-result`'s `script:`:

    ```yaml
    - |
      FAILED=0
      if [ "$LOADTEST_ERROR_RATE" -gt 5 ]; then
        echo "ERROR: Error rate ${LOADTEST_ERROR_RATE}% exceeds the 5% budget."
        FAILED=1
      fi
      if [ "$LOADTEST_AVG_MS" -gt 300 ]; then
        echo "ERROR: Average latency ${LOADTEST_AVG_MS}ms exceeds the 300ms threshold."
        FAILED=1
      fi
      if [ "$FAILED" -eq 1 ]; then
        echo "One or more SLO thresholds were violated — failing this job."
        exit 1
      fi
      echo "All SLOs met — error rate ${LOADTEST_ERROR_RATE}%, avg latency ${LOADTEST_AVG_MS}ms."
    ```

    **Why use a `FAILED` flag instead of two separate `exit 1` calls?**

    A single `exit 1` inside the first `if` block would short-circuit the script and skip the latency check entirely. Using `FAILED=1` allows both conditions to be evaluated and reported before the job exits, so the pipeline log shows exactly which SLOs were violated — not just the first one hit.

    **What to update in Dynatrace:**

    Also push the tightened thresholds into the `CUSTOM_INFO` event payload so the Dynatrace events feed reflects your actual SLO values:

    ```bash
    "slo_error_rate_budget\":\"5\",\"slo_latency_ms\":\"300\""
    ```

    This keeps the pipeline gate and the Dynatrace record in sync.

<div class="grid cards" markdown>
- [Continue to Use Case 5 — Dev/Prod Gates :octicons-arrow-right-24:](usecase5-devprodstages.md)
</div>
