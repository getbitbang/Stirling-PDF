## Task
Update `helm/values.yaml` — two changes in one edit.

## The fix
Find this block:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

# Stirling-PDF configuration via environment variables
# Full reference: https://docs.stirlingpdf.com/Configuration
envs:
  - name: SYSTEM_MAXFILESIZE
    value: "100"
  - name: SECURITY_ENABLELOGIN
    value: "false"
  - name: UI_APPNAME
    value: "BitBang PDF Editor"
  - name: UI_APPNAMENAVBAR
    value: "PDF Editor"
```

Replace it with:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi

# Stirling-PDF configuration via environment variables
# Full reference: https://docs.stirlingpdf.com/Configuration
envs:
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:MaxMetaspaceSize=256m -Xms256m -Xmx1g"
  - name: SYSTEM_MAXFILESIZE
    value: "100"
  - name: SECURITY_ENABLELOGIN
    value: "false"
  - name: UI_APPNAME
    value: "BitBang PDF Editor"
  - name: UI_APPNAMENAVBAR
    value: "PDF Editor"
```

## Why
The pod is crash-looping with `java.lang.OutOfMemoryError: Metaspace`.
Stirling-PDF loads a very large number of classes (Spring Boot, PDF
libraries, React frontend assets) and the JVM Metaspace grows unbounded
until it hits the 1Gi container limit and the OS kills the process.

- `JAVA_TOOL_OPTIONS` with `-XX:MaxMetaspaceSize=256m` caps Metaspace
  so it cannot consume all available memory
- `-Xmx1g` gives the heap 1GB within the new 2Gi container limit,
  leaving enough headroom for Metaspace + JVM overhead
- Memory limit raised to 2Gi gives the JVM enough total room to run

## Do not touch
All other files are correct — leave them as-is.