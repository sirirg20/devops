# Lab 6: Prometheus/Grafana Monitoring and a Jenkins Pipeline

## Scenario

A delivery service needs live visibility into how it is performing. A Python
exporter simulates deliveries and publishes Prometheus metrics; Prometheus
scrapes them and raises alerts; Grafana shows a dashboard; and a Jenkins
pipeline tests, validates and deploys the whole stack.

```
 exporter :8000 --scrape--> prometheus :9090 --datasource--> grafana :3000
        (all on one Compose network; services find each other by name)

 Jenkins :8080 --docker.sock--> host Docker --> builds and runs the stack above
```

## Objectives

- Expose application metrics (gauge, counter, summary) with `prometheus_client`.
- Configure scrape jobs and alert rules, and understand `for:` and `rate()`.
- Provision a Grafana data source and dashboard from files instead of clicking.
- Write a declarative Jenkins pipeline with test, validate, deploy and smoke-test stages.

## Prerequisites

Docker with the Compose plugin ([../../docs/prerequisites.md](../../docs/prerequisites.md)).
For the Jenkins part, a Linux or WSL 2 host is recommended because Jenkins uses
the host Docker socket. Local stdlib tests need only Python 3.10+.

## Files

| Path | Purpose |
|------|---------|
| `exporter/simulator.py` | Pure-Python delivery simulation (pending -> in transit -> delivered). |
| `exporter/delivery_metrics.py` | Publishes metrics on port 8000. |
| `exporter/test_simulator.py` | Unit tests for the simulation (stdlib only). |
| `exporter/Dockerfile` | Multi-stage: `test` (runs the unit tests) and `runtime`. |
| `prometheus/` | `prometheus.yml`, `alert_rules.yml`, and a Dockerfile that bakes them in. |
| `grafana/` | Provisioned data source, dashboard JSON, and a Dockerfile that bakes them in. |
| `docker-compose.yml` | The three-service stack on one network. |
| `jenkins/Dockerfile` | Jenkins LTS plus the Docker CLI and Compose plugin. |
| `Jenkinsfile` | The pipeline. |
| `deploy.sh`, `smoke_test.sh`, `jenkins.sh`, `cleanup.sh` | Start the stack, check it, run Jenkins, tear down. |
| `test_configs.py` | Cross-checks configs (metric names, ports, stage order) with the stdlib. |
| `.env.example` | Template for the Grafana admin password. |

## Metrics

| Metric | Type | Meaning |
|--------|------|---------|
| `deliveries_total` | Counter | Deliveries completed since start |
| `deliveries_pending` | Gauge | Orders waiting to be dispatched |
| `deliveries_in_transit` | Gauge | Deliveries on the way |
| `delivery_duration_seconds` | Summary | Time per delivery (`_sum`, `_count`) |

## Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| `HighPendingDeliveries` | `deliveries_pending > 15` for 30s | warning |
| `HighAverageDeliveryTime` | `rate(..._sum[1m]) / rate(..._count[1m]) > 30` for 30s | critical |
| `DeliveryServiceDown` | `up{job="delivery_service"} == 0` for 30s | critical |

Design decisions that differ from a naive setup:

- **Quiet by default.** With the default simulator settings the backlog stays
  well under 15, so alerts fire only when you ask them to.
- **Windowed average.** `_sum / _count` alone is cumulative since startup and reacts
  slowly; the `rate(...[1m])` form reflects the last minute.
- **One Compose network.** Services use names (`exporter:8000`, `prometheus:9090`)
  instead of the Docker bridge address `172.17.0.1` or host networking.
- **Config baked into images.** When Jenkins talks to the host Docker daemon,
  bind-mount paths are resolved on the host, not inside Jenkins, so mounted
  config files would silently be missing. Copying them into the images avoids this.
- **No default Grafana password.** The admin password comes from `.env` or a
  Jenkins credential; the Compose file refuses to start without it.

## Steps

### A. Run the stack directly

```bash
./deploy.sh        # creates .env with a random Grafana password if missing, builds, starts, smoke-tests
```

Then open:

- `http://localhost:8000/metrics` (raw metrics)
- `http://localhost:9090` -> Status > Targets (both `UP`), Alerts
- `http://localhost:3000` -> log in as `admin` with the password in `.env`; the
  **Delivery Overview** dashboard is already provisioned.

Trigger the alerts (set in `.env`, then `docker compose up -d`):

```
ARRIVAL_MAX=8      # backlog builds -> HighPendingDeliveries pending, then firing
DURATION_MAX=80    # slow deliveries -> HighAverageDeliveryTime
```

### B. Run it through Jenkins

1. `./jenkins.sh start`, open `http://localhost:8080`, unlock with the initial admin
   password (command printed by the script), install the suggested plugins.
2. **Manage Jenkins > Credentials > Add**: kind *Secret text*, ID
   `grafana-admin-password`, value = the password you want for Grafana.
3. **Create the Jenkins job**: New Item > Pipeline > *Pipeline script from SCM* >
   Git. Repository URL: your GitHub repo URL once you have pushed, or
   `file:///repo-src` to test your local commits before pushing (the script
   mounts the repository read-only at `/repo-src`). Script Path:
   `labs/06-monitoring-and-jenkins/Jenkinsfile`.
4. Build with parameters. Stages: Pre-check Docker, Unit tests, Validate monitoring
   config (`promtool`), Build images, Deploy stack, Smoke test.
5. Re-run with `ARRIVAL_MAX=8` to see the backlog alert fire, or tick `TEARDOWN`
   to remove the stack at the end.

## Verification

- `./smoke_test.sh` ends with `SMOKE TEST PASSED` (checks exporter metrics,
  Prometheus readiness, the `delivery_service` target being `up`, alert rules
  loaded, Grafana health).
- `docker compose ps` shows three running services.
- Grafana shows all four dashboard panels with moving data; the pending and
  average-time panels have a red threshold line.
- The Jenkins build shows every stage green; the console log ends with the
  smoke-test result.

Local checks that need only Python (no Docker):

```bash
(cd exporter && python test_simulator.py -v)
python test_configs.py -v
```

## Cleanup

```bash
./cleanup.sh              # stop and remove the stack, volumes and built images
./cleanup.sh --jenkins    # also remove the Jenkins container, volume and image
```

## Troubleshooting

- *Target `delivery_service` is DOWN*: the exporter container is not running or
  crashed; `docker compose logs exporter`.
- *Compose says GRAFANA_ADMIN_PASSWORD must be set*: create `.env` or run `./deploy.sh`.
- *Pipeline: `docker: permission denied`*: the Jenkins user must be in the group
  that owns `/var/run/docker.sock` (`jenkins.sh` passes `--group-add` for this).
- *Pipeline can't clone `file:///repo-src`*: `git config --global --add safe.directory /repo-src/.git`
  inside the Jenkins container (the script does this once).
- *Dashboard missing*: check Grafana logs for provisioning errors.

