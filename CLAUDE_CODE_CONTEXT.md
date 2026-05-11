## Task
Two files need to be updated. Apply both changes.

---

## Change 1 — `helm/values.yaml`

Find this block:

```yaml
probes:
  liveness:
    initialDelaySeconds: 60
    periodSeconds: 15
    timeoutSeconds: 10
    failureThreshold: 5
  readiness:
    initialDelaySeconds: 60
    periodSeconds: 15
    timeoutSeconds: 10
    failureThreshold: 5
```

Replace it with:

```yaml
probes:
  liveness:
    initialDelaySeconds: 60
    periodSeconds: 15
    timeoutSeconds: 10
    failureThreshold: 5
    path: /actuator/health
  readiness:
    initialDelaySeconds: 60
    periodSeconds: 15
    timeoutSeconds: 10
    failureThreshold: 5
    path: /actuator/health
```

---

## Change 2 — `helm/templates/deployment.yaml`

Find this block:

```yaml
          livenessProbe:
            httpGet:
              path: /
              port: http
              scheme: HTTP
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          readinessProbe:
            httpGet:
              path: /
              port: http
              scheme: HTTP
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
```

Replace it with:

```yaml
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path | default "/" }}
              port: http
              scheme: HTTP
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.liveness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path | default "/" }}
              port: http
              scheme: HTTP
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
            timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
            failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
```

---

## Why
The app starts cleanly in ~27s but gets killed exactly 100s later by the
liveness probe. With login enabled, the root path `/` returns a 302
redirect to `/login` — Kubernetes treats redirects as probe failures.
`/actuator/health` is unauthenticated and always returns
`200 {"status":"UP"}`, making it the correct probe endpoint.

## Do not touch
All other files are correct — leave them as-is.