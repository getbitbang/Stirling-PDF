## Task
Update `automation/build-pod-ci.yaml` — add `--insecure-registry` args to the docker container.

## The fix
The DinD container runs its own isolated Docker daemon that defaults to HTTPS for all
registries. Our Nexus registry at 192.168.8.72:32083 is HTTP only. The worker nodes
already have it configured as an insecure registry in their daemon.json, but that config
does not carry into the DinD pod. We need to pass the flag directly to dockerd via `args`.

## Exact change needed

Find the `docker` container in `automation/build-pod-ci.yaml` and add an `args` block
immediately after `securityContext`, before `env`:

```yaml
      args:
        - "--insecure-registry=192.168.8.72:32083"
        - "--insecure-registry=192.168.8.72:32084"
```

The result should look exactly like this:

```yaml
    - name: docker
      image: docker:27-dind
      securityContext:
        privileged: true
      args:
        - "--insecure-registry=192.168.8.72:32083"
        - "--insecure-registry=192.168.8.72:32084"
      env:
        - name: DOCKER_TLS_CERTDIR
          value: ""
```

No other files need to change. Do not touch Jenkinsfile.ci, Jenkinsfile.cd,
build-pod-cd.yaml, or any Helm files.