# LiteLLM on OpenShift

A generic, GitOps-ready deployment of [LiteLLM](https://docs.litellm.ai/) on OpenShift Container Platform (OCP).

LiteLLM is an OpenAI-compatible proxy that routes requests to any LLM provider — including RHOAI-hosted models, OpenAI, Anthropic, Cohere, and more. It provides a unified `/v1` API, spend tracking, key management, and an admin UI.

## Features

- **Base deployment** that works out of the box with no models pre-configured
- **Override pattern** — layer a `values-<env>.yaml` on top to add models, change secrets, etc.
- **SQLite by default** — no external database required; models reload from `model_list` in config on every restart, making config the source of truth
- **Optional Postgres** — enable `postgres.enabled: true` for persistent spend tracking and key management
- **`envFrom` support** — mount any Kubernetes Secret as environment variables for `os.environ/` resolution in `config.yaml` (e.g. model API keys from an ESO secret)
- **Consumer Secret** (`litellm-consumer`) — downstream credential Secret; must be populated externally (e.g. a PostSync Job) with a scoped virtual key — the chart never writes the master key into it
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
        ├── secret.yaml                    # litellm-secret: master key (skipped when auth.existingSecret is set)
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

#### Pre-create the master key secret (recommended)

The chart expects a Kubernetes Secret holding `LITELLM_MASTER_KEY` to exist in the target namespace **before** the ArgoCD sync runs. If the secret is absent the LiteLLM pod cannot start (`CreateContainerConfigError`), which acts as a deliberate gate against deploying with an unconfigured key.

```bash
oc create secret generic litellm-master-key \
  --namespace litellm \
  --from-literal=LITELLM_MASTER_KEY="sk-$(openssl rand -hex 16)"
```

> **Keep this key private.** It grants full admin access to the LiteLLM dashboard and key-management API.

Point the chart at this secret in your values override:

```yaml
auth:
  existingSecret: "litellm-master-key"
```

#### Alternative: let the chart manage the secret (local dev / quick testing)

For local development or quick tests, you can have the chart create `litellm-secret` from a value you supply at deploy time. Keep this value out of committed files:

```bash
argocd app set litellm --helm-set auth.masterKey=sk-$(openssl rand -hex 16)
```

Or in a values override file (do not commit it):

```bash
cp helm/values-example.yaml helm/values-myenv.yaml
# Set auth.masterKey and add your model endpoints
```

> **Note:** The chart does not create `litellm-consumer` in either path. Populate it separately (see the Consumer Secret section below).

#### Adding models with runtime key injection

To add models and use environment-variable substitution for API keys, set `api_key: "os.environ/MY_KEY_VAR"` in `model_list` and mount the Secret that holds `MY_KEY_VAR` using the `envFrom` value:

```yaml
# helm/values-myenv.yaml
litellm:
  config: |
    model_list:
      - model_name: my-model
        litellm_params:
          model: openai/my-model
          api_base: http://my-service/v1
          api_key: "os.environ/MY_MODEL_API_KEY"

envFrom:
  - secretRef:
      name: my-model-api-keys   # Secret containing MY_MODEL_API_KEY
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

Use `argocd app create --upsert` with `--helm-set` flags to deploy without modifying any files — useful for quick testing. Pre-create the `litellm-master-key` secret first (see step 2), then:

```bash
argocd app create litellm \
  --repo https://github.com/YOUR_ORG/litellm-on-ocp.git \
  --revision main \
  --path helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace litellm \
  --project default \
  --helm-set auth.existingSecret=litellm-master-key \
  --sync-policy automated \
  --upsert
```

The `--upsert` flag creates the Application if it doesn't exist, or updates it in place if it does. To layer in a model config and enable the virtual key Job as well, add the override file via `--values`:

```bash
argocd app create litellm \
  --repo https://github.com/YOUR_ORG/litellm-on-ocp.git \
  --revision main \
  --path helm \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace litellm \
  --project default \
  --values values.yaml \
  --values values-myenv.yaml \
  --helm-set auth.existingSecret=litellm-master-key \
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

# Pre-create litellm-master-key in the target namespace before deploying — see Quick Start step 2.
auth:
  existingSecret: "litellm-master-key"

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

### Database Backend

**SQLite (default)** — `postgres.enabled: false` is the default. LiteLLM uses a local SQLite file inside the pod. This file is ephemeral (lost on pod restart), but models defined in `model_list` are reloaded from `config.yaml` at every startup, so the config file remains the authoritative source. This is ideal for demos and GitOps-driven deployments where all model configuration lives in the repo.

```yaml
postgres:
  enabled: false   # default
```

**PostgreSQL** — Set `postgres.enabled: true` to deploy a bundled Postgres instance. Required if you need persistent virtual keys, spend tracking across restarts, or DB-backed key management. You must also set a secure password:

```yaml
postgres:
  enabled: true
  password: "my-secure-db-password"
```

### Consumer Secret

The chart **does not create** `litellm-consumer`. Populate it externally with a scoped LiteLLM virtual key so downstream consumers never hold the master key. A PostSync Job is the recommended mechanism — see the demo repos that build on this chart for a working example.

The secret should contain:

| Key | Value |
|-----|-------|
| `LITELLM_API_BASE` | `http://litellm.<namespace>.svc:4000` (internal service URL) |
| `LITELLM_API_KEY` | A scoped LiteLLM virtual key (not the master key) |

Downstream applications reference it via `envFrom` or `secretKeyRef`:

```yaml
envFrom:
  - secretRef:
      name: litellm-consumer
# Provides: LITELLM_API_BASE, LITELLM_API_KEY
```

---

## Sync Wave Order

Resources are created in dependency order using ArgoCD sync waves:

| Wave | Resources |
|------|-----------|
| 40 | Namespace, ServiceAccount, RoleBinding, `litellm-secret`† |
| 41 | ConfigMap, Postgres PVC‡, Postgres Deployment‡, Postgres Service‡ |
| 42 | LiteLLM Deployment, LiteLLM Service |
| 43 | Route |

† `litellm-secret` is only created when `auth.existingSecret` is not set.  
‡ Postgres resources only created when `postgres.enabled: true`.

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
