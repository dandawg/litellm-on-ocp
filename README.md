# LiteLLM on OpenShift

A generic, GitOps-ready deployment of [LiteLLM](https://docs.litellm.ai/) on OpenShift Container Platform (OCP).

LiteLLM is an OpenAI-compatible proxy that routes requests to any LLM provider — including RHOAI-hosted models, OpenAI, Anthropic, Cohere, and more. It provides a unified `/v1` API, spend tracking, key management, and an admin UI.

## Features

- **Base deployment** that works out of the box with no models pre-configured
- **Override pattern** — layer a `values-<env>.yaml` on top to add models, change secrets, etc.
- **Bundled Postgres** for spend tracking and key management (can be disabled)
- **LiteLLM master key auth** — API and admin UI both protected by a single key
- **ArgoCD Application** for GitOps-driven deployment
- **OpenShift Route** with edge TLS termination

## Repository Structure

```
litellm-on-ocp/
├── bootstrap.sh                          # Install OpenShift GitOps if not present
├── bootstrap/
│   └── gitops-operator/
│       ├── base/                         # GitOps Operator subscription
│       └── instance/                     # ArgoCD instance + RBAC
├── gitops/
│   └── litellm.yaml                      # ArgoCD Application manifest
└── helm/
    ├── Chart.yaml
    ├── values.yaml                        # Base values (works as-is)
    ├── values-example.yaml                # RHOAI model override example
    └── templates/
        ├── namespace.yaml
        ├── serviceaccount.yaml
        ├── rolebinding.yaml               # anyuid SCC binding
        ├── secret.yaml                    # Master key + DB credentials
        ├── configmap.yaml                 # LiteLLM config.yaml
        ├── postgres-pvc.yaml
        ├── postgres-deployment.yaml
        ├── postgres-service.yaml
        ├── deployment.yaml
        ├── service.yaml
        └── route.yaml
```

## Quick Start

### Prerequisites

- `oc` CLI authenticated to your OCP cluster
- Helm 3.x (for local rendering/testing)
- OpenShift GitOps (ArgoCD) — `bootstrap.sh` will install it if missing

### 1. Bootstrap GitOps (if needed)

```bash
./bootstrap.sh
```

This installs the OpenShift GitOps Operator and creates an ArgoCD instance if one doesn't already exist. Safe to re-run — it exits early if GitOps is already present.

### 2. Configure your deployment

For a base deployment with no models (useful as a starting point), the defaults in `helm/values.yaml` work as-is. You **must** change the placeholder secrets before deploying:

```yaml
auth:
  masterKey: "sk-changeme"    # replace with a secure key

postgres:
  password: "changeme"        # replace with a secure password
```

To add models and customize further, create an override file:

```bash
cp helm/values-example.yaml helm/values-myenv.yaml
# Edit values-myenv.yaml with real InferenceService URLs and secrets
```

### 3. Update the ArgoCD Application

Edit `gitops/litellm.yaml` and set:

- `spec.source.repoURL` — your fork/copy of this repo
- `spec.source.helm.valueFiles` — add your override file if using one

```yaml
source:
  repoURL: https://github.com/YOUR_ORG/litellm-on-ocp.git
  helm:
    valueFiles:
      - values.yaml
      - values-myenv.yaml   # add your override here
```

### 4. Deploy via ArgoCD

```bash
oc apply -f gitops/litellm.yaml
```

ArgoCD will sync the Helm chart and create all resources in the `litellm` namespace.

#### Alternative: Deploy with inline overrides (no file edits)

Use `argocd app create --upsert` with `--helm-set` flags to deploy without modifying any files — useful for quick testing:

```bash
argocd app create litellm \
  --repo https://github.com/YOUR_ORG/litellm-on-ocp.git \
  --revision main \
  --path helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace demo-apps \
  --project default \
  --helm-set auth.masterKey=sk-my-test-key \
  --helm-set postgres.password=my-test-db-password \
  --sync-policy automated \
  --upsert
```

The `--upsert` flag creates the Application if it doesn't exist, or updates it in place if it does. The `--helm-set` values override `values.yaml` without touching the file. To layer in a model config as well, add the override file via `--values`:

```bash
argocd app create litellm \
  --repo https://github.com/YOUR_ORG/litellm-on-ocp.git \
  --revision main \
  --path helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace demo-apps \
  --project default \
  --values values.yaml \
  --values values-myenv.yaml \
  --helm-set auth.masterKey=sk-my-test-key \
  --helm-set postgres.password=my-test-db-password \
  --sync-policy automated \
  --upsert
```

---

## Accessing LiteLLM

Once deployed, get the Route URL:

```bash
oc get route litellm -n litellm -o jsonpath='https://{.spec.host}'
```

### Admin UI

Navigate to `https://<route-host>/ui` in your browser. Log in with your `LITELLM_MASTER_KEY`.

The UI provides:
- Model management and health checks
- API key creation and management
- Spend tracking and usage dashboards

### API

LiteLLM exposes an OpenAI-compatible API at `/v1`:

```bash
LITELLM_URL=https://$(oc get route litellm -n litellm -o jsonpath='{.spec.host}')

curl $LITELLM_URL/v1/chat/completions \
  -H "Authorization: Bearer sk-your-master-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-model",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

---

## Configuration Override Pattern

The Helm chart uses a layered values approach. The base `values.yaml` is always applied first; one or more override files are merged on top.

### Adding RHOAI Models

Copy `helm/values-example.yaml` and fill in your InferenceService URLs:

```yaml
# helm/values-rhoai.yaml
litellm:
  config: |
    model_list:
      - model_name: granite-3-8b-instruct
        litellm_params:
          model: openai/granite-3-8b-instruct
          api_base: http://<inference-service>.<namespace>.svc.cluster.local/v1
          api_key: "none"

    litellm_settings:
      ssl_verify: false   # required for internal cluster TLS

auth:
  masterKey: "sk-my-secure-key"

postgres:
  password: "my-secure-db-password"
```

Reference it from your ArgoCD Application:

```yaml
helm:
  valueFiles:
    - values.yaml
    - values-rhoai.yaml
```

### Disabling Postgres

Set `postgres.enabled: false` in your override file. LiteLLM will run without spend tracking or DB-backed key management:

```yaml
postgres:
  enabled: false
```

---

## Sync Wave Order

Resources are created in dependency order using ArgoCD sync waves:

| Wave | Resources |
|------|-----------|
| 40 | Namespace, ServiceAccount, RoleBinding, Secret |
| 41 | ConfigMap, Postgres PVC, Postgres Deployment, Postgres Service |
| 42 | LiteLLM Deployment, LiteLLM Service |
| 43 | Route |

---

## Local Helm Testing

Render the chart locally without applying it:

```bash
helm template litellm helm/ -f helm/values.yaml
```

Render with an override file:

```bash
helm template litellm helm/ -f helm/values.yaml -f helm/values-example.yaml
```

Lint the chart:

```bash
helm lint helm/
```
