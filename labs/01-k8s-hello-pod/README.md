# Lab 1: Kubernetes Hello Pod

## Objective

Run a first containerized application on Kubernetes and reach it from a
browser. You will start a local single-node cluster, create a Pod running
nginx, expose it with a NodePort Service, and open it.

Concepts: Pod (smallest deployable unit), labels and selectors, Service of
type NodePort, readiness probe.

## Prerequisites

- Minikube, kubectl, and a driver (Docker is the default here). See
  [../../docs/prerequisites.md](../../docs/prerequisites.md).

## Files

| File           | Purpose |
|----------------|---------|
| `pod.yaml`     | Pod `hello-pod` running `nginx:1.27-alpine` on port 80, with resource limits and a readiness probe. |
| `service.yaml` | NodePort Service `hello-svc` selecting pods labeled `app=hello,tier=frontend`. |
| `deploy.sh`    | Starts Minikube if needed, applies both manifests, waits for readiness, prints the URL. |
| `cleanup.sh`   | Deletes the Service and Pod; `--stop` also stops Minikube. |

## Steps

Automated:

```bash
./deploy.sh
```

Manual equivalent:

```bash
minikube start --driver=docker
kubectl apply -f pod.yaml
kubectl apply -f service.yaml
kubectl get pods
minikube service hello-svc          # opens the browser
```

Set `MINIKUBE_DRIVER` to use another driver (for example `hyperv`).

## Verification

1. `kubectl get pod hello-pod` shows `STATUS Running` and `READY 1/1`.
2. `kubectl get svc hello-svc` shows type `NodePort`, port `80:<nodePort>/TCP`.
3. `kubectl get endpoints hello-svc` lists the pod IP (a check that the selector matches).
4. `curl "$(minikube service hello-svc --url)"` returns the HTML
   "Welcome to nginx!" page.

## Troubleshooting

- Pod stuck in `ContainerCreating` or `ImagePullBackOff`: check connectivity
  and run `kubectl describe pod hello-pod`.
- Empty endpoints: the Service selector and Pod labels do not match.
- On Windows with the Docker driver, `minikube service` may need to stay
  running in its terminal to keep the tunnel open.




