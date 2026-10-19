# Lab 9: Multi-Stage Jenkins Pipeline for Flask

## Scenario

A Flask service needs a repeatable path from commit to running software.
A Jenkins **declarative pipeline** (a `Jenkinsfile` kept in the repository)
builds the app, checks it, tests it, packages it, deploys it and checks that the
deployment responds, stopping at the first stage that fails. (The pipeline has
not been run in Jenkins yet; see Testing status below.)

```
Prepare -> Build -> Lint -> Test -> Package -> Deploy -> Smoke test
                              |         |         |          |
                       JUnit results  tarball  versioned   health check,
                       published     archived  release +   then always
                                               "current"   stop the app
```

## Objectives

- Write a multi-stage declarative `Jenkinsfile` with parameters, conditions, `post` blocks.
- Keep stage logic in scripts (`ci/*.sh`) so it can be tested without Jenkins.
- Publish test results and archive a versioned build artifact.
- Deploy to a versioned release directory, switch a `current` link, and prune old releases.
- Smoke-test the deployment and never leave a stray process behind.

## Prerequisites

- Docker with the Compose plugin ([../../docs/prerequisites.md](../../docs/prerequisites.md)), `curl`, `git`.
- The lab brings its own Jenkins (container `lab9-jenkins`) with Python, venv and pip
  **built into the image**. Nothing is installed by hand into a running container.
- Stop other Jenkins containers first or set `JENKINS_PORT`.
- **Commit before building.** The pipeline checks out the repository over
  `file:///repo-src` (your local repo, mounted read-only), so it builds your *committed*
  state, not uncommitted edits. After you publish the repository yourself, set `REPO_URL`
  in `.env` to the GitHub URL.

## Files

| Path | Purpose |
|------|---------|
| `app/app.py`, `app/test_app.py` | Flask app (`/`, `/health`, `/version`) and its unittest tests. |
| `app/requirements.txt`, `app/requirements-dev.txt`, `app/.flake8` | Pinned runtime and dev dependencies; lint config. |
| `Jenkinsfile` | The pipeline. |
| `ci/package.sh` | Builds `flask-app-<n>.tar.gz` (app, runtime requirements, `VERSION`; no tests). |
| `ci/deploy.sh` | Extracts to `releases/<n>`, creates the runtime venv, swaps `current`, prunes old releases. |
| `ci/run_app.sh` | `start`, `stop`, `status` with a PID file. |
| `ci/smoke_test.sh` | Retries `/`, `/health`, `/version` (and the expected version). |
| `jenkins/` | Dockerfile (Jenkins + Python), `plugins.txt`, admin-user init script. |
| `docker-compose.yml`, `.env.example` | Run Jenkins; settings and the admin password template. |
| `job/pipeline-config.xml.tmpl`, `render_job.sh`, `create_job.sh`, `lib.sh` | The pipeline job as code, created through the REST API. |
| `deploy.sh`, `verify.sh`, `cleanup.sh` | Bring up, verify with a real build, tear down. |
| `test_pipeline.py` | Tests of the scripts, job rendering and Jenkinsfile (no Jenkins needed). |

## Pipeline stages

| # | Stage | What it does | Fails when |
|---|-------|--------------|------------|
| 1 | Prepare | Prints git/python versions and the commit. | Tooling is missing. |
| 2 | Build | Creates `.venv` and installs the pinned dev requirements. | A dependency cannot be installed. |
| 3 | Lint | `flake8` with the config in `app/.flake8`. | Style or syntax errors. |
| 4 | Test | `pytest` writes `reports/junit.xml`; Jenkins publishes it. | Any test fails (results are still published). |
| 5 | Package | `ci/package.sh $BUILD_NUMBER`; archives `dist/*.tar.gz` with a fingerprint. | Packaging fails. |
| 6 | Deploy | `ci/deploy.sh` to `$JENKINS_HOME/deploy/flask-app`; keeps `KEEP_RELEASES`. Skipped when `DEPLOY` is false. | Bad archive, venv or link failure. |
| 7 | Smoke test | Starts the deployed app, checks `/`, `/health` and that `/version` equals the build number. The `post { always }` block stops the app. | Any check fails after retries. |

Parameters: `DEPLOY` (default true) and `KEEP_RELEASES` (default 3).

Differences from a naive pipeline, and why:

- **Build does real work** (isolated venv, pinned install) instead of printing a message.
- **Tests run once**, in the Test stage, with machine-readable results. The Smoke test
  checks the *deployed* app over HTTP, which is a different thing.
- **The app is always stopped** (PID file plus a `post { always }` block); the run that
  starts it can never leave it running.
- **Releases are versioned** with an atomic-style switch of `current` and automatic pruning,
  so a bad release can be rolled back by pointing `current` at an earlier one.
- **Tooling is in the image**, so rebuilding the container does not lose Python.

## Steps

```bash
git status                 # commit your work first; the pipeline builds committed code
./deploy.sh                # Jenkins + job (admin password generated into .env if missing)
./verify.sh                # runs the pipeline and checks every stage and output
```

Or in the UI: `http://localhost:8080/job/FlaskPipeline/` -> **Build Now**. Open the build to
see the stages, **Console Output**, **Test Result** and the archived artifact.

Create the job by hand instead: New Item > Pipeline > *Pipeline script from SCM* > Git,
Repository URL `file:///repo-src`, Branch `*/main`, Script Path
`labs/09-jenkins-flask-pipeline/Jenkinsfile`.

Try the failure paths: break a test in `app/test_app.py`, commit, build, and watch the pipeline
stop at Test with the failure in the Test Result page and no package or deploy.

## Verification

`./verify.sh` checks that:

1. Jenkins is reachable and the `FlaskPipeline` job exists; a build is queued.
2. The build finishes `SUCCESS`.
3. All seven stages appear in the console in order.
4. `SMOKE TEST PASSED` is in the log.
5. A JUnit report with more than zero tests is published.
6. `flask-app-<n>.tar.gz` is archived and downloadable.
7. Release `<n>` is the active deployment, and no `app.pid` remains.

Manual spot checks: `docker exec lab9-jenkins ls /var/jenkins_home/deploy/flask-app/releases`
shows at most `KEEP_RELEASES` entries.

Tests that need no Jenkins or Docker (bash, tar, curl, Python):

```bash
python test_pipeline.py -v
```

## Cleanup

```bash
./cleanup.sh          # stop Jenkins, keep data
./cleanup.sh --all    # also delete the volume (build history, releases), image and build output
```

## Troubleshooting

- *Pipeline builds old code*: it builds the last commit; commit and rebuild.
- *`python3 -m venv` fails*: the image must come from `jenkins/Dockerfile`; rebuild with `docker compose build`.
- *Smoke test fails with connection refused*: check `docker exec lab9-jenkins cat /var/jenkins_home/deploy/flask-app/app.log`.
- *Port 5001 in use inside the container*: only one pipeline run at a time is allowed (`disableConcurrentBuilds`).
- *`ln`/symlink problems on a filesystem without symlinks*: `deploy.sh` falls back to a directory replacement.


