--8<-- "snippets/dt-enablement.md"

# Use Case 8 — Blue-Green Deployment with SRG Pre-Merge Gate

Use Cases 5 and 6 introduced two quality gates: a manual human approval before production, and an automated SRG rollback *after* deployment. Both patterns react to a deployment that has already reached production. This use case moves the validation **earlier** — before the merge to `main` even happens.

The technique is **blue-green deployment**. Instead of updating the live environment in place, you stand up a parallel *green* environment, evaluate it with SRG against real-shaped traffic, and only switch production routing once the Guardian returns `PASS`. If SRG returns `FAIL`, the pipeline that powers the merge request fails — GitLab's merge check prevents the branch from landing in `main`, and the live *blue* environment never sees the broken version.

In this use case you will:

1. Set up two production namespaces — `kkm-pulse-blue` (live) and `kkm-pulse-green` (candidate) — sharing a single ingress that routes to whichever color is active
2. Add a `deploy-green` pipeline job that runs on merge-request branches and deploys the candidate version
3. Run an inline SRG evaluation inside the pipeline and **fail the job** when the Guardian returns `FAIL` — this blocks the merge
4. Add a `promote-green` job that patches the ingress to switch live traffic once the merge lands on `main`
5. Test both the PASS path (merge proceeds, traffic switches) and the FAIL path (merge is blocked, blue stays live)

---

## 1. Understand the blue-green model

```
MR branch pipeline                           main pipeline
─────────────────────────────────            ─────────────────────
build → test → deploy-green                  promote-green
               ↓                             (patches ingress: blue → green)
         SRG evaluation
               ↓
         PASS → MR unblocked ──merge──►
         FAIL → pipeline fails, MR blocked
```

At any point in time, one namespace (`kkm-pulse-blue` or `kkm-pulse-green`) is tagged as the **active** slot and receives all traffic. The other is either idle or running the candidate version. After a successful promotion, the roles swap: green becomes the new live, and blue becomes the new standby — ready for instant rollback with a single `kubectl` patch.

!!! info "Why not just redeploy to prod?"
    Rolling deployments update the live environment in place. Between the first and last pod replacement, traffic hits both old and new versions simultaneously. If the new version is broken, some users experience failures *during* the rollout — before any rollback is possible. Blue-green keeps the old version fully intact and serving 100% of traffic until the new version is validated. The switch is a single atomic ingress patch; rollback is the same patch in reverse.

---

## 2. Bootstrap the blue environment

The blue namespace represents the current production version. If you completed Use Case 6, `kkm-pulse-prod` is already running — promote it to blue by creating the namespace and copying the deployment.

```bash
# Create the blue namespace and deploy the current image
kubectl create namespace kkm-pulse-blue --dry-run=client -o yaml | kubectl apply -f -
CURRENT_IMAGE=$(kubectl get deployment kkm-pulse-demo -n kkm-pulse-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null \
  || echo "kkm-pulse-demo:latest")
sed -e "s#__NAMESPACE__#kkm-pulse-blue#" \
    -e "s#__IMAGE__#${CURRENT_IMAGE}#" \
    manifests/deployment.yaml | kubectl apply -f -
kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-blue --timeout=120s
```

Verify blue is healthy before continuing:

```bash
kubectl port-forward svc/kkm-pulse-demo 19080:3000 -n kkm-pulse-blue &
sleep 3
curl -sf http://localhost:19080/api/status
kill %1
```

---

## 3. Add blue-green ingress and ConfigMap

Create two manifests in the `manifests/` directory. The ingress routes to whichever namespace is recorded as active in the ConfigMap. The pipeline reads the ConfigMap to know which slot is live; a promotion job writes the new value.

```yaml title="manifests/ingress-bluegreen.yaml" linenums="1"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kkm-pulse-bluegreen-ingress
  namespace: kkm-pulse-blue
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
    - host: kkm-pulse.127.0.0.1.sslip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kkm-pulse-demo
                port:
                  number: 3000
```

```yaml title="manifests/bluegreen-state.yaml" linenums="1"
apiVersion: v1
kind: ConfigMap
metadata:
  name: kkm-bluegreen-state
  namespace: default
data:
  active: "blue"
```

Apply both:

```bash
kubectl apply -f manifests/bluegreen-state.yaml
kubectl apply -f manifests/ingress-bluegreen.yaml
```

Confirm the active slot and that the ingress is routing to blue:

```bash
kubectl get configmap kkm-bluegreen-state -n default -o jsonpath='{.data.active}'
# → blue

curl -H "Host: kkm-pulse.127.0.0.1.sslip.io" http://localhost/api/status
```

Commit and push the new manifests in the GitLab Web IDE.

---

## 4. Add the `deploy-green` and `srg-gate` jobs to the pipeline

These two jobs run only on **merge-request pipelines** — not on pushes to `main`. They deploy the candidate version to the green namespace, run a Dynatrace SRG evaluation, and either pass (unblocking the merge) or fail (blocking it).

Add the new stages and jobs to `.gitlab-ci.yaml`:

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev
  - load_test
  - deploy_green      # ← new: MR branch only
  - srg_gate          # ← new: MR branch only
  - deploy_prod
  - promote_green     # ← new: main branch only
  - rollback

deploy-green:
  stage: deploy_green
  tags:
    - shell
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  needs:
    - notify-dynatrace-test-result
  environment:
    name: green
    url: http://kkm-pulse.127.0.0.1.sslip.io
  script:
    - kubectl create namespace kkm-pulse-green --dry-run=client -o yaml | kubectl apply -f -
    - sed -e "s#__NAMESPACE__#kkm-pulse-green#"
          -e "s#__IMAGE__#kkm-pulse-demo:${CI_COMMIT_SHORT_SHA}#"
          manifests/deployment.yaml | kubectl apply -f -
    - kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-green --timeout=120s
    - kubectl port-forward svc/kkm-pulse-demo 20080:3000 -n kkm-pulse-green &
    - PF_PID=$!
    - sleep 3
    - curl -sf http://localhost:20080/api/status
    - kill $PF_PID || true
    - echo "Green deployment ready — candidate version ${CI_COMMIT_SHORT_SHA}"

srg-gate:
  stage: srg_gate
  tags:
    - shell
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  needs:
    - deploy-green
  script:
    - DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')

    # Notify Dynatrace that green is running (so SRG has a deployment event to anchor to)
    - >
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest"
      -H "Authorization: Api-Token ${DT_INGEST_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"eventType\":\"CUSTOM_DEPLOYMENT\",\"title\":\"kkm-pulse-demo green candidate\",\"properties\":{\"dt.event.deployment.name\":\"kkm-pulse-demo\",\"version\":\"${CI_COMMIT_SHORT_SHA}\",\"environment\":\"green\",\"mr_iid\":\"${CI_MERGE_REQUEST_IID}\"}}"

    # Allow 90 s for green metrics to propagate before evaluation
    - echo "Waiting 90 s for metrics to stabilize..."
    - sleep 90

    # Trigger SRG evaluation via API
    - >
      EVAL_RESPONSE=$(curl -sf -X POST
      "${DT_TENANT}/platform/srg/v1alpha/guardian-executions"
      -H "Authorization: Api-Token ${DT_PLATFORM_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"guardianId\":\"${SRG_GUARDIAN_ID}\",\"timeframeFrom\":\"now-3m\",\"timeframeTo\":\"now\"}")
    - EVAL_ID=$(echo "$EVAL_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['executionId'])")
    - echo "SRG evaluation started — ID $EVAL_ID"

    # Poll until the evaluation finishes (max 10 minutes)
    - |
      for i in $(seq 1 20); do
        EVAL_STATUS=$(curl -sf "${DT_TENANT}/platform/srg/v1alpha/guardian-executions/${EVAL_ID}" \
          -H "Authorization: Api-Token ${DT_PLATFORM_TOKEN}" \
          | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('executionStatus','RUNNING'))")
        echo "Poll $i/20 — status: $EVAL_STATUS"
        [ "$EVAL_STATUS" != "RUNNING" ] && break
        sleep 30
      done

    # Fail the pipeline if SRG did not pass
    - |
      if [ "$EVAL_STATUS" != "PASS" ]; then
        echo "SRG evaluation returned ${EVAL_STATUS} — merge is blocked."
        echo "Review the evaluation in Dynatrace: ${DT_ENVIRONMENT}ui/srg/evaluations/${EVAL_ID}"
        exit 1
      fi
    - echo "SRG PASS — evaluation ${EVAL_ID} — merge is unblocked."
```

Commit and push the changes in the GitLab Web IDE.

!!! info "Why `$CI_PIPELINE_SOURCE == 'merge_request_event'`?"
    GitLab runs a separate pipeline for merge requests and a separate one for pushes to branches. Using this rule, `deploy-green` and `srg-gate` run only on the MR pipeline — they don't fire on every push to a feature branch, only when the developer opens or updates a merge request targeting `main`. The `promote-green` job (Section 5) uses the opposite rule and runs only after the merge lands on `main`.

---

## 5. Add the `promote-green` job

Once the MR pipeline passes and the merge lands on `main`, a promotion job switches live traffic from blue to green and updates the ConfigMap so the pipeline knows which slot is now active.

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
promote-green:
  stage: promote_green
  tags:
    - shell
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
  needs:
    - notify-dynatrace-prod-deploy
  script:
    - ACTIVE=$(kubectl get configmap kkm-bluegreen-state -n default -o jsonpath='{.data.active}')
    - echo "Current active slot: $ACTIVE"

    # Switch ingress to green namespace
    - >
      kubectl patch ingress kkm-pulse-bluegreen-ingress -n kkm-pulse-blue
      --type='json'
      -p='[{"op":"replace","path":"/spec/rules/0/http/paths/0/backend/service/name","value":"kkm-pulse-demo"},
           {"op":"replace","path":"/metadata/namespace","value":"kkm-pulse-green"}]'

    # Record new active slot
    - kubectl patch configmap kkm-bluegreen-state -n default --patch '{"data":{"active":"green"}}'

    - echo "Traffic promoted to green (version ${CI_COMMIT_SHORT_SHA})"
    - 'curl -sf -H "Host: kkm-pulse.127.0.0.1.sslip.io" http://localhost/api/status'
    - echo "Blue namespace preserved as rollback target."

    - DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')
    - >
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest"
      -H "Authorization: Api-Token ${DT_INGEST_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"eventType\":\"CUSTOM_DEPLOYMENT\",\"title\":\"kkm-pulse-demo promoted green→production\",\"properties\":{\"dt.event.deployment.name\":\"kkm-pulse-demo\",\"version\":\"${CI_COMMIT_SHORT_SHA}\",\"environment\":\"prod\",\"slot\":\"green\"}}"
```

Commit and push the changes in the GitLab Web IDE.

---

## 6. Test the PASS path — merge proceeds

### Open a merge request

In the GitLab Web IDE, create a new branch `feature/usecase7-bluegreen` with a small, safe change — e.g., add a comment line `// blue-green workshop` to `server.js`. Commit and push the changes to that branch.

In GitLab, open a **New merge request** from `feature/usecase7-bluegreen` to `main`.

### Watch the MR pipeline

In **CI/CD → Pipelines** (filtered to this MR), you should see:

1. `build`, `test`, `code_quality`, `package`, `deploy_dev`, `load_test` — these run as normal
2. `deploy-green` — deploys the candidate to `kkm-pulse-green`
3. `srg-gate` — triggers an SRG evaluation, waits up to 10 minutes, exits 0 on PASS

While `srg-gate` is running, verify the green deployment is live:

```bash
kubectl get pods -n kkm-pulse-green
kubectl port-forward svc/kkm-pulse-demo 20080:3000 -n kkm-pulse-green &
sleep 3
curl -sf http://localhost:20080/api/status
kill %1
```

You should also see two namespaces running the app simultaneously:

```bash
kubectl get deployments -A | grep kkm-pulse
# kkm-pulse-blue    kkm-pulse-demo   1/1 ...
# kkm-pulse-green   kkm-pulse-demo   1/1 ...
```

### After SRG PASS

When `srg-gate` exits 0, the MR pipeline shows all green. The **Merge** button in the GitLab MR is now available. Click **Merge**.

The merge triggers a `main` pipeline. After `deploy_prod` and `notify-dynatrace-prod-deploy` complete, `promote-green` runs and patches the ingress:

```bash
# Confirm active slot switched
kubectl get configmap kkm-bluegreen-state -n default -o jsonpath='{.data.active}'
# → green

# Confirm production traffic is served from green
curl -H "Host: kkm-pulse.127.0.0.1.sslip.io" http://localhost/api/status
```

The blue namespace is still running — it's your instant rollback target.

---

## 7. Test the FAIL path — merge is blocked

### Introduce a regression on a new branch

In the GitLab Web IDE, create a new branch `feature/broken-release`, then edit `server.js` to inject failures on every request:

```js title="server.js (edit — temporary)" linenums="1"
app.get('/api/status', (req, res) => {
  res.status(500).json({ error: "simulated regression — use case 8" });
});
```

Commit and push the changes to the `feature/broken-release` branch in the Web IDE.

Open a **New merge request** from `feature/broken-release` to `main`.

### Watch the MR pipeline block the merge

1. `deploy-green` succeeds — Kubernetes starts the broken version in `kkm-pulse-green`
2. `srg-gate` triggers the SRG evaluation — after the evaluation window, the Errors objective breaches its threshold
3. `srg-gate` exits 1 with:
   ```
   SRG evaluation returned FAIL — merge is blocked.
   Review the evaluation in Dynatrace: ...
   ```
4. The MR pipeline shows **failed**

In GitLab's MR view, the **Merge** button is greyed out with the message:
> **Merge blocked: pipeline has failed**

Blue is unaffected — confirm production is still serving the working version:

```bash
curl -H "Host: kkm-pulse.127.0.0.1.sslip.io" http://localhost/api/status
# → healthy response from blue
```

Open **Apps → Site Reliability Guardian** in Dynatrace and find the evaluation. The per-objective breakdown shows which threshold was breached. This is the signal the developer needs to fix the regression before the MR can land.

### Revert and close the MR

Close the merge request and delete the `feature/broken-release` branch from the GitLab UI.

---

## 8. Manual rollback using the blue slot

If a problem surfaces after promotion (before Use Case 7's automated rollback fires), you can switch back to blue in seconds:

```bash
# Switch ingress back to blue
kubectl patch ingress kkm-pulse-bluegreen-ingress -n kkm-pulse-blue \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/rules/0/http/paths/0/backend/service/name","value":"kkm-pulse-demo"}]'

kubectl patch configmap kkm-bluegreen-state -n default --patch '{"data":{"active":"blue"}}'

echo "Rolled back to blue slot"
curl -H "Host: kkm-pulse.127.0.0.1.sslip.io" http://localhost/api/status
```

!!! tip "Blue-green + Use Case 7 automated rollback"
    The automated Dynatrace Workflow rollback from Use Case 7 remains active. When it fires, update the `rollback-prod` job to use the same `kubectl patch` command above instead of `kubectl rollout undo` — that way, both manual and automated rollbacks use the same mechanism and you always know which slot is live by reading the ConfigMap.

---

## What you've built

| Capability | Implementation |
|---|---|
| Pre-merge SRG gate | `srg-gate` job runs on MR pipelines, fails if Guardian returns FAIL |
| Merge blocked on quality failure | GitLab blocks the Merge button when the MR pipeline fails — no human override needed |
| Zero-downtime traffic switch | Ingress patched atomically to green after merge — blue unchanged until switch |
| Instant rollback target | Blue namespace stays live after promotion; revert is a single `kubectl patch` |
| Full Dynatrace audit trail | Deployment events sent for green candidate, promotion, and any rollback — all visible on the service timeline |
| Colour-agnostic state tracking | ConfigMap records which slot is active — pipeline reads this, not hardcoded namespace names |

---

## Knowledge Check

### Question 1 — Why does the SRG evaluation run on the green namespace, not on blue?

The `srg-gate` job waits for metrics from `kkm-pulse-green`, even though blue is serving production traffic. Explain what this means for the evaluation's validity and what the main limitation is.

??? question "Show Answer"

    **What it means for validity:**

    The SRG evaluation window covers the period after `deploy-green` completed and the 90-second stabilization wait. During that window, green is running but receiving no real user traffic — only the health-check requests the pipeline makes against it via `port-forward`. The metrics Dynatrace collects from green are therefore low-volume and may not reflect the error rate or latency the application would exhibit under production load.

    **The main limitation:**

    SRG evaluates what it can observe. If green has zero or near-zero traffic, the Errors and Traffic objectives may not fire even if the application is broken — there are not enough requests to produce a statistically meaningful error rate. The Traffic objective (minimum requests per minute) is specifically designed to catch this: if green receives no traffic, Traffic fails its floor and the evaluation returns FAIL.

    **How production teams address this:**

    - Send a **canary percentage** (e.g. 5–10%) of real traffic to green via weighted ingress rules while keeping the bulk on blue. SRG then evaluates green under real user load before the full switch.
    - Run a dedicated **smoke test** or **synthetic load** against green during the evaluation window — similar to the load test in Use Case 5, but targeted at the green endpoint.
    - In this workshop, the pipeline health-check and Traffic objective floor provide a minimal but meaningful gate.

---

### Question 2 — Hands-on: Extend the `promote-green` job to clean up the old slot after 24 hours

After a successful promotion, the blue slot stays running indefinitely as a rollback target. For a real environment, you want it cleaned up after a defined retention period to free cluster resources.

Design a GitLab scheduled pipeline or pipeline job that:

1. Checks which slot is currently **inactive** (reads the ConfigMap)
2. Verifies the active slot has been healthy for at least 24 hours (uses Dynatrace DQL or a simple timestamp)
3. Deletes the inactive namespace if healthy

Show the pipeline job and the check you would use.

??? question "Show Answer"

    **Approach — scheduled cleanup pipeline:**

    In GitLab: **CI/CD → Schedules → New schedule** — set the cron to `0 4 * * *` (daily at 04:00 UTC) and the branch to `main`.

    Add a job that only runs on scheduled pipelines:

    ```yaml
    cleanup-inactive-slot:
      stage: rollback
      tags:
        - shell
      rules:
        - if: '$CI_PIPELINE_SOURCE == "schedule"'
      script:
        - ACTIVE=$(kubectl get configmap kkm-bluegreen-state -n default -o jsonpath='{.data.active}')
        - INACTIVE=$([ "$ACTIVE" = "blue" ] && echo "green" || echo "blue")
        - echo "Active: $ACTIVE, candidate for cleanup: $INACTIVE"

        # Check whether the inactive slot exists at all
        - |
          if ! kubectl get namespace kkm-pulse-${INACTIVE} &>/dev/null; then
            echo "Inactive slot kkm-pulse-${INACTIVE} does not exist — nothing to clean up."
            exit 0
          fi

        # Use the ConfigMap to find when the last promotion happened
        - PROMOTED_AT=$(kubectl get configmap kkm-bluegreen-state -n default \
            -o jsonpath='{.metadata.annotations.promoted-at}' 2>/dev/null || echo "")
        - |
          if [ -z "$PROMOTED_AT" ]; then
            echo "No promotion timestamp found — skipping cleanup (safe default)."
            exit 0
          fi
        - NOW=$(date +%s)
        - THEN=$(date -d "$PROMOTED_AT" +%s 2>/dev/null || gdate -d "$PROMOTED_AT" +%s)
        - AGE_HOURS=$(( (NOW - THEN) / 3600 ))
        - echo "Promotion was $AGE_HOURS hours ago"
        - |
          if [ "$AGE_HOURS" -lt 24 ]; then
            echo "Less than 24 hours since promotion — keeping inactive slot."
            exit 0
          fi

        # Safe to delete
        - kubectl delete namespace kkm-pulse-${INACTIVE} --wait=false
        - echo "Cleaned up inactive slot kkm-pulse-${INACTIVE}"
    ```

    **Where to write the promotion timestamp:**

    In `promote-green`, after patching the ConfigMap, annotate it:

    ```bash
    kubectl annotate configmap kkm-bluegreen-state -n default \
      promoted-at="$(date -u +%Y-%m-%dT%H:%M:%SZ)" --overwrite
    ```

    **Why not use Dynatrace for the 24-hour health check?**

    Querying DQL for 24 hours of error-free production traffic is the gold standard (and the approach you'd use if integrating this cleanup into a Dynatrace Workflow). For a workshop pipeline job, a simple timestamp annotation avoids a cross-system dependency and keeps the script self-contained. In a real setup, run both checks: the timestamp as a minimum floor, and a DQL query against the Guardian's latest evaluation to confirm the active slot is still healthy before deleting the fallback.

---

## Recap — all seven use cases together

Across seven use cases you took `kkm-pulse-demo` from zero to a fully observable, self-healing pipeline with a pre-merge quality gate:

1. **Use Case 1** — Pipeline stages, jobs and artifacts built step by step in the GitLab Web IDE
2. **Use Case 2** — GitLab project and a self-hosted runner registered over SSH
3. **Use Case 3** — Build, test, SAST, and SonarQube quality gate on every push
4. **Use Case 4** — Docker image built in CI, loaded into k3d, deployed and exposed on Kubernetes
5. **Use Case 5** — Dynatrace OneAgent, deployment events, load test graded against an error budget
6. **Use Case 6** — Separate dev/prod environments with a structural gate: a bad build can never reach the ▶️ button
7. **Use Case 7** — Dynatrace Workflow evaluates SRG on every deployment; triggers GitLab rollback automatically when production degrades
8. **Use Case 8** — Blue-green deployment with SRG pre-merge gate: broken releases are blocked before they ever touch `main`

<div class="grid cards" markdown>
- [Continue to Cleanup :octicons-arrow-right-24:](cleanup.md)
</div>
