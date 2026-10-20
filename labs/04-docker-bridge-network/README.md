# Lab 4: Docker Bridge Networking

## Scenario

A Flask API, a MySQL database and a Redis cache run as three containers.
They must find each other by name and talk privately, while only the API is
reachable from the host. A user-defined bridge network provides that.

## Objectives

- Create and inspect a user-defined bridge network (driver, scope, subnet, gateway).
- Run containers on it and reach them by container name via Docker's embedded DNS.
- Publish only the API port; keep the database and cache unpublished.
- See why the default `bridge` network is different (no name resolution).

## Prerequisites

Docker ([../../docs/prerequisites.md](../../docs/prerequisites.md)) and `curl`.
Unit tests: Python 3.10+ and Flask. Run the scripts from Git Bash or WSL.

## Layout

```
 host :5001 --> [ web ]  flask-api, port 5001
                  |  lab-bridge-net (user-defined bridge)
          +-------+--------+
        [ db ]          [ cache ]
   mysql:8.4 :3306   redis:7-alpine :6379     (no ports published)
```

## Files

| File | Purpose |
|------|---------|
| `app.py` | `/about` (JSON), `/health`, `/deps` (resolves `db` and `cache` by name and opens TCP connections; 200 if both reachable, else 503). |
| `test_app.py` | `unittest` tests; network calls are mocked. |
| `requirements.txt`, `Dockerfile`, `.dockerignore` | `flask-api` image (non-root, port 5001). |
| `.env.example` | Template for the MySQL root password. |
| `deploy.sh` | Creates the network, builds the image, starts `db`, `cache`, `web`. |
| `verify.sh` | Runs the checks listed below and prints PASS/FAIL. |
| `cleanup.sh` | Removes containers, network and image. |

Versions are pinned (`mysql:8.4`, `redis:7-alpine`) instead of `latest` so the
lab behaves the same next month.

## Steps

Automated: `cp .env.example .env` (edit the password), `./deploy.sh`, `./verify.sh`.
Without a `.env` the script generates a random password for that run.

Manual equivalent:

```bash
docker network create --driver bridge lab-bridge-net
docker network ls
docker network inspect lab-bridge-net
docker build -t flask-api:1.0 .
docker run -d --name db    --network lab-bridge-net -e MYSQL_ROOT_PASSWORD='<password>' mysql:8.4
docker run -d --name cache --network lab-bridge-net redis:7-alpine
docker run -d --name web   --network lab-bridge-net -p 5001:5001 flask-api:1.0
docker exec web getent hosts db cache
curl http://localhost:5001/deps
```

`ping` is not installed in `python:*-slim`; `getent hosts` and `/deps` test the
same thing (name resolution and reachability) without adding packages.

## Verification

1. `docker network ls` lists `lab-bridge-net` with driver `bridge`, scope `local`.
2. `docker network inspect` shows a subnet and gateway (commonly `172.x.0.0/16`;
   the exact range depends on your Docker host) and, after deploy, three
   attached containers.
3. `docker exec web getent hosts db cache` prints an IP for each name.
4. `curl localhost:5001/deps` returns `"ok": true` (allow ~30 s for MySQL to start).
5. `curl localhost:5001/about` returns the service JSON.
6. `docker port db` and `docker port cache` print nothing: unpublished.
7. A container on the default bridge cannot resolve `db`.



## Cleanup

```bash
./cleanup.sh
```

