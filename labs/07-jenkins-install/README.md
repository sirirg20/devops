# Lab 7: Jenkins Installation

## Background

Continuous Integration (CI) means integrating code changes frequently and
having every change built and tested automatically, so problems surface within
minutes instead of at release time. Jenkins is an automation server that runs
those builds, tests and deployments. A Jenkins installation has a **controller**
(the web UI and scheduler; older documents call it "master") and optional
**agents** (machines that run builds). This lab installs a controller with
Docker.

## Objectives

- Run Jenkins in a container with its data on a persistent volume.
- Complete first-time setup two ways: interactively (setup wizard) and
  unattended (plugins and admin user defined in files).
- Verify the installation, including that data survives container re-creation.

## Prerequisites

- Docker; the unattended mode also needs the Compose plugin
  ([../../docs/prerequisites.md](../../docs/prerequisites.md)).
- Internet access (image and plugin downloads), `curl`, ~1 GB disk, port 8080 free
  (or set `JENKINS_PORT`).
- Run the scripts from Git Bash or WSL on Windows.

## Files

| File | Purpose |
|------|---------|
| `deploy.sh manual\|auto` | Start Jenkins in either mode (default `auto`). |
| `Dockerfile` | `jenkins/jenkins:lts-jdk17` plus preinstalled plugins and the init script; wizard disabled. |
| `plugins.txt` | Plugins to preinstall (Pipeline, Git, Credentials Binding, Timestamper, Workspace Cleanup). |
| `init.groovy.d/01-admin-user.groovy` | Creates the admin account from env vars; denies anonymous access; refuses to start unsecured. |
| `docker-compose.yml` | The unattended-mode service, volume and healthcheck. |
| `.env.example` | Template for admin user and password (`.env` is git-ignored). |
| `verify.sh [manual\|auto] [--persistence]` | Automated checks. |
| `cleanup.sh [--volumes]` | Remove containers; optionally the data volumes and image. |
| `test_static.py` | Stdlib checks of the files above. |

## Mode 1: manual (setup wizard)

```bash
./deploy.sh manual
docker exec lab7-jenkins-manual cat /var/jenkins_home/secrets/initialAdminPassword
```

1. Open `http://localhost:8080` and paste the password on the **Unlock Jenkins** page.
2. **Customize Jenkins**: choose *Install suggested plugins* and wait for them to finish.
3. **Create First Admin User**: fill in a username, a strong password and an email.
4. **Instance Configuration**: accept the URL; click *Start using Jenkins*.
5. The dashboard (landing page) appears.

The generated unlock password is specific to your container; never commit or reuse one
you saw in a tutorial.

## Mode 2: unattended

```bash
./deploy.sh auto
```

This builds the image (plugins are downloaded at build time), creates `.env` with
a random admin password if you have not made one, and starts Jenkins. Log in at
`http://localhost:8080` as the user in `.env`. No wizard appears.

How it works:

- `-Djenkins.install.runSetupWizard=false` skips the wizard.
- `jenkins-plugin-cli` installs the plugins in `plugins.txt` into the image.
- The init script runs on startup and creates the admin user from
  `JENKINS_ADMIN_USER` / `JENKINS_ADMIN_PASSWORD`. Because the wizard is off,
  this script is what keeps Jenkins from starting open to everyone, so it
  throws if the password is missing, and Compose also refuses to start without it.

## Design notes

- **Persistence.** Both modes mount a named volume at `/var/jenkins_home`. Without a
  volume, removing the container deletes all jobs and configuration.
- **Ports.** `8080` is the web UI. `50000` is for inbound agent connections;
  no agents exist in this lab, so it is not published. It is bound to loopback
  (`127.0.0.1`) so the UI is reachable only from your machine.
- **Image tag.** `lts-jdk17` tracks the current LTS line. Pin a version number
  (and plugin versions) if you need identical rebuilds.
- **Changing the password.** The init script only creates the account if it does
  not exist yet. Changing `.env` later does not change an existing password; use
  *Manage Jenkins > Users* or remove the volume.

## Verification

```bash
./verify.sh auto                 # or: ./verify.sh manual
./verify.sh auto --persistence   # also recreates the container and checks data survived
```

Checks performed in `auto` mode:

1. Container is running with port 8080 mapped.
2. `GET /login` returns 200.
3. The `X-Jenkins` response header is present (shows the version).
4. Anonymous `/api/json` is refused (401/403).
5. The admin credentials work against `/api/json`.
6. The `workflow-aggregator`, `git` and `credentials-binding` plugins are installed.
7. (`--persistence`) a marker file in the volume survives `compose down` + `up`.

Manual spot checks: `docker ps` shows the container `Up` (and `healthy` in auto mode);
the dashboard loads; *Manage Jenkins > Plugins > Installed plugins* lists Pipeline and Git.

## Cleanup

```bash
./cleanup.sh              # remove containers, keep data
./cleanup.sh --volumes    # also delete data volumes and the image
```


