## Task
Delete one file from the Helm templates.

## The fix
Delete `helm/templates/namespace.yaml` entirely.

## Why
The Helm chart tries to create the `stirling-pdf` namespace as a
Helm-managed resource via this template. But the namespace already
exists outside Helm's control, so Helm errors with:

  "namespaces stirling-pdf already exists"

The `--create-namespace` flag already present on the `helm upgrade
--install` command in Jenkinsfile.cd handles namespace creation
correctly — it creates the namespace if missing and skips it silently
if it already exists. The template is redundant and must be removed.

## Exact action
Delete the file: `helm/templates/namespace.yaml`

No other files need to change.