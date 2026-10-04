# kkm-pulse-demo

Demo app for the [GitLab CI + Dynatrace hands-on workshop](https://github.com/domuharahap/dynatrace-gitlab-runner-enablement). A small Express.js "hospital pulse monitor" used to exercise a full GitLab CI pipeline: build, test, SAST, SonarQube, Docker build, Kubernetes deploy, Dynatrace events, and a gated dev/prod promotion.

```text
kkm-pulse-demo/
├── .gitlab-ci.yaml          # full pipeline — see the workshop docs for how it's built up stage by stage
├── Dockerfile
├── package.json
├── server.js
├── manifests/
│   ├── deployment.yaml      # Deployment + Service (namespace/image substituted by CI)
│   ├── ingress-dev.yaml
│   └── ingress-prod.yaml
├── test/
│   └── server.test.js
└── views/
    └── index.html
```

## Endpoints

| Route | Description |
|---|---|
| `GET /` | Dashboard UI |
| `GET /api/status` | Simulated clinic status JSON |
| `GET /api/trigger-anomaly` | Spikes CPU for 3s — useful for demoing Dynatrace Davis AI anomaly detection |

## Run locally

```bash
npm install
npm test
npm start
```

## Workshop

Follow the full walkthrough starting at `docs/usecase3-nodejs-sast.md` in the [dynatrace-gitlab-runner-enablement](https://github.com/domuharahap/dynatrace-gitlab-runner-enablement) repository (or the published MkDocs site, if you have one set up).
