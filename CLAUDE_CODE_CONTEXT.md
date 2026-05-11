## Task
Update `automation/build-pod-cd.yaml` — fix the kubectl image tag.

## The fix
`bitnami/kubectl:1.35.0` does not exist on Docker Hub. Bitnami's kubectl
images only go up to 1.31. Change the tag to `1.31` — kubectl version
does not need to exactly match the cluster; within one minor version is
fully supported (1.31 client talking to 1.35 cluster is fine).

## Exact change needed

Find the `kubectl` container in `automation/build-pod-cd.yaml` and change
the image tag from `1.35.0` to `1.31`:

```yaml
    - name: kubectl
      image: bitnami/kubectl:1.31
```

No other files need to change. Do not touch Jenkinsfile.ci, Jenkinsfile.cd,
build-pod-ci.yaml, or any Helm files.