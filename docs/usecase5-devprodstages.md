--8<-- "snippets/dt-enablement.md"

# Use Case 5 — Dev/Prod Gates

The pipeline builds, tests, scans, containerizes, deploys to dev, and load-tests itself. The last piece: a **separate production environment** that only ever receives a build that already passed the load test — and a manual approval step, since promoting to prod should be a deliberate human decision, not an automatic one.

---

## 1. Look at the prod ingress

`manifests/ingress-prod.yaml` already ships in the repo. Unlike the dev ingress, it has **no catch-all rule** — on purpose, so it never competes with dev for the Codespace's port-80 forward. You'll reach prod with a `Host` header instead of a plain browser URL:

```yaml title="manifests/ingress-prod.yaml" linenums="1"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kkm-pulse-prod-ingress
  namespace: kkm-pulse-prod
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
    - host: kkm-pulse-prod.127.0.0.1.sslip.io
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

---

## 2. Add the `deploy_prod` stage

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev
  - load_test
  - deploy_prod

deploy-prod:
  stage: deploy_prod
  tags:
    - shell
  needs:
    - notify-dynatrace-test-result
  when: manual
  environment:
    name: production
    url: http://kkm-pulse-prod.127.0.0.1.sslip.io
  script:
    - kubectl create namespace kkm-pulse-prod --dry-run=client -o yaml | kubectl apply -f -
    - sed -e "s#__NAMESPACE__#kkm-pulse-prod#" -e "s#__IMAGE__#kkm-pulse-demo:${CI_COMMIT_SHORT_SHA}#" manifests/deployment.yaml | kubectl apply -f -
    - kubectl apply -f manifests/ingress-prod.yaml
    - kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-prod --timeout=120s
    - kubectl port-forward svc/kkm-pulse-demo 18080:3000 -n kkm-pulse-prod &
    - PF_PID=$!
    - sleep 3
    - curl -sf http://localhost:18080/api/status
    - kill $PF_PID || true

notify-dynatrace-prod-deploy:
  stage: deploy_prod
  tags:
    - shell
  needs:
    - deploy-prod
  script:
    - DT_TENANT=$(echo "$DT_ENVIRONMENT" | sed -E 's/\.apps\./.live./; s#/$##')
    - >
      curl -sf -X POST "${DT_TENANT}/api/v2/events/ingest"
      -H "Authorization: Api-Token ${DT_INGEST_TOKEN}"
      -H "Content-Type: application/json"
      -d "{\"eventType\":\"CUSTOM_DEPLOYMENT\",\"title\":\"kkm-pulse-demo deployed to PRODUCTION\",\"entitySelector\":\"type(SERVICE),tag(k8s.namespace.name:kkm-pulse-prod)\",\"properties\":{\"dt.event.deployment.name\":\"kkm-pulse-demo\",\"version\":\"${CI_COMMIT_SHORT_SHA}\",\"environment\":\"prod\"}}"
```

```bash
git add manifests/ingress-prod.yaml .gitlab-ci.yaml
git commit -m "ci: add gated production deployment"
git push
```

### Why this is a real gate, not decoration

- `deploy-prod` declares `needs: [notify-dynatrace-test-result]`. In GitLab CI, a `needs` dependency must **succeed** before the dependent job is even offered — `notify-dynatrace-test-result` is the job from Use Case 4 that `exit 1`s when the load test's error rate exceeds 10%.
- `when: manual` means even a passing pipeline **pauses** at `deploy_prod` — someone has to click ▶️ in the GitLab UI. This models a real approval gate (release manager, change board, whoever you'd want signing off in your org).
- Put both together: a bad build can never reach the manual button, and a good build never ships to prod by accident.

---

## 3. Watch the gate work — twice

### A passing run

1. Push a normal change, watch the pipeline run through `load_test`
2. In **CI/CD → Pipelines**, open the pipeline — `deploy-prod` appears in the graph with a ▶️ (manual) icon, available to click
3. Click it, then confirm `kkm-pulse-prod` is running and open it in your browser:

    ```bash
    kubectl get all -n kkm-pulse-prod
    ```

    Forward port **8080** to the prod service (use a different port than dev's 80 so both can run side-by-side):

    ```bash
    kubectl port-forward svc/kkm-pulse-demo 8080:3000 -n kkm-pulse-prod &
    ```

    Verify it responds:

    ```bash
    curl -sf http://localhost:8080/api/status
    ```

    **Open the prod app in your browser:**

    In the Codespace, go to the **Ports** tab — port **8080** should appear automatically once the forward is running. Click the globe icon next to it (or copy the **Forwarded Address**) to open the production app in your browser.

    The forwarded URL looks like:
    ```
    https://<your-codespace-name>-8080.app.github.dev
    ```

    You should now see the same `kkm-pulse-demo` UI as on dev (port 80), but served from the **`kkm-pulse-prod`** namespace — a completely separate deployment. Both are live at the same time, which is the point: prod only got here because the load test passed and a human clicked ▶️.

### A failing run — prove the gate actually blocks

Introduce an **intermittent failure** — the kind that slips past unit tests because tests only hit the endpoint once, but that load testing catches instantly. Edit `server.js`'s `/api/status` handler:

```js
app.get('/api/status', (req, res) => {
    if (Math.random() < 0.5) {
        res.status(500).json({ error: "intermittent failure — simulated for the workshop" });
    } else {
        res.json({
            hospital: "Hospital Kuala Lumpur (Demo)",
            status: "Normal Operations 🟢",
            activePatients: Math.floor(Math.random() * 50) + 120,
            averageWaitTimeMinutes: Math.floor(Math.random() * 15) + 10,
            staffMood: "Caffeinated & Ready ☕",
            timestamp: new Date().toISOString()
        });
    }
});
```

```bash
git add server.js
git commit -m "chore: simulate intermittent failure for workshop"
git push
```

Watch what happens:

- `test` (npm test) **still passes** — unit tests hit `/api/status` once or twice; 50% chance of success is good enough for a deterministic test run
- `deploy-dev` succeeds — Kubernetes sees the process is up, not what it returns
- `load-test` drives hundreds of requests and records ~50% error rate — far above the 10% budget
- `notify-dynatrace-test-result` fails the error-budget check and exits non-zero
- `deploy_prod` never appears as an available stage — there is no ▶️ button to click

This is the key insight: the unit tests gave you a **false green**. The load test gate is what actually caught the regression before it reached production.

> **Note — pipeline failed but not because of the intentional failure?**
> If your pipeline fails at an unexpected stage (e.g. `deploy-dev` or `load-test` errors out before it even runs requests), it may be a **transient infrastructure error** rather than the simulated bug — things like a momentary runner hiccup, a Kubernetes scheduling delay, or a network blip during image pull. In that case, simply open the failed pipeline in GitLab and click **Retry** (the circular arrow button at the top of the pipeline view). The pipeline will resume from the failed job without needing a new commit.

Check the events feed in Dynatrace — you'll see the dev `CUSTOM_DEPLOYMENT` and the `CUSTOM_INFO` load-test-result event with `error_rate_pct` around 50, but no production deployment event, because it never ran.

Revert the change once you've seen it:

```bash
git revert HEAD --no-edit
git push
```

---

## Knowledge Check

### Question 1 — Why does `when: manual` + `needs` create a real gate, not just a delay?

`deploy-prod` declares both `when: manual` and `needs: [notify-dynatrace-test-result]`. Explain precisely why a failed load test means the manual ▶️ button **never appears** in the pipeline graph — not just that it's blocked after you click it.

??? question "Show Answer"

    In GitLab CI, `needs:` creates a **hard dependency**: a job is only offered for execution (manual or automatic) once all jobs listed in its `needs:` array have **succeeded**. If a `needs` dependency fails, GitLab removes the dependent job from the pipeline graph entirely — it does not show as blocked, skipped, or greyed out; it simply does not appear.

    The chain works like this:

    ```
    load-test
      └─ notify-dynatrace-test-result   ← exits 1 when error rate > 10%
           └─ deploy-prod               ← never offered (no ▶️ button)
                └─ notify-dynatrace-prod-deploy  ← also never offered
    ```

    **Why `when: manual` alone is not a gate:**

    If you only used `when: manual` without `needs:`, the button would appear after every pipeline — including pipelines where the load test failed. A human could still click it and deploy a broken build. The combination of `needs:` (structural dependency on a passing job) and `when: manual` (human approval for a good build) is what makes this a real gate.

    **The key insight:** `needs:` gates on job *outcome*, not just job *completion*. A failing job counts as "did not succeed" and silently removes everything downstream from the graph.

---

### Question 2 — Hands-on: Add an automatic staging deploy before the manual prod gate

Your team wants an intermediate `staging` environment that deploys **automatically** after a passing load test, with production remaining a **manual** approval. Extend the pipeline with a `deploy-staging` job that:

- Runs automatically (no manual click) after `notify-dynatrace-test-result` passes
- Deploys to namespace `kkm-pulse-staging`
- Must succeed before `deploy-prod` becomes available

Sketch the job definition and explain where it fits in the `stages:` list.

??? question "Show Answer"

    Add `deploy_staging` between `load_test` and `deploy_prod` in the stages list, then define the job:

    ```yaml
    stages:
      - build
      - test
      - code_quality
      - package
      - deploy_dev
      - load_test
      - deploy_staging    # ← new
      - deploy_prod

    deploy-staging:
      stage: deploy_staging
      tags:
        - shell
      needs:
        - notify-dynatrace-test-result   # must pass before staging is offered
      environment:
        name: staging
        url: http://kkm-pulse-staging.127.0.0.1.sslip.io
      script:
        - kubectl create namespace kkm-pulse-staging --dry-run=client -o yaml | kubectl apply -f -
        - sed -e "s#__NAMESPACE__#kkm-pulse-staging#" -e "s#__IMAGE__#kkm-pulse-demo:${CI_COMMIT_SHORT_SHA}#" manifests/deployment.yaml | kubectl apply -f -
        - kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-staging --timeout=120s
    ```

    Update `deploy-prod`'s `needs:` to gate on `deploy-staging` instead of `notify-dynatrace-test-result`:

    ```yaml
    deploy-prod:
      stage: deploy_prod
      needs:
        - deploy-staging   # ← staging must succeed before prod button appears
      when: manual
      ...
    ```

    **Why change `deploy-prod`'s `needs:`?**

    You want the chain: load test passes → staging auto-deploys → staging succeeds → prod button appears. If `deploy-prod` still pointed at `notify-dynatrace-test-result`, the prod button would appear in parallel with the staging deploy, before you know whether staging itself is healthy. Pointing `needs` at `deploy-staging` ensures a clean staging deployment is a prerequisite for the prod approval.

---

## Recap

Across five use cases you took `kkm-pulse-demo` from zero to a pipeline that:

1. Builds and tests on every push (Use Case 2)
2. Statically scans the code and enforces a SonarQube quality gate (Use Case 2)
3. Packages a Docker image and deploys it to Kubernetes with no external registry (Use Case 3)
4. Reports every deployment and load-test result to Dynatrace as an event, and gets validated against Davis AI anomaly detection (Use Case 4)
5. Separates dev and prod, and structurally cannot promote a build that failed its load test (Use Case 5)

Use Case 6 goes further: Dynatrace's Site Reliability Guardian validates production against KPI and security objectives *after* deployment, and a Dynatrace Workflow automatically calls back the pipeline to trigger a rollback when production degrades.

<div class="grid cards" markdown>
- [Continue to Use Case 6 — SRG & Automated Rollback :octicons-arrow-right-24:](usecase6-srg-workflow.md)
</div>
