# What is GitHub Actions

GitHub Actions is a Continuous Integration & Deployment platform provided by GitHub that can be used to deploy your code from one environment to another environment. You can create workflows and jobs to trigger the pipeline and deployment.

## Key Features

- **Event-Driven** - Workflows are triggered by events: push, pull request, schedule, manual dispatch, and 30+ more.
- **Built into GitHub** - No external CI server needed. Workflows live in your repo under `.github/workflows/`.
- **Action Marketplace** - 10,000+ reusable actions from the community. Checkout code, setup Node, deploy to AWS in one line each.
- **Multi-Platform** - Run jobs on Linux, macOS, or Windows. Mix and match in the same workflow.

## Core Concepts

```
Workflow
  Event -> Job -> Step -> Action
```

- **Workflow** - The top-level automation. A YAML file in `.github/workflows/`. One repo can have many workflows (e.g., `ci.yml`, `deploy.yml`, `release.yml`).
- **Event / Trigger** - The signal that starts the workflow. E.g., push to main, opening a PR, a cron schedule, or clicking "Run workflow" manually.
- **Job** - A set of steps that run on the same runner (virtual machine). Jobs run in parallel by default. Use `needs:` to create dependencies between jobs.
- **Step** - An individual task inside a job. Steps run sequentially. A step can be a shell command (`run:`) or a pre-built action (`uses:`).
- **Action** - A reusable, packaged unit of automation. Published to the GitHub Marketplace. E.g., `actions/checkout@v4`, `actions/setup-node@v4`.

## Workflow Anatomy

```yaml
# ① Workflow Name — displayed on the Actions tab
name: CI Pipeline

# ② Event Triggers — what starts this workflow
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# ③ Jobs — units of work running on a runner
jobs:
  build-and-test:                    # job id
    name: Build & Test
    runs-on: ubuntu-latest           # runner OS

    # ④ Steps — sequential tasks in this job
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4    # ⑤ reusable Action

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:                       # action inputs
          node-version: 20

      - name: Install Dependencies
        run: npm ci                 # shell command

      - name: Run Tests
        run: npm test

      - name: Build Application
        run: npm run build
```

**Key elements:**

- **`name`**: Human-readable label shown in the GitHub UI under the Actions tab.
- **`on`**: Defines when the workflow runs. You can specify multiple events with fine-grained filters.
- **`jobs`**: Container for all jobs. Each job gets its own fresh runner environment.

## Workflow Triggers

The `on:` keyword is your workflow's starter switch. GitHub supports 35+ event types. Here are the most important ones you'll use daily.

| Event | When it fires | Typical use |
|---|---|---|
| `push` | On any git push | Build & test on every commit |
| `pull_request` | PR opened, updated, or synced | Validate before merging |
| `workflow_dispatch` | Manual click from GitHub UI | One-off deploys, dry runs |
| `schedule` | Cron expression (UTC) | Nightly builds, daily reports |
| `release` | Release published/created | Publish to NPM, PyPI, etc. |
| `issues` | Issue opened, closed, labeled | Auto-label, triage bots |
| `workflow_call` | Called by another workflow | Reusable workflow composition |
| `repository_dispatch` | External HTTP POST | Trigger from external systems |

```yaml
on:
  schedule:
    - cron: '0 2 * * 1-5'   # 2am UTC, Mon-Fri
  workflow_dispatch:        # enables the "Run workflow" button
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: staging
        type: choice
        options: [staging, production]
```

> **Pro tip:** Use `paths:` filters with `push` and `pull_request` to only trigger workflows when specific files change. This dramatically reduces unnecessary runs.

---

# GitHub Actions Contexts

Contexts let you access information about workflow runs, runner environments, jobs, steps, and more inside expressions (`${{ }}`). Below is a breakdown of the most commonly used contexts.

---

## `github` — info about the event & repo

| Property | Example Value | Notes |
|---|---|---|
| `github.event_name` | `"push"` / `"pull_request"` / `"workflow_dispatch"` | The event that triggered the workflow |
| `github.ref` | `"refs/heads/main"` | The branch or tag ref that triggered the run |
| `github.sha` | `"abc123..."` | The commit SHA that triggered the run |
| `github.actor` | `"octocat"` | The username that triggered the run |
| `github.repository` | `"owner/repo"` | Owner and repository name |
| `github.event` | *full webhook payload object* | The complete JSON payload for the triggering event |

**Common extras (not in the screenshot, but frequently used):**

| Property | Example Value | Notes |
|---|---|---|
| `github.workflow` | `"CI"` | Name of the workflow |
| `github.run_id` | `"1658821493"` | Unique ID for this workflow run |
| `github.run_number` | `"42"` | Incrementing run number for the workflow |
| `github.job` | `"build"` | The `job_id` of the currently running job |
| `github.workspace` | `"/home/runner/work/repo/repo"` | Default working directory on the runner |
| `github.token` | *masked* | Auto-generated `GITHUB_TOKEN` for API auth |
| `github.base_ref` | `"main"` | Base branch (PR events only) |
| `github.head_ref` | `"feature/xyz"` | Head branch (PR events only) |

---

## `env` — workflow/job/step env vars

| Property | Example Value | Notes |
|---|---|---|
| `env.MY_VAR` | *value of `$MY_VAR`* | Any variable defined in `env:` blocks at workflow, job, or step level |

```yaml
env:
  MY_VAR: "hello-world"

steps:
  - run: echo "${{ env.MY_VAR }}"
```

---

## `secrets` — encrypted secrets

| Property | Example Value | Notes |
|---|---|---|
| `secrets.MY_TOKEN` | `***` (masked) | Pulled from repo/org/environment secrets; always masked in logs |

```yaml
- name: Authenticate
  run: sf org login jwt --client-id ${{ secrets.CLIENT_ID }}
```

> ⚠️ Secrets are never printed in plaintext in logs — GitHub automatically redacts them, even if you try to echo them out.

---

## `runner` — info about the runner machine

| Property | Example Value | Notes |
|---|---|---|
| `runner.os` | `"Linux"` | Also `"Windows"`, `"macOS"` |
| `runner.arch` | `"X64"` | Also `"ARM"`, `"ARM64"` |
| `runner.name` | `"GitHub Actions 2"` | Name of the runner |
| `runner.temp` | `/tmp` | Path to a temp directory scoped to the run |
| `runner.tool_cache` | *path* | Path to cached tools directory |

---

## `job` — current job status

| Property | Example Value | Notes |
|---|---|---|
| `job.status` | `"success"` / `"failure"` / `"cancelled"` | Status of the current job, useful in `if:` conditions on later steps |

```yaml
- if: ${{ job.status == 'failure' }}
  run: echo "Something upstream failed"
```

---

## `steps` — outputs from previous steps

| Property | Example Value | Notes |
|---|---|---|
| `steps.<step-id>.outputs.<output-name>` | *custom value* | Requires the step to have an `id:` and to set an output |

```yaml
steps:
  - id: my-step-id
    run: echo "my-output=hello" >> "$GITHUB_OUTPUT"

  - run: echo "${{ steps.my-step-id.outputs.my-output }}"
```

---

## Other useful contexts

| Context | Purpose |
|---|---|
| `inputs` | Values passed into a reusable workflow or `workflow_dispatch` |
| `vars` | Configuration variables (non-secret) defined at repo/org/environment level |
| `matrix` | Current combination in a `strategy.matrix` build |
| `needs` | Outputs/results from jobs listed in `needs:` |
| `strategy` | Info about the current matrix strategy (e.g. `fail-fast`) |

---

### Quick syntax reminder

- Access with double curly braces: `${{ context.property }}`
- Use in `if:`, `env:`, `run:`, and most `with:` fields
- `github.event` is a deeply nested object — drill in with dot notation matching the webhook payload (e.g. `github.event.pull_request.number`)
