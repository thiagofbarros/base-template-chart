# Plan: Create Vaultwarden Helm Chart

## Objective
Create a new Helm chart for [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (unofficial Bitwarden server) based on the existing `base-template-chart`, and place it inside the `helm-charts` repository at `/home/thiago/Git/helm-charts/charts/vaultwarden/`.

## Source of Truth
- Base chart: `/home/thiago/Git/base-template-chart/`
- Target repo: `/home/thiago/Git/helm-charts/`

## Vaultwarden Requirements
- **Image**: `vaultwarden/server:latest` (or pinned tag)
- **Port**: `80` (HTTP web UI / API)
- **Persistent storage**: `/data` directory must be persisted (database, attachments, etc.)
- **Environment variables**: Vaultwarden is configured almost entirely via env vars
- **Probes**:
  - Liveness: `HTTP GET /alive` on port 80
  - Readiness: `HTTP GET /alive` on port 80
- **Security**: Should run as non-root; however the official image runs as UID 33 (`www-data`) or 1000 depending on tag. We keep `runAsUser: 1000` but note that users may override.
- **Ingress**: Should be enabled by default suggestion (Vaultwarden is typically accessed via a domain)

## Files to Create

### 1. Chart metadata
**Path**: `helm-charts/charts/vaultwarden/Chart.yaml`
- `apiVersion: v2`
- `name: vaultwarden`
- `description: A Helm chart for Vaultwarden (unofficial Bitwarden server)`
- `type: application`
- `version: 1.0.0`
- `appVersion: "1.33.2"` (latest stable at time of planning)

### 2. Default values
**Path**: `helm-charts/charts/vaultwarden/values.yaml`
Adapt from `base-template-chart/values.yaml` with these overrides:
- `application.name: vaultwarden`
- `image.repository: vaultwarden/server`
- `image.tag: "1.33.2"`
- `service.port: 80`
- `persistence.enabled: true`
- `persistence.name: data`
- `persistence.accessMode: ReadWriteOnce`
- `persistence.size: 1Gi`
- `persistence.mountPath: /data`
- `livenessProbe` and `readinessProbe` pointing to `/alive` on port `http`
- Default `env` including common Vaultwarden settings:
  - `WEB_VAULT_ENABLED: "true"`
  - `SIGNUPS_ALLOWED: "false"`
  - `INVITATIONS_ALLOWED: "false"`
  - `ADMIN_TOKEN` commented out (security)
- Sensible default `resources` (low footprint): limits + requests of `256Mi` memory, `100m` CPU
- `ingress.enabled: false` (user must opt-in to avoid accidental exposure)
- `ingress.hosts` example commented out
- Remove `deployment.hpa.enabled: true` scenarios; keep struct but default false

### 3. Templates
All template files are copied from `base-template-chart/templates/` and adapted:
- Replace every occurrence of chart-specific helper name `base-template-chart` with `vaultwarden`.
- Files:
  - `_helpers.tpl`
  - `deployment.yaml`
  - `service.yaml`
  - `ingress.yaml`
  - `pvc.yaml`
  - `configmap.yaml`
  - `hpa.yaml`
  - `httproute.yaml`
  - `external-secret.yaml`
  - `serviceaccount.yaml`

### 4. README
**Path**: `helm-charts/charts/vaultwarden/README.md`
Include:
- Chart name & description
- Installation instructions
- Key configurable values table
- Persistence note (importance of backing up `/data`)
- Security note (change default admin token, use HTTPS)

## Acceptance Criteria
1. `helm-charts/charts/vaultwarden/Chart.yaml` exists with correct metadata.
2. `helm-charts/charts/vaultwarden/values.yaml` exists with Vaultwarden-specific defaults.
3. All templates render without syntax errors: `helm template vaultwarden helm-charts/charts/vaultwarden/ > /dev/null` must exit 0.
4. The rendered output contains a Deployment with image `vaultwarden/server`, a Service on port 80, and a PVC when persistence is enabled.
5. README exists and documents installation + key values.
