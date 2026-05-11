## Task
Two files need to be updated. Apply both changes.

---

## Change 1 — `automation/build-pod-cd.yaml`

Replace the entire file content with the following (single container
instead of two separate helm + kubectl containers):

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: stirling-pdf-cd-agent
spec:
  serviceAccountName: jenkins-agent
  containers:
    - name: helm
      image: dtzar/helm-kubectl:3.17.0
      command: ["cat"]
      tty: true
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi

  restartPolicy: Never
```

## Why
`bitnami/kubectl` tags 1.31 and 1.35.0 do not exist on Docker Hub.
`dtzar/helm-kubectl` bundles both helm and kubectl in one image,
eliminating the need for a separate kubectl container.

---

## Change 2 — `Jenkinsfile.cd`

In the Verify stage, remove the `container('kubectl')` wrapper.
The stage currently looks like this:

```groovy
        stage('Verify') {
            steps {
                container('kubectl') {
                    sh """
                        echo "=== Deployment Status ==="
                        kubectl rollout status deployment/${APP_NAME} \
                            -n ${NAMESPACE} --timeout=3m

                        echo ""
                        echo "=== Pods ==="
                        kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}

                        echo ""
                        echo "=== Ingress ==="
                        kubectl get ingress -n ${NAMESPACE}
                    """
                }
            }
        }
```

It must become this (container wrapper removed, sh block promoted up one level):

```groovy
        stage('Verify') {
            steps {
                sh """
                    echo "=== Deployment Status ==="
                    kubectl rollout status deployment/${APP_NAME} \
                        -n ${NAMESPACE} --timeout=3m

                    echo ""
                    echo "=== Pods ==="
                    kubectl get pods -n ${NAMESPACE} -l app=${APP_NAME}

                    echo ""
                    echo "=== Ingress ==="
                    kubectl get ingress -n ${NAMESPACE}
                """
            }
        }
```

## Why
There is now only one container in the pod (`helm`, which is also the
`defaultContainer`). The `container('kubectl')` reference would fail
because no container named `kubectl` exists anymore.

---

## Do not touch
Jenkinsfile.ci, build-pod-ci.yaml, and all Helm files are correct — leave them as-is.