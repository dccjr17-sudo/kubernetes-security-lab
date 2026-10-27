# Kubernetes Security Lab

A hands-on project deploying and hardening an Nginx website on a local Kubernetes cluster using Windows, Docker Desktop, kind, and kubectl.

## Architecture

Browser -> localhost:8080 -> kubectl port-forward -> selected Nginx pod

The Service selects three Deployment replicas and maps port 80 to container port 8080. Service-based port-forward selects one pod; it does not demonstrate load balancing. All replicas share one local node.

## Security controls

- Non-root Nginx image; verified UID 101.
- `runAsNonRoot: true` and `allowPrivilegeEscalation: false`.
- All Linux capabilities dropped; verified effective capabilities were zero.
- RuntimeDefault seccomp; verified filtering mode 2.
- Service-account token mounting disabled.
- Per-container CPU request 100m and limit 500m.
- Per-container memory request 64Mi and limit 128Mi.
- HTTP startup, readiness, and liveness probes.

## Verified exercises

| Exercise | Observed result |
| --- | --- |
| Delete a website pod | Deployment created a replacement |
| Scale to three replicas | All three pods became ready |
| Deploy a nonexistent image tag | New pod entered ImagePullBackOff |
| Roll back the update | Deployment successfully rolled out |
| Inspect identity and process restrictions | UID 101, CapEff=0, NoNewPrivs=1, Seccomp=2 |
| Incorrect readiness URL | HTTP 404; replacement remained unready |
| Restore website.yaml | Successful rollout |
| Incorrect liveness URL on a separate test pod | Container restarted twice |
| Clean up test pod | liveness-test deleted |
| Reconnect after hardening | Nginx welcome page loaded |

## Run

Prerequisites: Docker Desktop running Linux containers, kind, and kubectl.

Create the cluster only if it does not already exist:

```powershell
kind create cluster --name security-lab
kubectl --context kind-security-lab wait --for=condition=Ready node/security-lab-control-plane --timeout=120s
```

Deploy:

```powershell
kubectl --context kind-security-lab apply -f website.yaml
kubectl --context kind-security-lab rollout status deployment/first-website --timeout=180s
kubectl --context kind-security-lab port-forward service/first-website 8080:80
```

Open http://localhost:8080. Leave port-forward running; restart it if its selected pod is replaced.

## Failure tests

These files deliberately cause failed health checks. Apply individual files, not the whole directory.

Readiness test (a strategic merge patch, not a standalone manifest):

```powershell
kubectl --context kind-security-lab patch deployment first-website --type=strategic --patch-file readiness-test.yaml
kubectl --context kind-security-lab get pods -l app=first-website
# Restore the working configuration:
kubectl --context kind-security-lab apply -f website.yaml
kubectl --context kind-security-lab rollout status deployment/first-website --timeout=180s
```

Liveness test (separate pod, not selected by the website Service):

```powershell
kubectl --context kind-security-lab apply -f liveness-test.yaml
kubectl --context kind-security-lab get pod liveness-test --watch
# Press Ctrl+C after observing restarts, then inspect and clean up:
kubectl --context kind-security-lab describe pod liveness-test
kubectl --context kind-security-lab delete pod liveness-test
```

## Lessons and limitations

Readiness failures exclude a pod from normal Service traffic; liveness failures restart its container. A Deployment replaces deleted pods. Incorrect probes can disrupt an otherwise working application.

This is a local learning lab, not a production deployment. Three replicas on one node do not provide node-level high availability. The mutable `stable` tag does not guarantee the same image on future runs. TLS, network policies, application-specific RBAC, and namespace admission policies are not configured.

## References

- [kind quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [NGINX unprivileged image](https://github.com/nginx/docker-nginx-unprivileged)
- [Kubernetes security contexts](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes health checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
