# Stirling-PDF — BitBang Deployment Guide

## Overview
Self-hosted PDF editor deployed on the BitBang Kubernetes cluster.
- **URL**: https://pdf-editor.getbitbang.com
- **Namespace**: `stirling-pdf`
- **Image**: `192.168.8.72:32083/stirling-pdf-dev:latest`

---

## File Structure Added

```
Stirling-PDF/
├── helm/
│   ├── Chart.yaml
│   ├── values.yaml              ← single environment, no env suffix
│   └── templates/
│       ├── namespace.yaml
│       ├── serviceaccount.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       └── pvc.yaml
├── automation/
│   ├── build-pod-ci.yaml   ← docker + helm containers
│   └── build-pod-cd.yaml   ← helm + kubectl containers
├── Jenkinsfile.ci
└── Jenkinsfile.cd
```

---

## Step 1 — Cloudflare / Nginx (Windows server 192.168.8.61)

Add a new Nginx server block on the Windows reverse proxy for the new hostname.
This mirrors the pattern used by all other apps.

```nginx
server {
    listen 80;
    server_name pdf-editor.getbitbang.com;

    location / {
        proxy_pass         http://192.168.8.70:80;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;

        # Stirling-PDF uploads can be large
        client_max_body_size 200m;
    }
}
```

Add to Cloudflare `config.yaml`:

```yaml
- hostname: pdf-editor.getbitbang.com
  service: http://localhost:80
```

---

## Step 2 — Jenkins Jobs

Create a folder `stirling-pdf` in Jenkins, then add two Pipeline jobs inside it.

### CI job
| Field | Value |
|-------|-------|
| Name | `ci` |
| Type | Pipeline from SCM |
| SCM | Git |
| Repository | `https://github.com/getbitbang/Stirling-PDF` |
| Credentials | `getbitbangtech-github-token` |
| Branch | `*/main` |
| Script Path | `Jenkinsfile.ci` |

### CD job
| Field | Value |
|-------|-------|
| Name | `cd` |
| Type | Pipeline from SCM |
| SCM | Git |
| Repository | `https://github.com/getbitbang/Stirling-PDF` |
| Credentials | `getbitbangtech-github-token` |
| Branch | `*/main` |
| Script Path | `Jenkinsfile.cd` |
| Parameters | Parametrized (ACTION, IMAGE_TAG) |

---

## Step 3 — Run CI Pipeline

Trigger the `stirling-pdf/ci` job. It will:
1. Checkout the forked Stirling-PDF repo
2. Build the Docker image using the existing `Dockerfile` (multi-stage Gradle build)
3. Tag as `stirling-pdf-dev:${BUILD_NUMBER}` and `stirling-pdf-dev:latest`
4. Push both tags to Nexus at `192.168.8.72:32083`

> ⚠️ The first build will take **several minutes** — Stirling-PDF's Dockerfile
> downloads Gradle dependencies and builds the full Java app inside the container.

---

## Step 4 — Run CD Pipeline

Trigger `stirling-pdf/cd` with:
- **ACTION**: `deploy`
- **IMAGE_TAG**: `latest` (or a specific build number)

This will:
1. Run `helm upgrade --install` using `helm/values-dev.yaml`
2. Create namespace `stirling-pdf` and all K8s resources
3. Verify the rollout and print the pod/ingress status

---

## Useful Commands

```bash
# Check pods
kubectl get pods -n stirling-pdf

# Check ingress
kubectl get ingress -n stirling-pdf

# Tail logs
kubectl logs -n stirling-pdf -l app=stirling-pdf -f

# Manual rollback
helm rollback stirling-pdf -n stirling-pdf
```

---

## Configuration

Stirling-PDF is configured via environment variables in `helm/values-dev.yaml` under `envs:`.
Key options:

| Variable | Default | Description |
|----------|---------|-------------|
| `SYSTEM_MAXFILESIZE` | `100` | Max upload size in MB |
| `SECURITY_ENABLELOGIN` | `false` | Enable user authentication |
| `UI_APPNAME` | `BitBang PDF Editor` | App title in UI |
| `LANGS` | (default) | Language pack |

Full config reference: https://docs.stirlingpdf.com/Configuration
