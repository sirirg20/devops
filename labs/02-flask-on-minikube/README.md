# Lab 2: Flask on Minikube

## Objective

Package a small Flask web service as a container image, run it on a local
Minikube cluster with a Deployment, and expose it with a NodePort Service.

Concepts: building images inside Minikube's Docker daemon,
`imagePullPolicy: Never`, Deployment, readiness/liveness probes, `port` vs
`targetPort`, why a Service is needed.

## Prerequisites

Minikube, kubectl and Docker (see [../../docs/prerequisites.md](../../docs/prerequisites.md)).
For local unit tests only: Python 3.10+ and Flask.

## Files

| File | Purpose |
|------|---------|
| `app.py` | Flask app on port 15000: `/` greeting, `/healthz` probe endpoint, `/whoami` returns the Pod name. |
| `test_app.py` | `unittest` tests using Flask's test client. |
| `requirements.txt` | Pinned Flask version. |
| `Dockerfile` / `.dockerignore` | Slim Python image, runs as a non-root user. |
| `k8s/deployment.yaml` | Deployment `flask-app`, 1 replica, `imagePullPolicy: Never`, probes, resource limits. |
| `k8s/service.yaml` | NodePort Service `flask-app-service`, port 15000 to targetPort 15000. |
| `deploy.sh`, `cleanup.sh` | Automate the steps below. |

## Steps

Automated: `./deploy.sh` (set `SKIP_SERVICE=1` to stop after step 5).

Manual:

1. Start the cluster: `minikube start --driver=docker`
2. Point your shell at Minikube's Docker daemon:
   `eval "$(minikube docker-env --shell bash)"`
   (PowerShell: `& minikube -p minikube docker-env --shell powershell | Invoke-Expression`)
3. Build the image there: `docker build -t flask-app:1.0 .`
   Because the image lives only in Minikube's daemon, the Deployment uses
   `imagePullPolicy: Never`.
4. Deploy: `kubectl apply -f k8s/deployment.yaml`
5. Inspect: `kubectl get deployment flask-app`,
   `kubectl get pods -l app=flask-app`, `kubectl describe deployment flask-app`,
   `kubectl logs deployment/flask-app`
6. **Why a Service:** from your host, `curl http://127.0.0.1:15000/` fails. The
   container port belongs to the Pod's network, not your machine.
7. Expose it: `kubectl apply -f k8s/service.yaml`
8. Get a reachable URL: `minikube service flask-app-service --url`
   With the Docker driver on Windows/macOS keep this terminal open, then use
   the printed URL from another terminal.

## Verification

- `kubectl get deployment flask-app` shows `READY 1/1`.
- `kubectl get pods -l app=flask-app` shows `Running`, `READY 1/1`.
- `kubectl logs deployment/flask-app` shows Flask listening on `0.0.0.0:15000`.
- `kubectl get svc flask-app-service` shows type `NodePort`, `15000:<nodePort>/TCP`.
- `curl <url>/` returns `Hello from Flask on Minikube!`
- `curl <url>/healthz` returns `{"status":"ok"}`; `curl <url>/whoami` returns the Pod name.

Local unit tests (no cluster needed):

```bash
python -m venv .venv && source .venv/bin/activate   # Git Bash: .venv/Scripts/activate
pip install -r requirements.txt
python -m unittest -v
```

## Cleanup

```bash
./cleanup.sh            # remove Service and Deployment


