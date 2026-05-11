# Stirling-PDF — Claude Code Review Context

## What you need to do
Review the `Stirling-PDF/` repo and verify every file we added is correct and complete.
The repo was forked from `Stirling-Tools/Stirling-PDF` and lives at `https://github.com/getbitbang/Stirling-PDF`.
A second folder `Stirling-PDF-chart/` contains the official Helm chart clone for reference only.

---

## Infrastructure facts (memorize these)

| Item | Value |
|------|-------|
| Kubernetes | v1.35.0, kubeadm, Ubuntu 22.04 |
| Master | 192.168.8.70 |
| Workers | 192.168.8.71 / .72 / .73 |
| Nexus hosted registry | 192.168.8.72:32083 (HTTP, insecure-registry on all nodes) |
| Nexus proxy (Docker Hub) | 192.168.8.72:32084 |
| Ingress controller | nginx (class: nginx) |
| NFS server | 192.168.8.64:/data/devops-apps |
| Storage class — retain | nfs-retain (databases) |
| Storage class — delete | nfs-client (default tools) |
| Jenkins SA | jenkins-agent (ClusterRole, full permissions) |
| Jenkins agent image | jenkins/inbound-agent:3355.v388858a_47b_33-3-jdk21 |
| Jenkins credential — GitHub | getbitbangtech-github-token |
| Jenkins credential — Nexus | nexus-docker |
| GitHub org | getbitbang / user: getBitbangTech |

## This app's specifics

| Item | Value |
|------|-------|
| App name | stirling-pdf |
| Namespace | stirling-pdf |
| Image in Nexus | 192.168.8.72:32083/stirling-pdf |
| External URL | https://pdf-editor.getbitbang.com |
| App port | 8080 |
| Branch | main (single environment, no dev/prod split) |
| Jenkins folder | stirling-pdf |
| CI job name | ci-pipeline |
| CD job name | cd-pipeline |
| CD parameters | ACTION (deploy/rollback), IMAGE_TAG (default: latest) |

---

## Files we added — verify each one

### `Jenkinsfile.ci`
- `agent.kubernetes.yamlFile` must be `'automation/build-pod-ci.yaml'` (NOT `yaml readFile(...)`)
- `defaultContainer` must be `'docker'`
- `IMAGE_NAME` hardcoded to `192.168.8.72:32083/stirling-pdf`
- Stages: Checkout → Docker Login → Build Image → Push Image → Summary
- `post.always` runs `docker logout`
- No branch detection, no env suffix on image name

### `Jenkinsfile.cd`
- `agent.kubernetes.yamlFile` must be `'automation/build-pod-cd.yaml'` (NOT `yaml readFile(...)`)
- `defaultContainer` must be `'helm'`
- Parameters: `ACTION` (choice: deploy/rollback), `IMAGE_TAG` (string, default: latest)
- No `ENV` parameter (single environment)
- Deploy stage: `helm upgrade --install stirling-pdf ./helm --namespace stirling-pdf --create-namespace --values helm/values.yaml --set image.tag=...`
- Rollback stage: `helm rollback stirling-pdf --namespace stirling-pdf`
- Verify stage runs in `container('kubectl')`
- No `--set image.registry=...` flag needed (registry is baked into values.yaml)

### `automation/build-pod-ci.yaml`
- Two containers: `docker` (docker:27-dind, privileged) and `helm` (alpine/helm:3.14.0)
- `serviceAccountName: jenkins-agent`
- docker container has `DOCKER_TLS_CERTDIR: ""` env var
- emptyDir volume for docker storage
- No maven container (Stirling-PDF's own Dockerfile handles the Gradle build internally)

### `automation/build-pod-cd.yaml`
- Two containers: `helm` (alpine/helm:3.14.0) and `kubectl` (bitnami/kubectl:1.35.0)
- `serviceAccountName: jenkins-agent`
- No docker container needed

### `helm/Chart.yaml`
- `apiVersion: v2`
- `name: stirling-pdf`
- `version: 0.1.0`

### `helm/values.yaml`
- `app.name: stirling-pdf`
- `app.namespace: stirling-pdf`
- NO `app.env` field (single environment)
- `image.registry: 192.168.8.72:32083`
- `image.name: stirling-pdf`
- `image.tag: latest`
- `service.port: 8080`
- `ingress.enabled: true`, `ingress.className: nginx`, `ingress.host: pdf-editor.getbitbang.com`
- `persistence.enabled: true`, `persistence.storageClass: nfs-client`, `persistence.size: 5Gi`
- `securityContext.fsGroup: 1000`
- liveness/readiness probes: `initialDelaySeconds: 30` (Stirling-PDF is slow to start)

### `helm/templates/deployment.yaml`
- image: `{{ .Values.image.registry }}/{{ .Values.image.name }}:{{ .Values.image.tag }}`
- containerPort: `{{ .Values.service.port }}`
- liveness/readiness httpGet path: `/` port: `http`
- If persistence enabled: volumeMounts for `/usr/share/tessdata`, `/configs`, `/customFiles`, `/logs` (all subPaths of one PVC)
- securityContext applied at pod level from `.Values.securityContext`

### `helm/templates/service.yaml`
- ClusterIP, port `{{ .Values.service.port }}`, targetPort `http`

### `helm/templates/ingress.yaml`
- Wrapped in `{{- if .Values.ingress.enabled }}`
- `ingressClassName: {{ .Values.ingress.className }}`
- host: `{{ .Values.ingress.host }}`, path `/`, pathType `Prefix`

### `helm/templates/pvc.yaml`
- Wrapped in `{{- if .Values.persistence.enabled }}`
- name: `{{ .Values.app.name }}-data`
- storageClassName from `{{ .Values.persistence.storageClass }}`

### `helm/templates/namespace.yaml`
- Creates the `stirling-pdf` namespace

### `helm/templates/serviceaccount.yaml`
- Wrapped in `{{- if .Values.serviceAccount.create }}`
- `automountServiceAccountToken: false`

---

## Known bug already fixed
`yaml readFile('automation/build-pod-ci.yaml')` inside the `agent` block fails because
there is no workspace context yet when the agent block is evaluated.
The correct form is `yamlFile 'automation/build-pod-ci.yaml'`.
Make sure BOTH Jenkinsfiles use `yamlFile`, not `yaml readFile(...)`.

---

## What to check in the Stirling-PDF repo root
- Confirm the existing `Dockerfile` is a valid multi-stage build (Gradle → JRE). Do NOT modify it.
- Confirm there is no existing `helm/` or `automation/` or `Jenkinsfile*` that would conflict.
- If any of those exist from the upstream fork, flag it.

---

## What NOT to do
- Do not modify any upstream Stirling-PDF source files (Java, Gradle, Docker, etc.)
- Do not pull from `Stirling-PDF-chart/` unless a template is genuinely missing
- Do not add a `values-dev.yaml` or `values-prod.yaml` — there is only `values.yaml`
- Do not add an `ENV` parameter to the CD pipeline
- Do not use `nfs-retain` storage class for this app (use `nfs-client`)
