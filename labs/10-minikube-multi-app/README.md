# Lab 10: Multi-Node Minikube, Multi-Application Deployment

## Scenario

An online store runs two services, a **Product Catalog** and a **Shopping Cart**, and
must stay available if a node fails. Each service runs several replicas spread over a
three-node Kubernetes cluster, so losing one node removes some replicas but not the service.

```
 Minikube profile "lab10": 3 nodes                   namespace lab10
 +-----------+  +-----------+  +-----------+
 |  node 1   |  |  node 2   |  |  node 3   |       product-catalog  x2  -> Service (NodePort :80)
 | (control) |  |           |  |           |       shopping-cart    x3  -> Service (NodePort :80)
 +-----------+  +-----------+  +-----------+                 |
   replicas of each app are spread across nodes                v
                                                    redis x1 -> Service (ClusterIP :6379)
```

## Objectives

- Create a multi-node Minikube cluster and get images onto every node.
- Deploy two applications with Deployments, Services and a shared dependency (Redis).
- Spread replicas across nodes and protect availability with PodDisruptionBudgets.
- Scale replicas, simulate losing a node, and verify the apps keep answering.
- Clean up in stages (resources, then the cluster).

## Prerequisites

- Docker, Minikube and kubectl ([../../docs/prerequisites.md](../../docs/prerequisites.md)). Minikube's Docker
  driver needs hardware virtualization/WSL 2 working; check `minikube start` works first.
- About 6 GB RAM and 6 CPUs free (three nodes), internet access for base images.
- Run the scripts from Git Bash or WSL on Windows. Run Docker as a normal user: the lab
  deliberately never uses `--force` (that flag only exists to let Minikube run as root, which
  weakens isolation).
- For local app tests only: Python 3.10+ (and Flask for the Flask tests).

## Files

| Path | Purpose |
|------|---------|
| `apps/product-catalog/` | Flask service: `GET /products` (3 products), `/health`; Dockerfile (non-root, Gunicorn, port 8080). |
| `apps/shopping-cart/` | Flask service: `GET/POST/DELETE /cart`, `/health` (readiness), `/livez` (liveness); `cart_store.py` chooses memory or Redis. |
| `k8s/00-namespace.yaml` | Namespace `lab10`. |
| `k8s/product-catalog.yaml`, `k8s/shopping-cart.yaml` | Deployment, Service (NodePort) and PodDisruptionBudget each. |
| `k8s/redis.yaml` | Redis Deployment (1 replica) and ClusterIP Service holding the carts. |
| `lib.sh` | Pins every command to this lab's cluster and namespace. |
| `deploy.sh`, `verify.sh`, `scale.sh`, `failover.sh`, `cleanup.sh` | The lab workflow. |
| `miniyaml.py`, `test_miniyaml.py`, `test_manifests.py` | A small YAML-subset reader (PyYAML is not installed here) and static checks across the manifests and scripts. |

## Design decisions (and the problems they avoid)

- **One namespace for everything.** A Service only selects pods in its own namespace; putting
  the Deployment and its Service in different namespaces yields a Service with no endpoints.
  `test_manifests.py` checks that every Service selects pods in its own namespace, and `verify.sh`
  checks the live endpoint counts.
- **A shared store for the cart.** With the cart in each replica's memory, three replicas keep
  three different carts and the user sees a different cart on each request. The cart service
  keeps its cart in Redis, so every replica should agree. `verify.sh` is written to check this by
  reading the cart 15 times and checking that replies come from several replicas yet all show
  the same items. That script has not been run yet; only the store logic is tested (with a fake Redis).
  Redis is a single, non-persistent instance (a restart empties the carts), which is acceptable
  for the lab but is the weak point of the design.
- **Spread without a trap.** `topologySpreadConstraints` (`maxSkew: 1`, `ScheduleAnyway`) spreads
  replicas across nodes but still allows scheduling when a node is gone. A "required" anti-affinity
  rule would leave replacement pods `Pending` when there are fewer nodes than replicas.
  `ScheduleAnyway` is a preference, so the scheduler is not guaranteed to spread perfectly;
  `verify.sh` asserts "more than one node", not "exactly one pod per node".
- **PodDisruptionBudgets** (`minAvailable` 1 of 2 and 2 of 3) keep the service up while a node is
  drained, yet leave room for a drain to proceed.
- **Different probes where it matters.** The cart's readiness check depends on Redis; its
  liveness check does not, because restarting a healthy cart pod does not fix a Redis outage.
- **Local images only.** Images are built locally, loaded onto all nodes with `minikube image load`,
  and referenced with `imagePullPolicy: Never`; there is no registry to configure.
- **Non-root, unprivileged.** Containers run as a numeric non-root user, drop all capabilities,
  forbid privilege escalation, and listen on 8080 (the Services still expose port 80).

## Steps

```bash
./deploy.sh        # 3-node cluster, build + load images, apply manifests, wait for rollouts
./verify.sh        # checks nodes, replicas, spread, endpoints, PDBs, behavior
./scale.sh         # catalog 2 -> 4, cart 3 -> 5; shows pods per node   (./scale.sh --reset to undo)
./failover.sh      # drain a worker, show the apps still answer, uncordon it again
```

Manual equivalents for the core steps:

```bash
minikube start -p lab10 --nodes 3 --driver=docker
docker build -t lab10/product-catalog:1.0 apps/product-catalog
docker build -t lab10/shopping-cart:1.0 apps/shopping-cart
minikube -p lab10 image load lab10/product-catalog:1.0
minikube -p lab10 image load lab10/shopping-cart:1.0
kubectl --context lab10 apply -f k8s/00-namespace.yaml
kubectl --context lab10 apply -f k8s/
kubectl --context lab10 -n lab10 get pods -o wide        # NODE column shows the spread
```

Try the apps from your machine (keep the terminal open; the Docker driver needs the tunnel):

```bash
minikube -p lab10 service product-catalog -n lab10 --url
curl <url>/products
minikube -p lab10 service shopping-cart -n lab10 --url
curl -X POST -H 'Content-Type: application/json' -d '{"item": "Laptop"}' <url>/cart
curl <url>/cart
```

## Verification

`./verify.sh` checks that:

1. The cluster has 3 nodes, all `Ready`.
2. `redis` 1/1, `product-catalog` 2/2 and `shopping-cart` 3/3 replicas are ready.
3. Each app's replicas run on at least two different nodes.
4. Each Service has one endpoint per ready replica.
5. Both PodDisruptionBudgets exist.
6. The catalog returns Laptop, Smartphone and Headphones.
7. The cart starts empty, accepts three items (HTTP 201), rejects an invalid body (400).
8. Fifteen cart reads all show 3 items, answered by at least two different cart replicas.

After `./scale.sh`: `kubectl --context lab10 -n lab10 get pods -o wide` shows 4 catalog and 5
cart pods spread across the nodes. After `./failover.sh`: no application pods on the drained
node while it was drained, and both apps still answered.

## Cleanup

```bash
./cleanup.sh             # delete the namespace (apps, Services, Redis, PDBs), keep the cluster
./cleanup.sh --cluster   # also delete the Minikube cluster and the local images
```

## Troubleshooting

- *`ErrImageNeverPull`*: the image is not on that node; rerun the two `minikube image load` commands.
- *Pods `Pending`*: not enough CPU/memory for three nodes; check `kubectl describe pod`.
- *Cart pods never Ready*: Redis is down or its Service has no endpoints; `kubectl -n lab10 get endpoints redis`.
- *Drain hangs*: a PodDisruptionBudget cannot be satisfied (for example only 2 cart replicas left running);
  scale up or wait for pods to become Ready.
- *`minikube service` returns immediately*: with the Docker driver on Windows/macOS the command must keep running.

ectl` call, scaling, the drain drill
  and `verify.sh`. The manifests were checked by my own reader, not by the Kubernetes API server, so a field
  the API rejects would not be caught until `kubectl apply`. Docker, kubectl and Minikube are not installed
  on the development machine, and hardware virtualization was reported as disabled. See
  [../../docs/testing-status.md](../../docs/testing-status.md).
