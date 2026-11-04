# Jenkins setup

`Jenkinsfile` is the Jenkins port of `.github/workflows/3-ci.yml`. It installs
dependencies for the three ecosystems in this repo (npm, Go modules, RubyGems),
each in its own container, with the InvisiRisk PSE Build Application Firewall
wrapped around the install.

The GitHub workflow is still in place; nothing was removed. Delete it when you
are satisfied the Jenkins job is the real one.

---

## 1. Prerequisites

### Plugins

| Plugin | Used for |
| --- | --- |
| Pipeline (`workflow-aggregator`) | the `pipeline { }` block |
| Pipeline: Declarative | `parallel`, `when`, per-stage `agent` |
| Docker Pipeline (`docker-workflow`) | `agent { docker { ... } }` |
| Credentials Binding (`credentials-binding`) | `withCredentials` |
| Git (`git`) | `checkout scm` |
| Timestamper (`timestamper`) | the `timestamps()` option |

If you would rather not install Timestamper, delete the `timestamps()` line
from the `options` block.

### Agent

The build needs a Jenkins agent that:

- has a working Docker daemon and can pull `node:20`, `golang:1.22`, `ruby:3.3`
- can reach `https://app.dev.invisirisk.com` (the `IR_API_URL` value)
- is allowed to run containers with `--cap-add=NET_ADMIN`

The pipeline does not pin an agent label, so it runs on any available agent. If
only some of your agents have Docker, add a label to each `docker` block:

```groovy
agent {
    docker {
        image 'node:20'
        args  '-u root --cap-add=NET_ADMIN'
        label 'docker'
    }
}
```

> If you add a label that no agent has, the build queues forever instead of
> failing. Check **Manage Jenkins → Nodes** for the right label first.

### Why the containers run as root with `NET_ADMIN`

`args '-u root --cap-add=NET_ADMIN'` is not optional:

- **`-u root`** — the PSE bootstrap installs a CA certificate into the system
  trust store and drops the `pse-data-collector` binary into `/usr/local/bin`.
  By default the Docker Pipeline plugin runs containers as the Jenkins user,
  which cannot write to either location.
- **`--cap-add=NET_ADMIN`** — in `native` mode PSE redirects the container's
  egress with iptables rules, which needs that capability.

Because the container runs as root, everything it writes into the bind-mounted
workspace becomes root-owned. The pipeline records the workspace owner before
the build and `chown`s the files back afterwards, so the next build can still
update the workspace as the Jenkins user. That is what the
"record workspace owner" and "restore workspace ownership" steps do.

---

## 2. Create the InvisiRisk credential

In GitHub Actions the action could authenticate with OIDC, falling back to a
token. Jenkins has no GitHub OIDC identity, so **a token is required**.

1. Get an app token from your InvisiRisk tenant (Settings → API tokens, or ask
   your InvisiRisk contact). It is the same value you would pass as the
   `app_token` input.
2. In Jenkins: **Manage Jenkins → Credentials → (global) → Add Credentials**
3. Fill in:
   - **Kind**: `Secret text`
   - **Secret**: the token
   - **ID**: `invisirisk-app-token`
   - **Description**: `InvisiRisk PSE app token`

The ID must match `IR_TOKEN_CREDENTIAL_ID` in the `environment` block. To use a
different ID (for example a per-team credential), change it there.

Jenkins masks the value in console output, and the pipeline's shell scripts are
written with single-quoted Groovy strings so the token is never interpolated
into the script text — only expanded by the shell at run time.

---

## 3. Create the job

### Option A — Pipeline from SCM (simplest)

1. **New Item** → name it → **Pipeline** → OK
2. Under **Pipeline**:
   - **Definition**: `Pipeline script from SCM`
   - **SCM**: `Git`
   - **Repository URL**: `https://github.com/Pranesh-Shrestha/3-ecosystems.git`
   - **Credentials**: a GitHub credential if the repo is private
   - **Branch**: `*/main`
   - **Script Path**: `Jenkinsfile`
3. Save.

### Option B — Multibranch Pipeline

**New Item → Multibranch Pipeline**, add the Git source, leave the Jenkinsfile
path as `Jenkinsfile`. Jenkins then discovers branches and PRs automatically,
which is the closest equivalent to how Actions behaves per-branch.

> Both options rely on `checkout scm`. If you instead paste the script into the
> job's **Pipeline script** box, `scm` is undefined — replace each
> `checkout scm` with an explicit step, for example:
> ```groovy
> git branch: 'main', url: 'https://github.com/Pranesh-Shrestha/3-ecosystems.git'
> ```

### First run

The parameters only appear after one build, because Jenkins learns them by
running the Jenkinsfile once. Click **Build Now**; from then on the job shows
**Build with Parameters**.

---

## 4. Running it

**Build with Parameters** exposes:

| Parameter | Default | Effect |
| --- | --- | --- |
| `SKIP_JAVASCRIPT` | false | skip the `npm install` stage |
| `SKIP_GO` | false | skip the `go mod download` stage |
| `SKIP_RUBY` | false | skip the `bundle install` stage |
| `PSE_DEBUG` | false | verbose PSE logging (`DEBUG=true`) |

The three ecosystem stages run in parallel, as the three GitHub jobs did.

---

## 5. How the GitHub workflow maps onto Jenkins

| GitHub Actions | Jenkins | Note |
| --- | --- | --- |
| `on: workflow_dispatch` | no `triggers` block | manual / API-triggered only |
| `inputs.SKIP_*` (boolean) | `booleanParam` | same names |
| three `jobs:` | `parallel` stages | still concurrent |
| `runs-on: ubuntu-latest` + `container: node:20` | `agent { docker { image 'node:20' } }` | per stage |
| `if: inputs.SKIP_X != true` | `when { expression { !params.SKIP_X } }` | `beforeAgent true` avoids starting a container just to skip |
| `uses: actions/checkout@v4` | `checkout scm` | runs after PSE setup, as before |
| `uses: invisirisk/pse-action@latest` | `withPse { }` helper | see below |
| action's `post:` step | cleanup inside `withPse` | runs even when the build fails |
| `permissions:` / `id-token: write` | *(dropped)* | GitHub-only; replaced by the token credential |
| `run: npm install ...` | `sh 'npm install ...'` | unchanged, including the `curl` egress probes |

### What replaced the PSE action

`invisirisk/pse-action` is a Node wrapper around two operations, and the
Jenkinsfile performs both directly:

**Setup** — request the bootstrap script and run it:

```
GET {IR_API_URL}/ingestionapi/v1/pse/bootstrap?mode=native&runner=jenkins
                &job_identifier=...&job_label=...
Header: x-api-key: <token>
```

The response is a shell script that installs the PSE interceptor and the
`pse-data-collector` binary. Unlike the action, which pipes `curl` straight
into `bash`, the pipeline saves the response first and checks the HTTP status,
so a 401 or 403 produces a clear message instead of bash trying to execute an
error page.

**Cleanup** — `pse-data-collector cleanup`, which finalises the scan. A
non-zero exit means a blocking policy violation or a failed `/end` call, and it
fails the stage. If the build command and the cleanup both fail, the original
build failure is reported as the cause.

Environment variables passed through, matching the action: `IR_URL`,
`IR_TOKEN`, `MODE`, `RUNNER`, `PSE_IMAGE_TAG`, `DEBUG`, `PSE_JOB_LABEL`,
`PSE_JOB_IDENTIFIER`, `SEND_JOB_STATUS`.

`PSE_JOB_IDENTIFIER` is `"<JOB_NAME> <stage>"`, which stays stable from build
to build so InvisiRisk can compare the same stage over time. `PSE_JOB_LABEL` is
the readable stage name (`build-javascript`, `build-go`, `build-ruby`), keeping
the names the GitHub jobs used.

`SEND_JOB_STATUS=false` because the action's job-status collection calls the
GitHub API for workflow status, which has no Jenkins equivalent.

---

## 6. Verifying it worked

A healthy build log shows, per stage:

1. `[BAF] Requesting PSE bootstrap (mode=native, runner=jenkins, job=build-go)`
2. `[BAF] Running PSE bootstrap in native mode`
3. `[BAF] PSE setup complete`
4. the install output (`npm install`, `go mod download -x`, `bundle install`)
5. `[BAF] Running PSE cleanup`

Then confirm the real success criterion outside Jenkins: **the three scans
appear in the InvisiRisk dashboard** at `https://app.dev.invisirisk.com`, with
the dependencies from `package.json`, `go.mod` and `Gemfile`. A green build on
its own only proves the commands exited zero — it does not prove the traffic was
inspected.

---

## 7. Troubleshooting

**`IR_TOKEN is empty - check the Jenkins credential`**
The credential ID does not exist or is not a *Secret text* credential. Check it
matches `IR_TOKEN_CREDENTIAL_ID`.

**`Unauthorized (401)` / `Forbidden (403)`**
The token is not valid for `IR_API_URL`, or is not authorised for this project.
Confirm the token belongs to the same tenant as the URL — `app.dev.invisirisk.com`
is a dev tenant, and a production token will not work against it.

**`Bootstrap rejected the request (400) ... may not accept runner='jenkins'`**
See the caveat in section 8 — try `PSE_RUNNER = 'github'`.

**`pse-data-collector not found on PATH`**
The bootstrap did not finish installing. Re-run with `PSE_DEBUG` checked and
read the bootstrap output.

**`Permission denied` writing to the workspace, or git checkout fails on the
second build**
The ownership restore did not run (for example the build was aborted hard).
Wipe the workspace: job → **Workspace → Wipe Out Current Workspace**.

**Certificate errors from npm / go / bundler**
PSE terminates TLS, so its CA must be trusted. This is what `-u root` enables.
If you removed it, put it back. Go additionally honours `GOFLAGS`/`SSL_CERT_FILE`
rather than the system store in some setups.

**The build hangs in the queue**
A `label` was added that no agent matches, or no agent has Docker.

**`Scripts not permitted to use method ...`**
The Jenkinsfile is being run through the Groovy sandbox from a non-SCM source.
Running it as *Pipeline script from SCM* avoids this; otherwise approve the call
under **Manage Jenkins → In-process Script Approval**.

---

## 8. Things to confirm before relying on this

Two details could not be verified without a live InvisiRisk tenant and a
Jenkins agent, so check them on your first run.

**The `runner` value.** The action hardcodes `runner=github`, and the value is
overridable in its source, but the set of values the bootstrap API accepts is
not public. The pipeline sends `runner=jenkins`. If the bootstrap returns 400,
set `PSE_RUNNER = 'github'` in the `environment` block and raise it with
InvisiRisk support — the only thing it changes is how the scan is labelled on
their side.

**Whether `native` mode needs environment variables carried between steps.**
In GitHub Actions the action can persist variables across steps via
`GITHUB_ENV`. Jenkins runs each `sh` step in a fresh shell, so anything the
bootstrap merely *exports* would be lost after the setup step. This pipeline
assumes `native` mode intercepts transparently with iptables (its legacy name
was `docker-intercept`), which survives across steps and needs no variable
passing — hence `NET_ADMIN`.

If your installs show traffic bypassing PSE, that assumption is wrong for your
tenant. The fix is to run setup and the build commands in a single `sh` call, so
the exported variables still apply, for example:

```groovy
sh label: 'install under PSE', script: '''#!/usr/bin/env bash
set -euo pipefail
source /path/to/pse/env   # whatever the bootstrap writes
npm install --no-audit --no-fund
'''
```

Ask InvisiRisk support which applies to the Jenkins runner.

### Sidecar mode

Setting `PSE_MODE = 'sidecar'` makes the bootstrap run the
`invisirisk/pse-proxy` container (tag from `PSE_IMAGE_TAG`) instead of the
in-container binary. That needs a Docker socket **and** a `docker` CLI inside
the build container, and `node:20`, `golang:1.22` and `ruby:3.3` ship neither.
Stay on `native` unless InvisiRisk tells you otherwise; if you do switch, mount
the socket and add the CLI:

```groovy
args '-u root --cap-add=NET_ADMIN -v /var/run/docker.sock:/var/run/docker.sock'
```

---

## 9. Kubernetes agents instead of Docker

On a Kubernetes-backed Jenkins, swap each `agent { docker { ... } }` for a
`kubernetes` agent. The privilege requirements carry over — the container needs
to run as root and hold `NET_ADMIN`:

```groovy
agent {
    kubernetes {
        yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: build
      image: node:20
      command: ['cat']
      tty: true
      securityContext:
        runAsUser: 0
        capabilities:
          add: ['NET_ADMIN']
'''
        defaultContainer 'build'
    }
}
```

A Pod Security Standard of `restricted` will reject `NET_ADMIN`; the namespace
needs to allow it.

---

## Validation already done

- All nine embedded shell scripts pass `bash -n`.
- The bootstrap URL builder was executed directly and percent-encodes correctly
  (`3-ecosystems/main build-javascript` → `3-ecosystems%2Fmain%20build-javascript`).
- The `set -euo pipefail` paths were exercised to confirm the optional
  `job_label` / `job_identifier` branches do not abort the script.

Not done, because this machine has no Java, Groovy or Docker: the Jenkins
declarative linter. Run it against your controller before trusting the first
build:

```bash
curl -X POST -F "jenkinsfile=<Jenkinsfile" https://<jenkins>/pipeline-model-converter/validate
```

---

Sources: interface and behaviour taken from the
[invisirisk/pse-action](https://github.com/invisirisk/pse-action) repository
(`action.yml`, `src/index.js`, `src/post.js`, `src/utils.js`) and the
[Marketplace listing](https://github.com/marketplace/actions/invisirisk-pse-security-proxy).
Content was rephrased for compliance with licensing restrictions.
