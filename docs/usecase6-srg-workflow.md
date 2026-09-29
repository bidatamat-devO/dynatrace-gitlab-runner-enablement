--8<-- "snippets/dt-enablement.md"

# Use Case 6 — Site Reliability Guardian & Automated Rollback

The pipeline now deploys to production with a human approval gate and a load-test quality check. But what happens *after* the deployment completes? Even when pre-deployment gates pass, real production traffic can reveal issues — elevated latency, higher error rates, or newly-detected security vulnerabilities — that only emerge under genuine load.

**Site Reliability Guardian (SRG)** is Dynatrace's automated post-deployment validation engine. It evaluates a deployment against a set of objectives anchored to real observability data — including security posture — and returns a binary **PASS / FAIL** verdict your pipeline can act on. And if production problems surface *after* the pipeline has already finished, a **Dynatrace Workflow** can call the GitLab API to trigger a rollback automatically.

In this use case you will:

1. Create a Guardian in Dynatrace and define objectives — error rate, response time, and critical security vulnerabilities
2. Trigger an SRG evaluation from the pipeline after every production deployment
3. Fail the pipeline if production does not meet its objectives
4. Build a Dynatrace Workflow that calls back the GitLab pipeline to trigger a rollback if Dynatrace detects a problem in production after the pipeline has finished

---

## 1. Create a Site Reliability Guardian

In Dynatrace, navigate to **Apps → Site Reliability Guardian** (search for it in the app launcher if it isn't pinned).

Click **+ New Guardian**, then choose **Choose Template** and select **Four Golden Signals**.

On the **Getting started with template** popup, click **Run Query**, select **kkm-pulse-demo**, and click **Apply Template**.

### Define objectives

Click **Add More objective** for each of the three below.

#### Objective 4 — P95 Response Time

| Field | Value |
|---|---|
| **Name** | `Average CPU usage` |
| **DQL** | `timeseries val = avg(dt.kubernetes.container.cpu_usage, default: 0), filter: in(dt.smartscape.k8s_namespace, { toSmartscapeId("<<NAMESPACE_ID_TO_REPLACE>>>") })
| fields avg = arrayAvg(val)` |
| **Fails if result** | `> 50` |
| **Warning if result** | `> 40` |

#### Objective 5 — Critical Security Vulnerabilities

| Field | Value |
|---|---|
| **Name** | `Critical Vulnerabilities` |
| **DQL** | `fetch security.events
| filter event.provider=="Dynatrace"
| filter event.kind=="SECURITY_EVENT"
| filter event.type=="VULNERABILITY_STATE_REPORT_EVENT"
| filter event.level=="ENTITY"
| fieldsAdd matcher="match"
| lookup [
fetch security.events
| filter event.provider=="Dynatrace"
| filter event.kind=="SECURITY_EVENT"
| filter event.type=="VULNERABILITY_STATE_REPORT_EVENT"
| filter event.level=="ENTITY"
| fields maxTimestamp=timestamp, matcher="match"
| limit 1
], sourceField:matcher, lookupField:matcher, fields:{maxTimestamp}
| filter timestamp==maxTimestamp
| filter event.status=="OPEN"
| filter in(vulnerability.risk.level,{"CRITICAL","HIGH"})
| filter in(affected_entity.id, {"PROCESS_GROUP-A085A3959D385BE8"})
| summarize Filtered_high-profile_vulnerabilities=arraySize(collectDistinct(vulnerability.id))` |
| **Fails criterion** | `> 0` |

!!! info "Application Security required"
    The security objective requires **Dynatrace Application Security** to be enabled. If it's unavailable on your tenant, skip this objective — the error rate and latency objectives are sufficient for the workshop. The principle (SRG can gate on security KPIs the same way it gates on performance KPIs) is the key takeaway.

Save the Guardian. Note the **Guardian ID** from the URL — it looks like `guardian-XXXXXXXXXXXXXXXX`.

---

## 2. Create a Platform API token

SRG's evaluation API uses a **Platform token** (not the classic ingest token from Use Case 4). In Dynatrace:

**Settings → Access tokens → Generate new token**

| Field | Value |
|---|---|
| **Name** | `kkm-pulse-demo SRG pipeline` |
| **Scopes** | `Davis data: Read` · `Site Reliability Guardian: Read evaluations` · `Site Reliability Guardian: Write evaluations` |

In the `kkm-pulse-demo` GitLab project, add two CI/CD variables:

| Key | Value | Mask? |
|---|---|---|
| `DT_PLATFORM_TOKEN` | the token you just generated | Yes |
| `SRG_GUARDIAN_ID` | your guardian ID (e.g., `guardian-XXXXXXXXXXXXXXXX`) | No |

---

## 3. Trigger SRG evaluation from the pipeline

Add an `srg_validate` stage after `deploy_prod`. The job waits two minutes for post-deployment metrics to stabilize, triggers the evaluation, polls until it completes, then fails the job — and therefore blocks any downstream work — if the result is not `PASS`.

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev
  - load_test
  - deploy_prod
  - srg_validate

srg-evaluate-prod:
  stage: srg_validate
  tags:
    - shell
  needs:
    - notify-dynatrace-prod-deploy
  script:
    - echo "Waiting 120s for post-deployment metrics to stabilize..."
    - sleep 120
    - |
      EVAL_RESPONSE=$(curl -sf -X POST "${DT_ENVIRONMENT}/api/v2/site-reliability-guardian/evaluations" \
        -H "Authorization: Api-Token ${DT_PLATFORM_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "{
          \"guardianId\": \"${SRG_GUARDIAN_ID}\",
          \"timeframeFrom\": \"now-3m\",
          \"timeframeTo\": \"now\"
        }")
      EVAL_ID=$(echo "$EVAL_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['evaluationId'])")
      echo "SRG evaluation triggered: $EVAL_ID"
      echo "SRG_EVAL_ID=$EVAL_ID" >> srg.env
    - |
      STATUS="RUNNING"
      ATTEMPTS=0
      MAX_ATTEMPTS=30
      while [ "$STATUS" = "RUNNING" ] && [ "$ATTEMPTS" -lt "$MAX_ATTEMPTS" ]; do
        sleep 10
        RESULT=$(curl -sf "${DT_ENVIRONMENT}/api/v2/site-reliability-guardian/evaluations/${EVAL_ID}" \
          -H "Authorization: Api-Token ${DT_PLATFORM_TOKEN}")
        STATUS=$(echo "$RESULT" | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
        echo "Attempt $((ATTEMPTS+1))/$MAX_ATTEMPTS — SRG status: $STATUS"
        ATTEMPTS=$((ATTEMPTS + 1))
      done
      echo "SRG_STATUS=$STATUS" >> srg.env
      if [ "$STATUS" != "PASS" ]; then
        echo "ERROR: SRG evaluation returned '$STATUS'."
        echo "One or more production objectives were not met — check Site Reliability Guardian in Dynatrace for per-objective detail."
        exit 1
      fi
      echo "SRG PASSED — all reliability and security objectives met."
  artifacts:
    reports:
      dotenv: srg.env
```

```bash
git add .gitlab-ci.yaml
git commit -m "ci: add SRG post-deployment validation gate"
git push
```

Deploy to production (approve the Use Case 5 manual gate if you haven't already), then watch `srg-evaluate-prod` poll Dynatrace every 10 seconds. In Dynatrace, open **Site Reliability Guardian → kkm-pulse-demo production** and watch the evaluation populate in real time.

---

## 4. See PASS and FAIL in action

### A passing evaluation

With a clean deployment and no anomalies running, all three objectives should be green and `srg-evaluate-prod` reports `SRG PASSED`.

### Force a failing evaluation

Spike the application while the evaluation window is open:

```bash
for i in $(seq 1 8); do
  curl -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/trigger-anomaly &
done
wait
```

Wait for the next pipeline run's `srg-evaluate-prod` job (or re-trigger manually). The P95 latency objective will breach its threshold during the anomaly window, the SRG returns `FAIL`, and the job exits non-zero. No downstream jobs run.

!!! tip "SRG vs. the load test"
    The load test in Use Case 4 measures what *your curl loop* observed — a synthetic sample under controlled conditions. SRG evaluates metrics Dynatrace collected from *real application traffic* during the evaluation window. These are complementary gates: the load test catches obvious breakage early; SRG catches subtle performance regressions that only appear under concurrent production load.

---

## 5. Build a Dynatrace Workflow for automated rollback

SRG stops the current pipeline run when validation fails. But what if production degrades *after* the pipeline has already finished — say, an hour after deployment when peak traffic arrives? A Dynatrace **Workflow** can watch for production problems and automatically call the GitLab API to trigger a rollback pipeline.

### Step 1 — Create a GitLab pipeline trigger token

In the `kkm-pulse-demo` GitLab project: **Settings → CI/CD → Pipeline triggers → Add new trigger**

Name it `Dynatrace rollback` and copy:
- The **token** (a long string)
- The **trigger URL**, which looks like `https://gitlab.com/api/v4/projects/12345678/trigger/pipeline`

Note your numeric **project ID** from the URL — you need it below.

### Step 2 — Store the token in Dynatrace Vault

In Dynatrace: **Settings → Credentials Vault → Add credential**

| Field | Value |
|---|---|
| **Name** | `GITLAB_TRIGGER_TOKEN` |
| **Type** | `Token` |
| **Token value** | the trigger token you just copied |

Vault credentials are encrypted at rest and referenced in Workflows as `{{ vault.GITLAB_TRIGGER_TOKEN }}` — they never appear in plain text in logs or Workflow definitions.

### Step 3 — Create the Workflow

In Dynatrace: **Apps → Workflows → + New Workflow**

**Trigger** — set to **Event-based**:

| Field | Value |
|---|---|
| **Event type** | `Site Reliability Guardian evaluation completed` |
| **Filter condition** | `event.status == "FAIL" AND event.guardian.name == "kkm-pulse-demo production"` |

!!! tip "Catching Davis AI problems too"
    Add a second trigger for **Davis Problem** events filtered by entity `kkm-pulse-demo`. This catches issues Davis detects independently of SRG — for example, a memory leak that only becomes visible two hours after deployment.

**Action 1 — HTTP Request** (call GitLab to trigger rollback):

| Field | Value |
|---|---|
| **Label** | `Trigger GitLab rollback pipeline` |
| **Method** | `POST` |
| **URL** | `https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/trigger/pipeline` |
| **Content-Type** | `application/x-www-form-urlencoded` |

Body (form-encoded):
```
token={{ vault.GITLAB_TRIGGER_TOKEN }}&ref=main&variables[ROLLBACK]=true&variables[ROLLBACK_REASON]=Dynatrace SRG FAIL
```

**Action 2 — Send notification** (optional): chain a Slack or email notification so on-call is alerted the moment the Workflow fires.

Save and **activate** the Workflow.

### Step 4 — Add a rollback job to the pipeline

The rollback pipeline is triggered by the `ROLLBACK=true` variable injected by the Workflow. Add a conditional job that only runs when Dynatrace fires:

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev
  - load_test
  - deploy_prod
  - srg_validate
  - rollback

rollback-prod:
  stage: rollback
  tags:
    - shell
  rules:
    - if: '$ROLLBACK == "true"'
  script:
    - echo "Rollback triggered by Dynatrace — reason: ${ROLLBACK_REASON}"
    - echo "Rolling back kkm-pulse-prod to previous ReplicaSet..."
    - kubectl rollout undo deployment/kkm-pulse-demo -n kkm-pulse-prod
    - kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-prod --timeout=120s
    - 'curl -sf -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/status'
    - echo "Rollback complete — notifying Dynatrace..."
    - DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')
    - >
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest"
      -H "Authorization: Api-Token ${DT_INGEST_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"eventType\":\"CUSTOM_DEPLOYMENT\",\"title\":\"kkm-pulse-demo ROLLBACK triggered by Dynatrace\",\"properties\":{\"dt.event.deployment.name\":\"kkm-pulse-demo\",\"environment\":\"prod\",\"reason\":\"${ROLLBACK_REASON}\",\"triggered_by\":\"Dynatrace Workflow\"}}"
```

```bash
git add .gitlab-ci.yaml
git commit -m "ci: add Dynatrace-triggered rollback job"
git push
```

Normal pipeline runs skip `rollback-prod` entirely because `$ROLLBACK` is unset. Only the Dynatrace Workflow-triggered pipeline runs it.

---

## 6. Test the end-to-end loop

1. **Deploy to production** (push a change, run through the pipeline, approve the manual gate in Use Case 5, let `srg-evaluate-prod` pass)

2. **Simulate a sustained anomaly** in production after the pipeline finishes:
    ```bash
    for i in $(seq 1 15); do
      curl -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/trigger-anomaly &
    done
    wait
    ```

3. **Watch Dynatrace → Problems** — Davis AI surfaces a CPU saturation problem for `kkm-pulse-demo`

4. **Confirm the Workflow fires** — in **Apps → Workflows → kkm-pulse-demo rollback → Executions**, you should see a run start within a minute of the problem being detected

5. **Watch GitLab** — **CI/CD → Pipelines** shows a new pipeline (branch: `main`, triggered by API) with only the `rollback-prod` job

6. **Verify the rollback**:
    ```bash
    kubectl rollout history deployment/kkm-pulse-demo -n kkm-pulse-prod
    curl -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/status
    ```

7. **See the rollback event in Dynatrace** — in the `kkm-pulse-demo` service timeline, the `CUSTOM_DEPLOYMENT` rollback event appears alongside the Davis problem — a complete audit trail with no manual steps.

This closes the **deploy → observe → act** loop entirely within Dynatrace and GitLab, with no human intervention required.

---

## What you've built

| Capability | Implementation |
|---|---|
| Automated post-deployment validation | SRG evaluates error rate, response time, and security vulnerabilities against defined objectives after every prod deployment |
| Binary PASS/FAIL pipeline gate | `srg-evaluate-prod` exits non-zero on any SRG FAIL — downstream stages are blocked |
| Security vulnerability as a quality gate | SRG objective counts critical CVEs — a deployment with unresolved critical vulnerabilities fails the gate |
| Proactive rollback for post-pipeline issues | Dynatrace Workflow fires on SRG FAIL or Davis Problem → calls GitLab trigger API → `rollback-prod` job runs |
| Secure credential handling | GitLab trigger token stored in Dynatrace Vault, never exposed in logs or Workflow YAML |
| Full observability of the rollback itself | `rollback-prod` sends a `CUSTOM_DEPLOYMENT` event back to Dynatrace — the rollback is visible on the service timeline |

---

## Knowledge Check

### Question 1 — Why does `srg-evaluate-prod` wait 120 seconds before triggering?

The job begins with `sleep 120` before calling the SRG API. Explain what category of incorrect results becomes more likely if you remove this wait and trigger the evaluation immediately after deployment.

??? question "Show Answer"

    SRG evaluates metrics that Dynatrace collected **during the evaluation time window** (`timeframeFrom` to `timeframeTo`). Immediately after a deployment:

    - The newly deployed Pod may still be in its startup phase — the first few requests are handled during JIT compilation, connection-pool warmup, and DNS resolution, producing artificially high latency and a spike in error rate.
    - Dynatrace's metrics pipeline ingests and aggregates data with a small delay; some data points from the first seconds of traffic may not have arrived in the platform yet when the evaluation window closes.

    **Without the wait, you risk a false FAIL:**

    The evaluation window captures the startup noise — elevated latency and possibly some 5xx responses during Pod initialization — and compares it against steady-state thresholds. A healthy deployment can fail its SRG objectives simply because the evaluation ran too early.

    **Why not wait even longer?**

    A longer wait delays feedback and keeps the pipeline blocked. 120 seconds is a pragmatic balance: enough time for the application to reach steady state and for metrics to propagate through the Dynatrace ingest pipeline, but short enough that the feedback loop remains useful. In production, tune this to match your application's actual warm-up profile — some services need 30 seconds, others need 5 minutes.

    **Rule of thumb:** set the wait to *at least* the time it takes for your application's error rate to stabilize after a cold start, plus 30–60 seconds for metric ingestion latency.

---

### Question 2 — Hands-on: Surface the rollback reason in Dynatrace and GitLab logs

The `rollback-prod` job already logs `${ROLLBACK_REASON}` to the console. Extend the end-to-end loop so that:

1. The Dynatrace Workflow passes the SRG evaluation ID as a third pipeline variable (`ROLLBACK_EVAL_ID`) alongside `ROLLBACK` and `ROLLBACK_REASON`
2. `rollback-prod` includes `$ROLLBACK_EVAL_ID` in the `CUSTOM_DEPLOYMENT` event it sends back to Dynatrace so the rollback event is directly linked to the evaluation that triggered it

Show the changes needed in both the Workflow body and the pipeline job.

??? question "Show Answer"

    **Step 1 — Add `ROLLBACK_EVAL_ID` to the Workflow HTTP action body:**

    In Dynatrace Workflows, the SRG evaluation event exposes the evaluation ID as `{{ event()['evaluationId'] }}`. Update the form-encoded body:

    ```
    token={{ vault.GITLAB_TRIGGER_TOKEN }}&ref=main&variables[ROLLBACK]=true&variables[ROLLBACK_REASON]=Dynatrace SRG FAIL&variables[ROLLBACK_EVAL_ID]={{ event()['evaluationId'] }}
    ```

    **Step 2 — Include `ROLLBACK_EVAL_ID` in the `CUSTOM_DEPLOYMENT` event sent from `rollback-prod`:**

    Update the `curl` call at the end of the rollback job's `script:`:

    ```yaml
    - >
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest"
      -H "Authorization: Api-Token ${DT_INGEST_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"eventType\":\"CUSTOM_DEPLOYMENT\",\"title\":\"kkm-pulse-demo ROLLBACK triggered by Dynatrace\",\"properties\":{\"dt.event.deployment.name\":\"kkm-pulse-demo\",\"environment\":\"prod\",\"reason\":\"${ROLLBACK_REASON}\",\"triggered_by\":\"Dynatrace Workflow\",\"srg_evaluation_id\":\"${ROLLBACK_EVAL_ID}\"}}"
    ```

    **What this gives you:**

    The `CUSTOM_DEPLOYMENT` rollback event now carries the `srg_evaluation_id` property. In Dynatrace's event feed you can click from the rollback event directly to the SRG evaluation detail — you'll see exactly which objectives failed, what the measured values were, and when the breach occurred. This closes the audit trail: deployment → SRG FAIL → rollback, all cross-linked without any manual correlation.

    **Verify in the GitLab log:**

    Add a log line to `rollback-prod` before the curl so the pipeline job output also records the link:

    ```yaml
    - echo "Rolling back due to SRG evaluation ${ROLLBACK_EVAL_ID}: ${ROLLBACK_REASON}"
    ```

---

## Recap — all six use cases together

Across six use cases you took `kkm-pulse-demo` from zero to a fully observable, self-healing pipeline:

1. **Use Case 1** — GitLab project and a self-hosted runner registered over SSH
2. **Use Case 2** — Build, test, SAST, and SonarQube quality gate on every push
3. **Use Case 3** — Docker image built in CI, loaded into k3d, deployed and exposed on Kubernetes
4. **Use Case 4** — Dynatrace OneAgent, deployment events, load test graded against an error budget
5. **Use Case 5** — Separate dev/prod environments with a structural gate: a bad build can never reach the ▶️ button
6. **Use Case 6** — SRG validates production against KPIs including security; Dynatrace Workflow triggers an automatic rollback when production degrades after the pipeline finishes

<div class="grid cards" markdown>
- [Continue to Cleanup :octicons-arrow-right-24:](cleanup.md)
</div>
