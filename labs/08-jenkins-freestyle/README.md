# Lab 8: Jenkins Freestyle Job

## Scenario

You have a small shell program in a Git repository. Jenkins should fetch the
latest version and run it on demand, proving that Jenkins can pull code from
Git and execute a command. A **Freestyle project** is Jenkins's simplest job
type: fill in a form (source, build steps, triggers) and Jenkins runs it.

## Objectives

- Understand the parts of a Freestyle job: description, parameters, Source Code
  Management (SCM), triggers, build steps.
- Create the job in the UI, and also as code (a `config.xml` created through the REST API).
- Run a build, read the console output, and understand what the workspace is.
- Publish the sample code to GitHub yourself, safely, and point the job at it.

## Prerequisites

- Docker with the Compose plugin ([../../docs/prerequisites.md](../../docs/prerequisites.md)), `curl`, `git`.
- This lab brings its own Jenkins (same pattern as Lab 7, container `lab8-jenkins`).
  Stop other Jenkins containers first, or set `JENKINS_PORT`.
- Run the scripts from Git Bash or WSL on Windows.
- Only for the GitHub part: a GitHub account.

## Files

| File | Purpose |
|------|---------|
| `sample-code/hello-world.sh` | The program to build: prints `Hello, <name>!` plus build info when run by Jenkins. |
| `test_hello_world.sh` | Tests for the script (bash only). |
| `job/hello-world-config.xml.tmpl` | Freestyle job definition: a Git SCM, a `GREETEE` string parameter, an Execute shell step, bounded build history. |
| `render_job.sh` | Fills in the template (`REPO_URL`, `BRANCH`, optional `POLL_SCM_SPEC`) with correct XML/sed escaping. |
| `make_sample_repo.sh` | Builds a throwaway local git repo from `sample-code/` (in `.work/`, git-ignored). No remote, never pushes. |
| `Dockerfile`, `plugins.txt`, `init.groovy.d/`, `docker-compose.yml` | Jenkins with the Git plugin, an admin user from `.env`, and the sample repo mounted read-only at `/sample-repo`. |
| `lib.sh`, `create_job.sh` | REST helpers (auth plus CSRF crumb) and an idempotent create/update of the job. |
| `deploy.sh`, `verify.sh`, `cleanup.sh` | Bring everything up, check it with a real build, tear down. |
| `test_static.py` | Tests that render the job XML and exercise the helper scripts. |

## Part A: automated run

```bash
./deploy.sh      # sample repo + Jenkins + job (admin password generated into .env if missing)
./verify.sh      # triggers a build with GREETEE=Verifier and checks the result
```

Then open `http://localhost:8080/job/HelloWorld/`, log in with the credentials in `.env`,
and click **Build with Parameters**.

## Part B: create the job by hand (what the XML automates)

1. Dashboard > **New Item**, name `HelloWorld`, choose **Freestyle project**, OK.
2. **General**: add a description; tick *This project is parameterized* and add a
   String parameter `GREETEE` with default `Jenkins`.
3. **Source Code Management**: choose **Git**, Repository URL `file:///sample-repo`
   (or your GitHub URL), Branch Specifier `*/main`. For a private GitHub repo add
   credentials (see below); the public sample repo needs none.
4. **Build Triggers**: leave empty for manual runs. "Build Now" is a manual
   action, not a trigger. For automatic builds tick **Poll SCM** and enter a
   schedule such as `H/2 * * * *` (`POLL_SCM_SPEC` does this in the automated path).
5. **Build Steps** > *Add build step* > **Execute shell**:
   ```
   bash hello-world.sh "$GREETEE"
   ```
6. **Save**, then **Build with Parameters**.

## What happens in a build

Jenkins creates a workspace (`/var/jenkins_home/workspace/HelloWorld`), clones the
repository into it, runs the shell step there with the parameters as environment
variables, and records the exit status: 0 is SUCCESS, non-zero is FAILURE.

## Verification

`./verify.sh` checks that:

1. Jenkins is reachable and the `HelloWorld` job exists.
2. A parameterized build can be queued (HTTP 201).
3. The build finishes with `SUCCESS`.
4. The console output contains `Hello, Verifier!` and a Git checkout.
5. `hello-world.sh` exists in the job workspace.

By hand: open the build under *Build History*, then **Console Output**. Expect lines
for cloning/fetching the repository, `+ bash hello-world.sh Jenkins`, the greeting,
`job=HelloWorld build=#1`, and `Finished: SUCCESS`.

To see a change flow through: `./make_sample_repo.sh --new-commit`, then build again;
the console shows the new commit being fetched (or enable `POLL_SCM_SPEC` and wait).

## Publishing the sample code to GitHub (manual, your decision)

Nothing in this lab pushes or configures a remote. If you want the job to use GitHub:

1. Create a **public** repository named `devops-sample-code` on GitHub (empty, no README).
2. Create a **fine-grained personal access token** limited to that one repository with
   *Contents: Read and write*. Treat it like a password; never put it in a file in this repo.
3. In a separate working copy of the sample code:
   ```bash
   git clone .work/devops-sample-code ~/devops-sample-code && cd ~/devops-sample-code
   git remote set-url origin https://github.com/<your-account>/devops-sample-code.git
   git push -u origin main        # username = your account, password = the token
   ```
4. Set `REPO_URL=https://github.com/<your-account>/devops-sample-code.git` in `.env`
   and run `./create_job.sh` to update the job. Public repos need no Jenkins credentials.

(The lab's own repo-wide rule applies here too: this repository's `origin` is never changed by any script.)

## Cleanup

```bash
./cleanup.sh          # stop Jenkins, keep data
./cleanup.sh --all    # also delete the volume, image and local sample repo
```

## Troubleshooting

- *`dubious ownership` during checkout*: the image sets `safe.directory '*'` for the
  lab; if you changed the mount, check that setting.
- *Job update returns 403*: the CSRF crumb needs the same session cookie; the helper
  handles this, so check the admin password in `.env` matches the one Jenkins was
  first created with (see Lab 7: the init script does not overwrite an existing user).
- *Build is SUCCESS but no greeting*: check the Execute shell step and that the
  parameter is named `GREETEE`.

