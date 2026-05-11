## Task
Update `helm/values.yaml` — replace the `envs` and `persistence` blocks.

## The fix
Find this block:

```yaml
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

persistence:
  enabled: false
```

Replace it with:

```yaml
envs:
  - name: JAVA_TOOL_OPTIONS
    value: "-XX:MaxMetaspaceSize=256m -Xms256m -Xmx1g"
  - name: DISABLE_ADDITIONAL_FEATURES
    value: "false"
  - name: SECURITY_ENABLELOGIN
    value: "true"
  - name: SECURITY_JWT_PERSISTENCE
    value: "true"
  - name: SYSTEM_MAXFILESIZE
    value: "100"
  - name: UI_APPNAME
    value: "BitBang PDF Editor"
  - name: UI_APPNAMENAVBAR
    value: "PDF Editor"

persistence:
  enabled: true
  storageClass: nfs-client
  accessMode: ReadWriteOnce
  size: 2Gi
```

## Why
The official docs at https://docs.stirlingpdf.com/Configuration/System%20and%20Security/
state that login requires:
- `DISABLE_ADDITIONAL_FEATURES=false` to activate security features in the JAR
- `SECURITY_ENABLELOGIN=true` to show the login screen
- `SECURITY_JWT_PERSISTENCE=true` so JWT signing keys survive pod restarts
  (without this, every restart logs all users out)

Persistence must be re-enabled so the user database file
(`stirling-pdf-DB.mv.db`) stored in `/configs` survives pod restarts.
Without it every restart wipes all users. The current Helm release
(revision 3) has no PVC attached so creating one fresh will not conflict.
NFS client storage class is correct for this app (non-database workload).

## Do not touch
All other files are correct — leave them as-is.