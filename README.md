<!-- markdownlint-disable-next-line -->
# <img src="https://cdn.bfldr.com/B686QPH3/at/w5hnjzb32k5wcrcxnwcx4ckg/Dynatrace_signet_RGB_HTML.svg?auto=webp&format=pngg" alt="DT logo" width="45"> GitLab CI + Dynatrace — Hands-On Enablement

[![Dynatrace](https://img.shields.io/badge/Dynatrace-Observability-purple?logo=dynatrace&logoColor=white)](https://github.com/domuharahap/dynatrace-gitlab-runner-enablement)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg?color=green)](LICENSE)

___

A hands-on workshop built on the [Dynatrace Enablement Framework](https://dynatrace-wwse.github.io/codespaces-framework) that teaches **GitLab CI/CD** end to end, observed with **Dynatrace**. Everything — the Kubernetes cluster (k3d), the GitLab Runner, SonarQube, and the Dynatrace OneAgent — runs inside a single GitHub Codespace. There is nothing to install on your laptop besides a browser and a free [gitlab.com](https://gitlab.com) account.

<p align="center">
  <img src="docs/img/framework_banner.png" alt="DT Enablement">
</p>

___

## What's in this repo

| Component | Description |
|---|---|
| **kkm-pulse-demo** | The workshop's demo app — a small Express.js "hospital pulse monitor" (`.devcontainer/apps/kkm-pulse-demo`), with a full reference `.gitlab-ci.yaml` and Kubernetes manifests |
| **Framework functions** | Core shell library for cluster management (k3d), ingress, app registry, SonarQube, and Dynatrace credential handling |
| **Workshop docs** | Five progressive use cases, published via MkDocs — see below |

## The six use cases

| # | Use Case | What you'll do |
|---|---|---|
| 1 | [Pipeline Stages, Jobs & Artifacts](docs/usecase1-pipelinestages.md) | Build a pipeline step by step in the GitLab Web IDE: a job, a failing test, stages, and artifacts |
| 2 | [First GitLab Project & Runner](docs/usecase2-gitlabrunner.md) | Create a GitLab.com project, connect over SSH, install & register a GitLab Runner inside the Codespace |
| 3 | [Node.js CI, Test & SAST](docs/usecase3-nodejs-sast.md) | Push `kkm-pulse-demo`, add build/test stages, GitLab SAST, and a SonarQube quality gate |
| 4 | [Docker Build & Deploy to K8s](docs/usecase4-deployk8s.md) | Build a Docker image in CI and deploy it into the local k3d cluster — no registry required |
| 5 | [Dynatrace Events & Load Testing](docs/usecase5-dynatrace.md) | Deploy the OneAgent, send deployment events, run a load test, validate with Davis AI |
| 6 | [Dev/Prod Gates](docs/usecase6-devprodstages.md) | Split into dev/prod environments with a manual gate that structurally blocks a bad build from reaching production |

## Quick start

```bash
# 1. Open this repo in GitHub Codespaces (populate DT_ENVIRONMENT,
#    DT_OPERATOR_TOKEN, DT_INGEST_TOKEN as Codespaces secrets first)
# 2. The k3d cluster starts automatically — verify it:
kubectl get nodes

# 3. Follow the docs starting here:
#    docs/getting-started.md → usecase1 → usecase2 → ... → usecase6
```

Serve the docs locally with:

```bash
pip install -r docs/requirements/requirements-mkdocs.txt
mkdocs serve
```

## Documentation

| Doc | Description |
|---|---|
| [Getting Started](docs/getting-started.md) | Prerequisites, Codespace launch, required secrets |
| [Use Cases 1–5](docs/index.md) | The full workshop, in order |
| [Cleanup](docs/cleanup.md) | Tear down everything the workshop created |
| [Framework functions](docs/functions.md) | Full shell function reference |
| [Framework architecture](docs/framework.md) | Versioned pull model, file classification, image tiers |

## Source repos

- Framework base: [github.com/dynatrace-wwse/codespaces-framework](https://github.com/dynatrace-wwse/codespaces-framework)
