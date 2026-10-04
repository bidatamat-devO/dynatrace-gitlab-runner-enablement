--8<-- "snippets/dt-enablement.md"

# Getting Started

## Prerequisites

Before launching the Codespace, ensure you have everything in place.

!!! warning "Requirements"
    - A **GitLab.com** account — [sign up for free](https://gitlab.com/users/sign_up) if you don't have one
    - A **Dynatrace Platform** environment (SaaS) — [free trial available](https://www.dynatrace.com/signup/)
    - **GitHub Codespaces** access (or a local Dev Container)
    - All **Codespace secrets** populated (see table below)

### Required Secrets

| Secret | Description |
|---|---|
| `DT_ENVIRONMENT` | Your Dynatrace platform URL, e.g. `https://abc123.apps.dynatrace.com` |
| `DT_OPERATOR_TOKEN` | Operator token from the DT UI (auto-created when adding a cluster) |
| `DT_INGEST_TOKEN` | Ingest token for logs, metrics, traces, and **events** |

---

## Part 1 — Launch the Codespace

!!! example "Step-by-step"

    1. Open this repository on GitHub and click **Code > Codespaces > Create codespace on main**
    2. GitHub will prompt you to confirm secrets — verify all three are populated:

        | Secret | Status |
        |---|---|
        | `DT_ENVIRONMENT` | :material-check-circle:{ .green } required |
        | `DT_OPERATOR_TOKEN` | :material-check-circle:{ .green } required |
        | `DT_INGEST_TOKEN` | :material-check-circle:{ .green } required |

    3. Wait for the Codespace to finish initializing — the post-create script starts a local **k3d** Kubernetes cluster and installs `k9s`
    4. Open the **Terminal** panel in VS Code (`View → Open View → Terminal`)
    5. Verify the cluster is running:

    ```bash
    kubectl get nodes
    ```

!!! tip "What the post-create script does"
    The `post-create.sh` script automatically:

    - Creates a **k3d** Kubernetes cluster named `enablement` (context `k3d-enablement`)
    - Installs `k9s` for browsing the cluster
    - Exposes the MkDocs documentation on port 8000

    It does **not** install GitLab, SonarQube, or the Dynatrace Operator — you'll install each of those yourself as you work through the use cases, so you understand exactly what each piece does.

---

## Part 2 — Know Your Forwarded Ports

The Codespace pre-declares three forwarded ports:

| Port | Label | Used for |
|---|---|---|
| `80` | Ingress (Applications) | Any app exposed via the in-cluster nginx ingress — this is how you'll reach `kkm-pulse-demo` from your browser starting in Use Case 4 |
| `8929` | GitLab | Reserved by the framework image; not used in this workshop since we use GitLab.com |
| `9000` | SonarQube | Reachable once you run `installSonarqube` in Use Case 3 |

!!! example "Making a port public"
    1. Open the **Ports** panel in VS Code (`View → Open View → Ports`)
    2. Right-click the port → **Port Visibility → Public**
    3. Click the forwarded URL to open it in your browser — it looks like `https://<codespace-name>-80.app.github.dev`

!!! warning "Revert when done"
    Set ports back to **Private** at the end of the workshop — see [Cleanup](cleanup.md).

---

## Part 3 — What You'll Build

Across the eight use cases you will:

1. Learn how pipeline stages, jobs and artifacts fit together by building a small pipeline step by step in the GitLab Web IDE, running on a **GitLab Runner** you install in this Codespace (Use Case 1)
2. Create a project on **GitLab.com**, connect the Codespace over SSH, and go deeper on runner registration and tags (Use Case 2)
3. Push the `kkm-pulse-demo` Node.js app into that project and build a CI pipeline for it, adding SAST and SonarQube (Use Case 3)
4. Extend the pipeline to build a Docker image and deploy it to the local k3d cluster (Use Case 4)
5. Wire the pipeline into Dynatrace — deployment events, a load test, and Davis AI (Use Case 5)
6. Split the pipeline into dev/prod stages with an automated gate (Use Case 6)
7. Validate production with Dynatrace Site Reliability Guardian and roll back automatically via a Dynatrace Workflow (Use Case 7)
8. Deploy blue-green with an SRG pre-merge gate so broken releases never reach `main` (Use Case 8)

<div class="grid cards" markdown>
- [Continue to Use Case 1 — Pipeline Stages, Jobs & Artifacts :octicons-arrow-right-24:](usecase1-pipelinestages.md)
</div>
