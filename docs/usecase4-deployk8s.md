--8<-- "snippets/dt-enablement.md"

# Use Case 4 — Docker Build & Deploy to K8s

Your pipeline can build, test, and quality-check `kkm-pulse-demo`. Now let's actually ship it: build a Docker image in CI and run it as a real Deployment on the k3d cluster already running inside this Codespace — **no external registry required**.

---

## Why no registry?

The Codespace's Docker CLI talks to the **same Docker daemon** that k3d uses to run its cluster nodes (the devcontainer mounts `/var/run/docker.sock` from the host). That means an image you build with `docker build` already exists next to the cluster — you just need to hand it to k3d's containerd with `k3d image import`, instead of pushing to Docker Hub or standing up a registry.

```text
gitlab-runner: docker build  →  image in host Docker daemon
                              →  k3d image import
                              →  k3d node containerd
                              →  Deployment (imagePullPolicy: IfNotPresent)
```

---

## 1. Look at the Dockerfile

Already in the repo:

```dockerfile title="Dockerfile"
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

Try it manually once, so the concept is concrete before CI does it for you:

```bash
docker build -t kkm-pulse-demo:manual .
docker run --rm -p 3001:3000 kkm-pulse-demo:manual
```
Validate the docker are able to run and you will see the apps running with port 3000, type **`control+C`** to exit from the docker running.

---

## 2. Look at the Kubernetes manifests

Two files already ship in `manifests/`:

- `manifests/deployment.yaml` — a Deployment + ClusterIP Service. `__NAMESPACE__` and `__IMAGE__` are placeholders the pipeline substitutes at deploy time with `sed`.
- `manifests/ingress-dev.yaml` — an nginx Ingress with **two rules**: a friendly hostname (`kkm-pulse-dev.127.0.0.1.sslip.io`, useful for `curl -H "Host: ..."` checks from the terminal) and a **catch-all** rule with no `host:` at all — that second rule is what makes the app reachable through the Codespace's forwarded port 80 in a real browser, since GitHub's forwarding doesn't send a matching `Host` header.

```yaml title="manifests/deployment.yaml" linenums="1"
# Deployed by GitLab CI — __NAMESPACE__ and __IMAGE__ are substituted at deploy time.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kkm-pulse-demo
  namespace: __NAMESPACE__
  labels:
    app: kkm-pulse-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kkm-pulse-demo
  template:
    metadata:
      labels:
        app: kkm-pulse-demo
    spec:
      containers:
        - name: kkm-pulse-demo
          image: __IMAGE__
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "250m"
          readinessProbe:
            httpGet:
              path: /api/status
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: kkm-pulse-demo
  namespace: __NAMESPACE__
spec:
  type: ClusterIP
  selector:
    app: kkm-pulse-demo
  ports:
    - port: 3000
      targetPort: 3000
```

---

## 3. Add the `package` stage — build & import the image

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package

docker-build:
  stage: package
  tags:
    - shell
  needs:
    - test-job
  script:
    - docker build -t kkm-pulse-demo:${CI_COMMIT_SHORT_SHA} .
    - k3d image import kkm-pulse-demo:${CI_COMMIT_SHORT_SHA} -c enablement
```

`${CI_COMMIT_SHORT_SHA}` gives every pipeline run its own image tag — no accidental "latest" reuse, and you can always trace a running Pod back to the commit that built it.

---

## 4. Add the `deploy_dev` stage

```yaml title=".gitlab-ci.yaml (append)" linenums="1"
stages:
  - build
  - test
  - code_quality
  - package
  - deploy_dev

deploy-dev:
  stage: deploy_dev
  tags:
    - shell
  needs:
    - docker-build
  environment:
    name: development
    url: http://kkm-pulse-dev.127.0.0.1.sslip.io
  script:
    - kubectl create namespace kkm-pulse-dev --dry-run=client -o yaml | kubectl apply -f -
    - sed -e "s#__NAMESPACE__#kkm-pulse-dev#" -e "s#__IMAGE__#kkm-pulse-demo:${CI_COMMIT_SHORT_SHA}#" manifests/deployment.yaml | kubectl apply -f -
    - kubectl apply -f manifests/ingress-dev.yaml
    - kubectl rollout status deployment/kkm-pulse-demo -n kkm-pulse-dev --timeout=120s
    - kubectl port-forward svc/kkm-pulse-demo 18080:3000 -n kkm-pulse-dev &
    - PF_PID=$!
    - sleep 3
    - curl -sf http://localhost:18080/api/status
    - kill $PF_PID || true
```

The last four lines are the smoke test. `kubectl port-forward` opens a tunnel from `localhost:18080` straight to the service — no DNS, no ingress, no sslip.io lookup. If the app doesn't answer after rollout, the job fails here instead of silently leaving a broken deployment behind.

!!! note "Why not curl the hostname directly?"
    `kkm-pulse-dev.127.0.0.1.sslip.io` resolves to `127.0.0.1` via public DNS, but that lookup can fail or be blocked from inside the CI runner's network context. Port-forward is always reliable because it goes through the Kubernetes API server, not the network path.

```bash
git add manifests/ .gitlab-ci.yaml
git commit -m "ci: build docker image and deploy to k3d"
git push
```

---

## 5. Validate

1. Watch `docker-build` and `deploy-dev` turn green in **CI/CD → Pipelines**
2. From the terminal:

    ```bash
    kubectl get all -n kkm-pulse-dev
    ```

3. Open it in a browser: **Ports panel → port 80 → Open in Browser** (make it Public first if you want to share the link with someone else)
4. You should see the KKM Pulse dashboard, and `https://<codespace-name>-80.app.github.dev/api/status` should return the JSON payload

---

## Knowledge Check

### Question 1 — Why does `imagePullPolicy: IfNotPresent` matter here?

The `deployment.yaml` sets `imagePullPolicy: IfNotPresent`. Given this pipeline never pushes to a remote registry, explain why this setting is critical and what would break if you changed it to `Always`.

??? question "Show Answer"

    **`IfNotPresent`** tells the kubelet to use a locally cached image if it already exists in the node's container runtime. After `k3d image import`, the image lives inside k3d's containerd — not in Docker Hub or any remote registry.

    If you set `imagePullPolicy: Always`, Kubernetes would ignore the locally imported image and try to pull from a remote registry on every Pod start. Since the image was never pushed anywhere, the pull would fail with `ErrImagePull` / `ImagePullBackOff` and the Pod would never start.

    **Rule of thumb:** use `IfNotPresent` (or the tag-based default, which behaves the same way for non-`latest` tags) whenever you import or pre-load images onto nodes. Reserve `Always` for registries you actually control and push to.

---

### Question 2 — Hands-on: diagnose a CrashLoopBackOff

Your `deploy-dev` pipeline job turns green and `kubectl rollout status` reports success, but moments later:

```bash
kubectl get pods -n kkm-pulse-dev
```

```
NAME                              READY   STATUS             RESTARTS   AGE
kkm-pulse-demo-7d9f6c8b4-xkq2p   0/1     CrashLoopBackOff   3          90s
```

List the two `kubectl` commands you would run first to find the root cause, and describe what each one tells you.

??? question "Show Answer"

    **Step 1 — check the Pod events and configuration:**

    ```bash
    kubectl describe pod -l app=kkm-pulse-demo -n kkm-pulse-dev
    ```

    This prints the full Pod spec, resource limits, and — most importantly — the **Events** section at the bottom. Events show exactly what Kubernetes tried to do: image pull result, container start attempts, OOM kills, and failed readiness/liveness probes.

    **Step 2 — read the application logs:**

    ```bash
    kubectl logs -l app=kkm-pulse-demo -n kkm-pulse-dev --previous
    ```

    The `--previous` flag fetches logs from the *last* (crashed) container instance rather than the new one that is still starting. This is where you'll see the Node.js stack trace, a missing environment variable, or a port conflict that caused the process to exit.

    **What to look for:**

    | Symptom in output | Likely cause |
    |---|---|
    | `Error: Cannot find module` | `npm install` didn't run or `COPY` missed a file in the Dockerfile |
    | `EADDRINUSE` | Another process owns port 3000 on the node |
    | `OOMKilled` in `describe` | Memory limit too low — raise `resources.limits.memory` |
    | Readiness probe failure | App starts but `/api/status` returns non-2xx — check application logic |

<div class="grid cards" markdown>
- [Continue to Use Case 5 — Dynatrace Events & Load Testing :octicons-arrow-right-24:](usecase5-dynatrace.md)
</div>
