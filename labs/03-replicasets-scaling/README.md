# Lab 3: ReplicaSets and Scaling

## Scenario

During an e-commerce flash sale, traffic can jump from ~100 to ~10,000 requests per
minute. One Pod would not cope, so the checkout service runs as several identical Pods
managed by a ReplicaSet, behind a Service. When the sale ends, the Pod count can go back down.

## Objectives

- Understand ReplicaSets and Pods.
- Scale the Flask app.
- Observe how Pods are distributed.

## Prerequisites

Minikube, kubectl and Docker ([../../docs/prerequisites.md](../../docs/prerequisites.md)).
Unit tests need Python 3.10+ and Flask.

## Files

| File | Purpose |
|------|---------|
| `app.py` | Flask service: `/` (welcome, Pod name, timestamp), `/buy` (simulated checkout; optional `?user=`), `/health` (probes). |
| `test_app.py` | `unittest` tests for the three endpoints. |
| `requirements.txt`, `Dockerfile`, `.dockerignore` | Image running the app under Gunicorn on port 5000 as a non-root user. |
| `k8s/replicaset.yaml` | ReplicaSet `flashsale-rs`: 3 replicas, container `flashsale-container`, probes on `/health`, resource requests and limits. |
| `k8s/service.yaml` | ClusterIP Service `flashsale-svc`, port `http` 80 to container port 5000. |
| `deploy.sh`, `scale.sh`, `spread.sh`, `cleanup.sh` | Optional helper scripts for the steps below. |

The exercise keeps the ReplicaSet and Service in one file (`flashsale-replicaset.yaml`);
here they are two files that create the same two resources.

Names are kept consistent: image `flashsale:1.0`, ReplicaSet `flashsale-rs`, container
`flashsale-container`, Service `flashsale-svc`, label `app=flashsale`. (The exercise's sample
output shows `flask-app-rs` and builds an image called `flask-app`; use the names above.)

## Steps

These follow the exercise's order. Run the commands from this folder.

**Step 1: clean up any previous Minikube cluster.** This deletes your default `minikube`
cluster and everything in it.

```bash
minikube stop
minikube delete
```

**Step 2: start Minikube with a single node and check it.**

```bash
minikube start --nodes=1        # add --driver=docker if Minikube does not pick a driver
kubectl get nodes
```

Expect one node named `minikube`, `STATUS Ready`, `ROLES control-plane`.

**Step 3: the manifests** are `k8s/replicaset.yaml` and `k8s/service.yaml`.

**Step 4: apply them.**

```bash
kubectl apply -f k8s/replicaset.yaml -f k8s/service.yaml
```

Expected output:

```
replicaset.apps/flashsale-rs created
service/flashsale-svc created
```

**Step 5: build the image inside Minikube's Docker.** The Pods cannot start until this image
exists (until then they may show `ErrImageNeverPull` or `ImagePullBackOff`; the kubelet retries).

```bash
eval $(minikube docker-env --shell bash)
docker build -t flashsale:1.0 .
```

**Step 6: verify the Pods and the ReplicaSet.**

```bash
kubectl get pods        # 3 Pods, READY 1/1, STATUS Running
kubectl get rs          # DESIRED 3, CURRENT 3, READY 3
```

**Step 7: scale the ReplicaSet to 5.**

```bash
kubectl scale rs flashsale-rs --replicas=5
```

Expected: `replicaset.apps/flashsale-rs scaled`.

**Steps 8 and 9: verify the ReplicaSet and the Pods.**

```bash
kubectl get rs          # DESIRED 5, CURRENT 5, READY 5
kubectl get pods        # 5 Pods Running; two are newer than the rest
```

**Step 10: delete one Pod** (self-healing).

```bash
kubectl delete pod <pod-name>
```

Expected: `pod "<pod-name>" deleted`.

**Step 11: verify the Pods again, then view their distribution.**

```bash
kubectl get pods        # still 5; one name is new
kubectl get pods -o wide
```

The ReplicaSet created a replacement for the deleted Pod. With one node, all 5 Pods run on
the `minikube` node, each with its own IP.

### Optional helper scripts

Not part of the exercise; they run the same steps (`deploy.sh` builds the image before
applying the manifests, which avoids the `ErrImageNeverPull` window in Step 5).

- `./deploy.sh`: start Minikube if needed, build, apply, wait for the Pods.
- `./scale.sh [n]`: scale to `n` (default 5), then delete a Pod.
- `./spread.sh [n]`: send `n` requests to `/buy` from a throwaway in-cluster client and count
  responses per Pod (`served_by_pod`), to see requests spread over the Pods.
- `./cleanup.sh [--image]`: remove the Service and ReplicaSet (and optionally the image).

## Verification

- `kubectl get nodes` shows one Ready node.
- `kubectl get rs` shows DESIRED = CURRENT = READY.
- `kubectl get pods -o wide` shows every Pod on the single Minikube node, each with its own IP.
- After the delete, the old Pod name is gone and a new one is `Running`.
- `kubectl get endpoints flashsale-svc` lists one address per Ready Pod.
- `spread.sh` output lists several different Pods. Distribution is roughly, not perfectly, even.

## Additional challenges

From the exercise:

- **Use a different image.** Build another tag (for example `flashsale:2.0`), change `image:` in
  `k8s/replicaset.yaml`, and `kubectl apply` it. A ReplicaSet does not replace existing Pods when its
  template changes; delete the Pods to see new ones created from the new image.
- **Create a Deployment instead of a ReplicaSet.** Change `kind: ReplicaSet` to `kind: Deployment`
  in a copy of the manifest (the rest is the same) and apply it; a Deployment manages ReplicaSets
  for you and rolls out image changes.
- **Inspect with `kubectl describe`.** Run `kubectl describe rs flashsale-rs` and
  `kubectl describe pod <pod-name>`.

## Unit tests

```bash
python -m venv .venv && source .venv/bin/activate   # Git Bash: .venv/Scripts/activate
pip install -r requirements.txt
python -m unittest -v
```

## Cleanup

```bash
./cleanup.sh            # remove Service and ReplicaSet
./cleanup.sh --image    # also remove the image from Minikube
```


