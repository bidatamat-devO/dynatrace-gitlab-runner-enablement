--8<-- "snippets/dt-enablement.md"

# GitLab CI + Dynatrace — Hands-On Enablement

!!! example ""
    ![Workshop Banner](img/framework_banner.png){ align=center }

## About this Workshop

This hands-on workshop teaches you how to build a complete CI/CD pipeline with **GitLab CI** and observe it with **Dynatrace** — starting from an empty GitLab.com account and ending with a Node.js application that builds, tests, scans, containerizes, and deploys itself to Kubernetes, gated by automated quality and load-test checks.

Everything runs inside a single **GitHub Codespace**: the Kubernetes cluster (k3d), the GitLab Runner, SonarQube, and the Dynatrace OneAgent all live in the same container — there is nothing to install on your laptop.

!!! info "Source Repository"
    [:material-github: github.com/domuharahap/dynatrace-gitlab-runner-enablement](https://github.com/domuharahap/dynatrace-gitlab-runner-enablement)

---

## What You'll Learn

By the end of this workshop you will be able to:

- [x] Understand **pipeline stages, jobs and artifacts** by building a pipeline step by step in the GitLab Web IDE
- [x] Create a **GitLab.com** project and connect it to your Codespace over SSH
- [x] Install and register a **GitLab Runner** by hand, and reason about executor types and tags
- [x] Build, test, and lint a **Node.js** application (`kkm-pulse-demo`) in a real pipeline
- [x] Add **SAST** and **SonarQube** static analysis quality gates to the pipeline
- [x] Build a **Docker image** in CI and load it into the local Kubernetes cluster (no registry needed)
- [x] Deploy the application to Kubernetes from GitLab CI and reach it from a browser
- [x] Send **Dynatrace deployment events** and load-test results from the pipeline via the Events API v2
- [x] Split the pipeline into **dev** and **prod** stages with an automated gate that blocks a bad build from reaching production
- [x] Validate every production deployment with **Site Reliability Guardian** — KPI and security vulnerability objectives — and auto-rollback via a Dynatrace Workflow if production degrades after the pipeline finishes

---

## The Demo App — `kkm-pulse-demo`

A small Express.js app simulating a hospital pulse-monitoring dashboard, used as the workshop's running example across the use cases from Use Case 3 onward:

| Endpoint | Purpose |
|---|---|
| `GET /` | Serves the dashboard UI |
| `GET /api/status` | Returns simulated clinic status JSON |
| `GET /api/trigger-anomaly` | Spikes CPU for 3s — great for showing Davis AI anomaly detection |

Source lives at [.devcontainer/apps/kkm-pulse-demo](https://github.com/domuharahap/dynatrace-gitlab-runner-enablement/tree/main/.devcontainer/apps/kkm-pulse-demo) inside this repository — you'll push a copy of it to your own GitLab project in Use Case 3.

---

## Workshop Structure

| Use Case | Content |
|---|---|
| [Getting Started](getting-started.md) | Prerequisites, Codespace launch, Dynatrace secrets |
| [1 — Pipeline Stages, Jobs & Artifacts](usecase1-pipelinestages.md) | Build a pipeline step by step in the GitLab Web IDE: a job, a failing test, stages, artifacts |
| [2 — First GitLab Project & Runner](usecase2-gitlabrunner.md) | Create a GitLab.com project, SSH keys, install & register a GitLab Runner |
| [3 — Node.js CI, Test & SAST](usecase3-nodejs-sast.md) | Push `kkm-pulse-demo`, build/test stages, GitLab SAST, SonarQube quality gate |
| [4 — Docker Build & Deploy to K8s](usecase4-deployk8s.md) | Build a Docker image in CI, load it into k3d, deploy & expose it |
| [5 — Dynatrace Events & Load Testing](usecase5-dynatrace.md) | Deploy the OneAgent, send deployment events, run a load test, validate in Dynatrace |
| [6 — Dev/Prod Gates](usecase6-devprodstages.md) | Separate dev/prod environments, manual approval, stop a bad build automatically |
| [7 — Site Reliability Guardian & Automated Rollback](usecase7-srg-workflow.md) | SRG validates prod KPIs and security vulnerabilities; Dynatrace Workflow triggers automatic rollback |
| [8 — Blue-Green Deployment with SRG Pre-Merge Gate](usecase8-blue-green-srg.md) | Deploy to a green namespace, let SRG gate the merge, and block broken releases before they reach `main` |
| [Cleanup](cleanup.md) | Tear down everything created during the workshop |
| [Resources](resources.md) | Reference links and further reading |

<div class="grid cards" markdown>
- [Let's get started :octicons-arrow-right-24:](getting-started.md)
</div>
