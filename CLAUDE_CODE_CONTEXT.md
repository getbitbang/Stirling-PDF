## Task
Update one field in `helm/values.yaml`.

## The fix
Set `persistence.enabled` to `false`.

Find this block in `helm/values.yaml`:

```yaml
persistence:
  enabled: true
  storageClass: nfs-client
  accessMode: ReadWriteOnce
  size: 5Gi
```

Replace it with:

```yaml
persistence:
  enabled: false
```

Remove the storageClass, accessMode, and size lines entirely.

## Why
The previous failed Helm run left a partial release registered. Helm's
upgrade is now failing because it expects a PVC named
`stirling-pdf-data` that doesn't exist yet. Stirling-PDF is a stateless
tool — it processes PDFs in memory and returns them immediately, nothing
is stored between requests. Persistence is not needed.

## Do not touch
All other files are correct — leave them as-is.