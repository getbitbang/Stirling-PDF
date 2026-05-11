## Task
Update `helm/values.yaml` — increase the probe delays.

## The fix
Find this block:

```yaml
probes:
  liveness:
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 5
  readiness:
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
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
  readiness:
    initialDelaySeconds: 60
    periodSeconds: 15
    timeoutSeconds: 10
    failureThreshold: 5
```

## Why
The app starts cleanly in ~27 seconds but the liveness probe fires at
30s and sometimes catches it mid-startup, causing a restart. Logs
confirm no errors — the app reaches "Stirling-PDF Started" successfully
every time. Increasing initialDelaySeconds to 60 gives the JVM a
comfortable buffer past the 27s startup time.

## Do not touch
All other files are correct — leave them as-is.