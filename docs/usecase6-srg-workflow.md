--8<-- "snippets/dt-enablement.md"

# Use Case 6 — Site Reliability Guardian & Automated Rollback

The pipeline now deploys to production with a human approval gate and a load-test quality check. But what happens *after* the deployment completes? Even when pre-deployment gates pass, real production traffic can reveal issues — elevated latency, higher error rates, or newly-detected security vulnerabilities — that only emerge under genuine load.

**Site Reliability Guardian (SRG)** is Dynatrace's automated post-deployment validation engine. It evaluates a deployment against a set of objectives anchored to real observability data — including security posture — and returns a binary **PASS / FAIL** verdict. A **Dynatrace Workflow** triggers that evaluation automatically on every deployment event and, if production fails its objectives, calls the GitLab API to kick off a rollback pipeline — no human in the loop.

In this use case you will:

1. Create a Guardian in Dynatrace and define objectives — error rate, response time, and critical security vulnerabilities
2. Build a Dynatrace Workflow that evaluates the Guardian automatically after every production deployment
3. Test the PASS and FAIL paths
4. Extend the Workflow to call the GitLab API and trigger a rollback when the evaluation fails
5. Add a rollback job to the pipeline that runs only when Dynatrace fires

---

## 1. Create a Site Reliability Guardian

In Dynatrace, navigate to **Apps → Site Reliability Guardian** (search for it in the app launcher if it isn't pinned).

Click **+ New Guardian**, then choose **Choose Template** and select **Four Golden Signals**.

On the **Getting started with template** popup, click **Run Query**, select **kkm-pulse-demo**, and click **Apply Template**.

### Define objectives

Click **Add More objective** for each of the two below.

#### Objective 4 — Average CPU Usage

| Field | Value |
|---|---|
| **Name** | `Average CPU usage` |
| **DQL** | `timeseries val = avg(dt.kubernetes.container.cpu_usage, default: 0), filter: in(dt.smartscape.k8s_namespace, { toSmartscapeId("<<NAMESPACE_ID_TO_REPLACE>>>") }) | fields avg = arrayAvg(val)` |
| **Fails if result** | `> 50` |
| **Warning if result** | `> 40` |

#### Objective 5 — Critical Security Vulnerabilities

| Field | Value |
|---|---|
| **Name** | `Critical Vulnerabilities` |
| **DQL** | `fetch security.events | filter event.provider=="Dynatrace" | filter event.kind=="SECURITY_EVENT" | filter event.type=="VULNERABILITY_STATE_REPORT_EVENT" | filter event.level=="ENTITY" | fieldsAdd matcher="match" | lookup [ fetch security.events | filter event.provider=="Dynatrace" | filter event.kind=="SECURITY_EVENT" | filter event.type=="VULNERABILITY_STATE_REPORT_EVENT" | filter event.level=="ENTITY" | fields maxTimestamp=timestamp, matcher="match" | limit 1 ], sourceField:matcher, lookupField:matcher, fields:{maxTimestamp} | filter timestamp==maxTimestamp | filter event.status=="OPEN" | filter in(vulnerability.risk.level,{"CRITICAL","HIGH"}) | filter in(affected_entity.id, {"PROCESS_GROUP-A085A3959D385BE8"}) | summarize Filtered_high-profile_vulnerabilities=arraySize(collectDistinct(vulnerability.id))` |
| **Fails criterion** | `> 0` |

!!! info "Application Security required"
    The security objective requires **Dynatrace Application Security** to be enabled. If it's unavailable on your tenant, skip this objective — the error rate and latency objectives are sufficient for the workshop. The principle (SRG can gate on security KPIs the same way it gates on performance KPIs) is the key takeaway.

Save the Guardian. Note the **Guardian ID** from the URL — it looks like `guardian-XXXXXXXXXXXXXXXX`.

---

## 2. Create credentials

### Platform token (for Workflow to trigger SRG)

In Dynatrace: **Settings → Access tokens → Generate new token**

| Field | Value |
|---|---|
| **Name** | `kkm-pulse-demo SRG workflow` |
| **Scopes** | `Davis data: Read` · `Site Reliability Guardian: Read evaluations` · `Site Reliability Guardian: Write evaluations` |

In the `kkm-pulse-demo` GitLab project, add two CI/CD variables:

| Key | Value | Mask? |
|---|---|---|
| `DT_PLATFORM_TOKEN` | the token you just generated | Yes |
| `SRG_GUARDIAN_ID` | your guardian ID (e.g., `guardian-XXXXXXXXXXXXXXXX`) | No |

### GitLab pipeline trigger token (for Workflow to call rollback)

In the `kkm-pulse-demo` GitLab project: **Settings → CI/CD → Pipeline triggers → Add new trigger**

Name it `Dynatrace rollback` and copy:
- The **token** (a long string)
- The **trigger URL**, which looks like `https://gitlab.com/api/v4/projects/12345678/trigger/pipeline`

Note your numeric **project ID** from the URL — you need it below.

### Store the GitLab token in Dynatrace Vault

In Dynatrace: **Settings → Credentials Vault → Add credential**

| Field | Value |
|---|---|
| **Name** | `GITLAB_TRIGGER_TOKEN` |
| **Type** | `Token` |
| **Token value** | the trigger token you just copied |

Vault credentials are encrypted at rest and referenced in Workflows as `{{ vault.GITLAB_TRIGGER_TOKEN }}` — they never appear in plain text in logs or Workflow definitions.

---

## 3. Create a Workflow to evaluate SRG after deployment

The pipeline's `notify-dynatrace-prod-deploy` job already sends a `CUSTOM_DEPLOYMENT` event to Dynatrace every time a production deployment completes. You will now create a Workflow that listens for that event and runs an SRG evaluation automatically — no pipeline polling required.

In Dynatrace: **Apps → Workflows → + New Workflow**

### Trigger — Event-based

| Field | Value |
|---|---|
| **Event type** | `Custom deployment event` |
| **Filter condition** | `event.name == "kkm-pulse-demo" AND event.deployment.environment == "prod"` |

This fires once for every deployment the `notify-dynatrace-prod-deploy` job sends.

### Action 1 — Wait for metrics to stabilize

Add a **JavaScript** action and paste:

```js
import { execution } from "@dynatrace-sdk/automation-utils";

export default async function () {
  // Allow 120 s for post-deployment metrics to propagate
  await new Promise(r => setTimeout(r, 120_000));
  return { waited: true };
}
```

| Field | Value |
|---|---|
| **Label** | `Wait for metrics to stabilize` |

### Action 2 — Trigger SRG evaluation

Add a **Site Reliability Guardian — Run evaluation** action (search for it in the action picker).

| Field | Value |
|---|---|
| **Label** | `Run SRG evaluation` |
| **Guardian** | select `kkm-pulse-demo production` |
| **Timeframe from** | `now-3m` |
| **Timeframe to** | `now` |

The action completes when the evaluation finishes and exposes the result as `{{ result("Run SRG evaluation").executionStatus }}`.

!!! tip "What does the result look like?"
    The SRG action output includes `executionStatus` (`PASS` or `FAIL`), `totalScore`, and a per-objective breakdown. You can inspect it in **Workflow executions → select a run → Action 2 → Output**.

Save and **activate** the Workflow.

### Test the first step

Push a trivial commit to trigger the pipeline through to production, approve the Use Case 5 manual gate, and let `notify-dynatrace-prod-deploy` run.

In Dynatrace: **Apps → Workflows → kkm-pulse-demo SRG → Executions** — a new execution should appear within seconds. After ~2 minutes wait plus evaluation time, the execution completes. Open it and check:

- Action 1 output: `{ "waited": true }`
- Action 2 output: `executionStatus` is `PASS` (if production is healthy)

In **Apps → Site Reliability Guardian → kkm-pulse-demo production** you can see the evaluation listed with its per-objective results.

---

## 4. Force a failing evaluation

Spike the application while an evaluation is running to see the `FAIL` path:

```bash
for i in $(seq 1 8); do
  curl -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/trigger-anomaly &
done
wait
```

Trigger another pipeline run (or re-run the Workflow manually). The latency objective will breach its threshold; the SRG returns `FAIL`; Action 2 output shows `executionStatus: FAIL`.

!!! tip "SRG vs. the load test"
    The load test in Use Case 4 measures what *your curl loop* observed — a synthetic sample under controlled conditions. SRG evaluates metrics Dynatrace collected from *real application traffic* during the evaluation window. These are complementary gates: the load test catches obvious breakage early; SRG catches subtle regressions that only appear under concurrent production load.

---

## 5. Extend the Workflow to trigger a GitLab rollback

With the evaluation result available, add a conditional action that calls the GitLab trigger API only when SRG fails.

Open the Workflow in edit mode and add **Action 3**.

### Action 3 — Trigger GitLab rollback (conditional)

Add an **HTTP Request** action:

| Field | Value |
|---|---|
| **Label** | `Trigger GitLab rollback pipeline` |
| **Run condition** | `{{ result("Run SRG evaluation").executionStatus == "FAIL" }}` |
| **Method** | `POST` |
| **URL** | `https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/trigger/pipeline` |
| **Content-Type** | `application/x-www-form-urlencoded` |

Body (form-encoded):

```
token={{ vault.GITLAB_TRIGGER_TOKEN }}&ref=main&variables[ROLLBACK]=true&variables[ROLLBACK_REASON]=Dynatrace SRG FAIL&variables[ROLLBACK_EVAL_ID]={{ result("Run SRG evaluation").evaluationId }}
```

**Action 4 (optional) — Send notification**: chain a Slack or email notification so on-call is alerted the moment the Workflow fires.

Save and re-activate the Workflow.

!!! tip "Catching Davis AI problems too"
    Add a second event trigger for **Davis Problem** events filtered by entity `kkm-pulse-demo`. This catches issues Davis detects independently of SRG — for example, a memory leak that only becomes visible two hours after deployment — and calls the same rollback action.

---

## 6. Add a rollback job to the pipeline

The Workflow injects `ROLLBACK=true`, `ROLLBACK_REASON`, and `ROLLBACK_EVAL_ID` as pipeline variables when it calls the GitLab trigger API. Add a conditional job that only runs when Dynatrace fires:

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev
  - load_test
  - deploy_prod
  - rollback

rollback-prod:
  stage: rollback
  tags:
    - shell
  rules:
    - if: '$ROLLBACK == "true"'
  script:
    - echo "Rollback triggered by Dynatrace — reason: ${ROLLBACK_REASON} (eval ${ROLLBACK_EVAL_ID})"
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
      -d "{\"eventType\":\"CUSTOM_DEPLOYMENT\",\"title\":\"kkm-pulse-demo ROLLBACK triggered by Dynatrace\",\"properties\":{\"dt.event.deployment.name\":\"kkm-pulse-demo\",\"environment\":\"prod\",\"reason\":\"${ROLLBACK_REASON}\",\"triggered_by\":\"Dynatrace Workflow\",\"srg_evaluation_id\":\"${ROLLBACK_EVAL_ID}\"}}"
```

```bash
git add .gitlab-ci.yaml
git commit -m "ci: add Dynatrace-triggered rollback job"
git push
```

Normal pipeline runs skip `rollback-prod` entirely because `$ROLLBACK` is unset. Only the Dynatrace Workflow-triggered pipeline runs it.

---

## 7. Test the end-to-end loop

1. **Deploy to production** — push a change, run through the pipeline, approve the manual gate, let `notify-dynatrace-prod-deploy` complete

2. **Watch the Workflow fire** — in **Apps → Workflows → kkm-pulse-demo SRG → Executions**, a new execution starts within seconds

3. **Simulate a sustained anomaly** after the pipeline finishes:
    ```bash
    for i in $(seq 1 15); do
      curl -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/trigger-anomaly &
    done
    wait
    ```

4. **Re-trigger the Workflow manually** (or wait for the next deployment). After the SRG evaluation completes with `FAIL`, Action 3 fires.

5. **Watch GitLab** — **CI/CD → Pipelines** shows a new pipeline (branch: `main`, triggered by API) with only the `rollback-prod` job.

6. **Verify the rollback**:
    ```bash
    kubectl rollout history deployment/kkm-pulse-demo -n kkm-pulse-prod
    curl -H "Host: kkm-pulse-prod.127.0.0.1.sslip.io" http://localhost/api/status
    ```

7. **See the rollback event in Dynatrace** — in the `kkm-pulse-demo` service timeline, the `CUSTOM_DEPLOYMENT` rollback event appears alongside the SRG evaluation. The `srg_evaluation_id` property links you directly to the evaluation that triggered it — a complete audit trail with no manual correlation.

This closes the **deploy → observe → act** loop entirely within Dynatrace and GitLab, with no human intervention required.

---

## What you've built

| Capability | Implementation |
|---|---|
| Automated post-deployment validation | Workflow triggers SRG evaluation on every `CUSTOM_DEPLOYMENT` event from the pipeline |
| SRG evaluates real production traffic | Error rate, response time, and security vulnerabilities measured against defined objectives |
| Security vulnerability as a quality gate | SRG objective counts critical CVEs — a deployment with unresolved critical vulnerabilities fails the gate |
| Proactive rollback for post-pipeline issues | Workflow fires on SRG FAIL → calls GitLab trigger API → `rollback-prod` job runs |
| Secure credential handling | GitLab trigger token stored in Dynatrace Vault, never exposed in logs or Workflow YAML |
| Full observability of the rollback itself | `rollback-prod` sends a `CUSTOM_DEPLOYMENT` event back to Dynatrace with the SRG evaluation ID — rollback is visible on the service timeline and cross-linked to the evaluation that caused it |

---

## Knowledge Check

### Question 1 — Why does the Workflow wait 120 seconds before triggering the SRG evaluation?

The JavaScript action waits two minutes before the SRG evaluate action runs. Explain what category of incorrect results becomes more likely if you remove this wait.

??? question "Show Answer"

    SRG evaluates metrics that Dynatrace collected **during the evaluation time window** (`timeframeFrom` to `timeframeTo`). Immediately after a deployment:

    - The newly deployed Pod may still be in its startup phase — the first few requests are handled during JIT compilation, connection-pool warmup, and DNS resolution, producing artificially high latency and a spike in error rate.
    - Dynatrace's metrics pipeline ingests and aggregates data with a small delay; some data points from the first seconds of traffic may not have arrived in the platform yet when the evaluation window closes.

    **Without the wait, you risk a false FAIL:**

    The evaluation window captures the startup noise — elevated latency and possibly some 5xx responses during Pod initialization — and compares it against steady-state thresholds. A healthy deployment can fail its SRG objectives simply because the evaluation ran too early.

    **Why not wait even longer?**

    A longer wait delays feedback. 120 seconds is a pragmatic balance: enough time for the application to reach steady state and for metrics to propagate through the Dynatrace ingest pipeline, but short enough that the feedback loop remains useful. In production, tune this to match your application's actual warm-up profile.

    **Rule of thumb:** set the wait to *at least* the time it takes for your application's error rate to stabilize after a cold start, plus 30–60 seconds for metric ingestion latency.

---

### Question 2 — Hands-on: Add a Slack notification to the rollback Workflow

The Workflow currently triggers a GitLab rollback but sends no human-readable alert. Extend **Action 4** (the optional notification step) so on-call receives a Slack message that includes:

1. Which Guardian evaluation failed (`{{ result("Run SRG evaluation").evaluationId }}`)
2. The total score (`{{ result("Run SRG evaluation").totalScore }}`)
3. A direct link to the evaluation in Dynatrace

Show the Workflow action configuration and the message template you would use.

??? question "Show Answer"

    Add a **Send Slack message** action (requires the Slack connector to be configured in your tenant):

    | Field | Value |
    |---|---|
    | **Label** | `Notify on-call of rollback` |
    | **Run condition** | `{{ result("Run SRG evaluation").executionStatus == "FAIL" }}` |
    | **Channel** | `#oncall-alerts` |

    Message body:

    ```
    :rotating_light: *kkm-pulse-demo production rollback triggered*

    SRG evaluation `{{ result("Run SRG evaluation").evaluationId }}` returned *FAIL* (score: {{ result("Run SRG evaluation").totalScore }}).

    GitLab rollback pipeline has been triggered automatically.

    View evaluation: {{ your-tenant }}/ui/srg/evaluations/{{ result("Run SRG evaluation").evaluationId }}
    ```

    **Why this matters:**

    The Workflow handles the mechanical rollback without waking anyone up, but on-call still needs to know a rollback happened and why. The Slack message provides the evaluation link so they can go straight to the per-objective breakdown — no manual searching in Dynatrace.

---

## Recap — all six use cases together

Across six use cases you took `kkm-pulse-demo` from zero to a fully observable, self-healing pipeline:

1. **Use Case 1** — GitLab project and a self-hosted runner registered over SSH
2. **Use Case 2** — Build, test, SAST, and SonarQube quality gate on every push
3. **Use Case 3** — Docker image built in CI, loaded into k3d, deployed and exposed on Kubernetes
4. **Use Case 4** — Dynatrace OneAgent, deployment events, load test graded against an error budget
5. **Use Case 5** — Separate dev/prod environments with a structural gate: a bad build can never reach the ▶️ button
6. **Use Case 6** — Dynatrace Workflow evaluates SRG on every deployment; triggers GitLab rollback automatically when production degrades

<div class="grid cards" markdown>
- [Continue to Cleanup :octicons-arrow-right-24:](cleanup.md)
</div>
