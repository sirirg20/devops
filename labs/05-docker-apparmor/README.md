# Lab 5: Docker Security and AppArmor

## Scenario

A Flask app runs in a container. Even if it were compromised, it should not be
able to read account files, write to variable-data directories, start system
binaries or perform administrative kernel operations. An AppArmor profile
attached at `docker run` time enforces this, and the Docker Python SDK is used
to apply the profile and test it.

## Objectives

- Write an AppArmor profile for a container and understand why deny rules win.
- Load the profile on the host and attach it with `--security-opt apparmor=...`.
- Confirm the profile is applied by inspecting the container.
- Prove that actions are blocked **because of** the profile by comparing a
  confined container with an unconfined control.

## Environment requirements (read first)

AppArmor is a Linux kernel feature. This lab **cannot run on native Windows or
macOS**. You need all of:

| Requirement | How to check |
|-------------|--------------|
| Linux host, VM, or WSL 2 with an AppArmor-enabled kernel | `cat /sys/module/apparmor/parameters/enabled` prints `Y` |
| AppArmor userspace tools (`apparmor_parser`, optionally `aa-status`) | `command -v apparmor_parser` |
| Docker Engine on that Linux system, daemon built with AppArmor support | `docker info` lists `apparmor` under Security Options |
| `sudo` rights (loading a profile is privileged) | |
| Python 3.10+ and the Docker SDK in a virtualenv | `pip install -r requirements-sdk.txt` |

Docker Desktop's built-in VM does not let you load custom host profiles; use a
real Linux machine or a VM where you control the kernel. The scripts check these
requirements and stop with a message; they never install anything.

## Files

| File | Purpose |
|------|---------|
| `app.py`, `requirements.txt`, `Dockerfile`, `.dockerignore` | Flask app (`/`, `/health`) in a non-root image `lab5-flask:1.0`. |
| `profile/lab5-restricted` | The AppArmor profile (commented). |
| `lab_common.py` | Shared Docker SDK helpers. |
| `apply_profile.py` | Builds the image, runs it under the profile, inspects `SecurityOpt` and `AppArmorProfile`, checks `GET /`. |
| `check_restrictions.py` | Runs probes in a confined container and an unconfined control and prints a comparison table. |
| `test_app.py` | Unit tests for the Flask app. |
| `test_profile_static.py` | Text-level checks of the profile (no kernel needed). |
| `requirements-sdk.txt` | `docker` SDK pin for the host-side scripts. |
| `deploy.sh` / `cleanup.sh` | Load/unload the profile and run or remove everything. |

## What the profile does

| Rule | Effect |
|------|--------|
| `network inet/inet6 stream`, `inet dgram`, `unix` | Web server networking allowed |
| `file,` | Broad file access as the base (as in Docker's default profile) |
| `deny /etc/passwd r`, `/etc/shadow*`, `/etc/sudoers*` | Account files cannot be read |
| `deny /var/** wklx` | No writes to variable data |
| `deny /bin/** /sbin/** /usr/bin/** /usr/sbin/** mx` | System binaries cannot be executed |
| `capability net_bind_service` allowed; `sys_admin`, `sys_module`, `sys_ptrace` denied | No administrative operations |
| `deny mount`, `deny umount`, writes to `/proc/sys` and `/sys` denied | No mounting or kernel tuning |

Design notes:

- Denying all of `/etc/**` or `/usr/bin/**` would likely stop the app from
  starting (the loader and interpreter need parts of them), so the profile
  denies the specific sensitive files instead. The Python interpreter lives in
  `/usr/local/bin`, which is left alone.
- Because denies override allows, the permissive `file,` base does not weaken
  the deny rules.

## Steps

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements-sdk.txt
./deploy.sh
```

`deploy.sh` syntax-checks the profile (`apparmor_parser -Q -K`), loads it
(`apparmor_parser -r`), runs `apply_profile.py`, then `check_restrictions.py`.

Manual equivalent:

```bash
sudo apparmor_parser -r profile/lab5-restricted
docker build -t lab5-flask:1.0 .
docker run -d --name lab5-confined --security-opt apparmor=lab5-restricted -p 5055:5000 lab5-flask:1.0
docker inspect -f '{{.AppArmorProfile}} {{.HostConfig.SecurityOpt}}' lab5-confined
curl localhost:5055/
docker exec lab5-confined python -c "open('/etc/passwd').read()"   # Permission denied
docker exec lab5-confined /bin/sh -c true                          # exec denied
```

## Expected results

`check_restrictions.py` prints one row per probe; each must PASS:

| Probe | Confined | Unconfined control |
|-------|----------|--------------------|
| `python -c "print('ok')"` | succeeds | succeeds |
| read `/etc/passwd` | blocked (non-zero exit) | succeeds |
| write `/var/tmp/probe` | blocked | succeeds |
| write `/tmp/probe` | succeeds | succeeds |
| exec `/bin/sh` | blocked (exec denied, often exit 126) | succeeds |
| exec `/usr/bin/env` | blocked | succeeds |

Exit codes other than zero count as blocked; the exact code varies with how
Docker and the runtime report the denial. Denials also appear in the host log
(`dmesg | grep -i apparmor` or `journalctl -k | grep -i DENIED`).

## Local checks that need no Linux or Docker

```bash
python test_profile_static.py -v      # profile text sanity, stdlib only
python -m unittest -v test_app        # needs Flask (pip install -r requirements.txt)
```

## Cleanup

```bash
./cleanup.sh     # removes lab containers/image and unloads the profile
```

