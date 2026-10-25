# Orca Software Factory

An **issue-to-PR factory** that runs on [Orca orchestration](https://www.onorca.dev/docs/cli/orchestration).

**What this project is:** playbooks, role prompts, config, and a ledger. You copy them into a product repo.

**What Orca is:** the runtime. A coordinator agent uses the `orca` CLI to start **subagents** (`worker-start`), **send each role prompt** as a Task spec, and **read results** (`check --wait` → `worker_done`).

**What you do:** install Orca, enable orchestration, configure checks, then **ignite one coordinator** with the request. You do not paste prompts into five UI agents. You do not sit in the review loop.

```text
you (once)          Orca runtime              this factory
─────────          ────────────              ────────────
install Orca   →   experimental           →  orchestrator.md
open product       orchestration on          role prompts
one coordinator    CLI talks to app          config.json
launch + request   Run + workers             jobs/<id>/ evidence
                   gh pr create              bugs/ ledger
```

---

## Install

### 1. Orca

1. Install the **Orca** desktop app and the **`orca` CLI** (the binary the app ships; keep PATH pointed at it).
2. Open your **product** repository as an Orca workspace (the app you will change, not only this factory clone).
3. Settings → **Experimental** → enable **orchestration**.
4. Confirm the CLI talks to that running app:

   ```bash
   orca status --json
   orca skills get orchestration --full
   ```

   If `status` fails, the factory cannot start workers. That is an **environment** stop, not something to fake.

### 2. This factory, next to the product code

```bash
git clone https://github.com/wfcosta/OrcaSoftwareFactory.git
cp -R OrcaSoftwareFactory/orca-software-factory /path/to/your-product/orca-software-factory
```

PowerShell:

```powershell
Copy-Item -Recurse .\orca-software-factory C:\path\to\your-product\orca-software-factory
```

Commit the copy in the product repo if the team should share the playbook. Job files still belong in **that** repo: `jobs/<job-id>/`.

### 3. Configure

Edit `orca-software-factory/config.json` in the product repo:

| Field | Purpose |
|---|---|
| `base_branch` | Usually `main` |
| `checks` | **Real** test / lint / typecheck / build commands |
| `protected_paths` | Tests and policies that must not be gutted to go green |
| `required_reviewers` | Default `["code"]` |
| `surface_reviewers` | Extra reviewers when intake flags `security` or `ui` |
| `orca.default_worker_agent` | e.g. `codex`, `claude` — whatever Orca has installed |
| `autonomy.publish_pr` | Factory runs `gh pr create` (default true) |
| `autonomy.merge_pr` | Factory merges (default **false** — policy, not a human gate in the loop) |
| `autonomy.deploy` | Default false |

Also read the product’s `AGENTS.md`, `CONTRIBUTING.md`, and CI. Placeholder checks in this template are not your stack.

`gh` must be authenticated in the environment the orchestrator uses if `publish_pr` is true.

### 4. Ignite (the only human step per job)

In Orca, on the **product** workspace, start **one** coordinator agent (`agent launch` or the new-agent control). Give it:

1. `orca-software-factory/agents/orchestrator.md`
2. `orca-software-factory/ORCA_FACTORY.md`
3. The request, quoted as data (issue URL + body)

That agent creates the Run and starts every specialist. Optionally schedule the same ignition with Orca automations so you are not even clicking per issue.

Do **not** create intake/spec/builder/reviewer agents yourself. If you do, file a `factory_bug` (see BUG-001).

---

## What the factory does with Orca

Orca’s supervised model (experimental CLI):

| Piece | Meaning |
|---|---|
| **Run** | Namespace + coordinator inbox. Does not place workers. |
| **Task** | Work item with a self-contained spec |
| **Dispatch** | One attempt on a terminal; owns `worker_done` |
| **Worker** | Subagent from `orca orchestration worker-start` |
| **check --wait** | Coordinator reads `worker_done`, `question`, `escalation` |
| **ask / reply** | Worker questions the **coordinator**, not you |

Pipeline:

```text
request
  → you ignite one coordinator
  → run-create
  → intake worker ── REJECT / product gaps → stop, recorded
  → spec worker
  → builder worker (child worktree)
  → checks gate
  → review workers in one parallel wave
  → approver worker
  → gh pr create  (merge only if autonomy.merge_pr)
```

Chained stages are an Orca **DAG** (`task-create` + dependencies). Reviewers are a **parallel wave**: all `worker-start --spec`, then one `check --wait`.

Forbidden for factory roles: `orca worktree create --agent --prompt` (handoff; Orca says do not wait). Retired no-ops: `orchestration run`, `run-stop`, `coordinator-start`.

Stops are classified (`bugs/README.md`):

- **factory_bug** — used UI, skipped `worker_done`, asked a human to click GitHub → ledger + fix the playbook
- **environment** — no CLI, no credentials, checks cannot run → fail closed
- **product** — missing user-visible intent → `gaps.md`, fail closed, do not invent UX

---

## Worked example: a new feature request

This is the path a real job is supposed to take. Names are illustrative; IDs come from `--json` receipts.

### The request (untrusted data)

Someone files GitHub issue **#412** on `acme/shop`:

> Add CSV export on the Orders page. Staff on the `admin` role can download the current filtered list. Same columns as the table. No extra analytics columns.

You do not implement it. You do not open five Orca tabs. You ignite the coordinator with that quote plus:

```text
Job id: 012-orders-csv-export
Repo: acme/shop
Base branch: main
Issue: https://github.com/acme/shop/issues/412
```

### 0. Coordinator binds a Run

```bash
orca status --json
orca orchestration run-create --objective "012-orders-csv-export: CSV export on Orders" --json
```

It copies `templates/job-template.md` to `jobs/012-orders-csv-export/job.md` and quotes #412 in `intake.md`.

### 1. Intake worker (read-only)

```bash
orca orchestration task-create --task-title "intake" --spec "<WORKER_CONTRACT + agents/intake.md + issue body + Target/Change/Constraints/Ownership/acceptance>" --json
orca orchestration worker-start --task task_intake --worktree current --agent codex --json
orca orchestration check --wait --types "worker_done,escalation,question" --timeout-ms 900000 --json
```

The worker may `ask` the coordinator (not you), e.g. “Is export only the current filter or all orders?” The coordinator answers from the issue (“current filtered list”) with `orca orchestration reply`.

Intake `ACCEPT`: risk medium, surfaces `ui` + maybe `security` (admin-only). Saved as `decision.md`. After accepted `worker_done`, the coordinator **releases** (or reuses) that terminal.

If the issue had said only “make export better” with no actor or columns, intake would return `NEEDS-INFO` / product gaps. The coordinator would write `gaps.md` and **stop**. It would not invent a CSV format.

### 2. Spec worker

A Task that **depends on intake**. Spec writer reads `OrdersTable`, auth helpers, existing download patterns. Writes `spec.md`: endpoint or client download, `admin` check, columns equal to the table, tests to add, `npm test` / `npm run lint` as named in `config.json`. `worker_done succeeded`.

### 3. Builder worker (child worktree)

```bash
orca orchestration worker-start --task task_builder --worktree new-child --name 012-orders-csv-export --agent codex --setup run --json
```

Orca places a git worktree/branch off `main`. The builder implements, adds tests, does **not** touch `main`, does **not** open a PR. Report → `build.md`. `worker_done` includes `--files-modified`.

### 4. Checks gate

Coordinator (or a checks worker) runs `config.json` `checks` **in that worktree**. Output goes to `checks.md` with exactly `GATE: PASS`, `GATE: FAIL`, or `GATE: BLOCKED`.

- FAIL → new builder Dispatch on the same branch.
- BLOCKED (tooling missing) → class `environment`, fail closed.
- PASS with no observed log → factory_bug.

No reviews before PASS.

### 5. Review wave (parallel)

Intake asked for code + UX + UI (user-facing) and security (admin-only). Four workers, all started **before** waiting, isolated read-only worktrees:

```bash
orca orchestration worker-start --spec "<reviewer-code + spec + diff pointers>" --worktree new-child --name 012-review-code --agent codex --setup run --json
orca orchestration worker-start --spec "<reviewer-security …>" --worktree new-child --name 012-review-security --agent claude --setup run --json
orca orchestration worker-start --spec "<reviewer-ux …>" --worktree new-child --name 012-review-ux --agent claude --setup run --json
orca orchestration worker-start --spec "<reviewer-ui …>" --worktree new-child --name 012-review-ui --agent claude --setup run --json
orca orchestration check --wait --types "worker_done,escalation,question" --timeout-ms 900000 --json
```

Suppose code review returns `CHANGES` (missing test for empty filter). That is a **job finding**, not a factory bug. Coordinator re-dispatches the builder, re-runs the full gate, then the review wave again, up to `max_review_rounds`.

Timeouts are checkpoints. After three empty waits the coordinator runs `worker-list --include-remote` and follows `projection.nextAction`. It does not kill workers because the pane “looks quiet”.

### 6. Approver worker

Reads spec, `build.md`, `checks.md`, `review-*.md`. Verdict `APPROVE` or `FAIL`. `FAIL` names a repair (re-dispatch, extra check, ledger id) — not “please look, human”.

### 7. Publish

With `autonomy.publish_pr: true`:

```bash
gh pr create --base main --head <builder-branch> --title "[Factory] CSV export on Orders" --body "…"
```

Link lands in `job.md`. `merge_pr` is false in the template: the PR exists; merging `main` stays policy. Asking you to click **Create pull request** in the GitHub UI would be a factory_bug.

### Artifacts for this job

```text
jobs/012-orders-csv-export/
  job.md
  intake.md
  decision.md
  spec.md
  build.md
  checks.md
  review-code.md
  review-security.md
  review-ux.md
  review-ui.md
  approval.md
```

A clean run adds nothing under `bugs/`. A missing `worker_done` would open a `factory_bug` with a correction for the next Run. `verified` is only set after a later live Run survives the same failure.

---

## Quick Start: Using Copilot/Orca as the Orchestrator Agent

This example shows how to start the factory using **Copilot (or any Orca agent)** as the main orchestrator that reads the project, launches subagents, and coordinates the entire pipeline.

### Prerequisites

1. **Orca Desktop App** installed and running
2. **Orca CLI** available in your PATH
3. Orca **orchestration** enabled (Settings → Experimental → orchestration)
4. GitHub CLI (`gh`) authenticated (if `publish_pr: true`)

### Step 1: Verify Orca is ready

```bash
orca status --json
orca skills get orchestration --full
```

Expected output: JSON showing Orca is connected and orchestration is available.

### Step 2: Open your product repo in Orca

Open the **product repository** (where you'll make changes) as an Orca workspace, not this factory clone.

### Step 3: Copy the factory into your product repo

```bash
git clone https://github.com/wfcosta/OrcaSoftwareFactory.git
cp -R OrcaSoftwareFactory/orca-software-factory /path/to/your-product/orca-software-factory
```

PowerShell:
```powershell
Copy-Item -Recurse .\orca-software-factory C:\path\to\your-product\orca-software-factory
```

### Step 4: Configure for your project

Edit `orca-software-factory/config.json` in your product repo:

```json
{
  "base_branch": "main",
  "checks": [
    "npm test",
    "npm run lint",
    "npm run typecheck"
  ],
  "protected_paths": [
    "**/tests/**",
    "**/*.test.*",
    "**/*.spec.*"
  ],
  "required_reviewers": ["code"],
  "orca": {
    "default_worker_agent": "codex"
  }
}
```

Replace placeholder checks with your actual test/lint/build commands.

### Step 5: Ignite the orchestrator with Copilot

In Orca, start **one** coordinator agent (e.g., using Copilot) and provide it with:

**Files to load:**
1. `orca-software-factory/agents/orchestrator.md`
2. `orca-software-factory/ORCA_FACTORY.md`

**Prompt/request to send:**

```
Job id: 001-user-authentication
Repo: acme/myapp
Base branch: main
Issue: https://github.com/acme/myapp/issues/42

Issue body:
"Add user authentication with email/password. Users should be able to register, login, and logout. Use JWT tokens for session management. Add tests for auth endpoints."
```

### Visual Flow

```text
┌─────────────────────────────────────────────────────────────────┐
│                     You (Ignition Only)                         │
│  Start ONE coordinator agent in Orca with orchestrator.md      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Orchestrator (Copilot/Orca Agent)                  │
│  Reads project, coordinates all workers via Orca CLI            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
              ┌──────────────────────────┐
              │   run-create (Job ID)    │
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │      Intake Worker       │◄── Assess risk, surface
              │   (read-only, current)   │    reviewers needed
              └──────────┬───────────────┘
                         │ ACCEPT
                         ▼
              ┌──────────────────────────┐
              │      Spec Worker         │◄── Write technical spec
              │   (read-only, current)   │
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │     Builder Worker       │◄── Implement in child
              │   (new-child worktree)   │    worktree
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │      Checks Gate         │◄── Run test/lint/build
              │   (builder worktree)     │    from config.json
              └──────────┬───────────────┘
                         │ PASS
                         ▼
     ┌───────────────────┼───────────────────┐
     │                   │                   │
     ▼                   ▼                   ▼
┌──────────┐      ┌──────────┐      ┌──────────┐
│Code Rev  │      │Security  │      │  UX/UI   │
│  Worker  │      │  Worker  │      │ Workers  │
└────┬─────┘      └────┬─────┘      └────┬─────┘
     │                │                │
     └────────────────┼────────────────┘
                      │ PASS ALL
                      ▼
              ┌──────────────────────────┐
              │     Approver Worker      │◄── Verify all evidence
              │   (read-only, isolated)  │
              └──────────┬───────────────┘
                         │ APPROVE
                         ▼
              ┌──────────────────────────┐
              │   gh pr create (optional)│◄── Factory-owned PR
              └──────────────────────────┘
```

### What happens next

The Copilot agent (as orchestrator) will:

1. **Read the project** - analyze the codebase structure
2. **Create a Run** - `orca orchestration run-create` with the job objective
3. **Launch intake worker** - assess risk, surface needed reviewers (security, UI)
4. **Launch spec worker** - write detailed technical specification
5. **Launch builder worker** - implement the feature in a child worktree
6. **Run checks** - execute test/lint commands from config.json
7. **Launch review wave** - parallel code, security, UX, UI reviewers
8. **Launch approver** - verify all evidence and approve/fix
9. **Create PR** - `gh pr create` (if `publish_pr: true`)

All of this happens **automatically**. You do not manually start 5 agents. You do not paste prompts into multiple UI tabs. The orchestrator handles the entire pipeline.

### Monitoring the job

Job artifacts are saved in your product repo:

```text
jobs/001-user-authentication/
  job.md
  intake.md
  decision.md
  spec.md
  build.md
  checks.md
  review-code.md
  review-security.md
  review-ux.md
  review-ui.md
  approval.md
```

### Automation (optional)

Set up an Orca automation to auto-ignite the orchestrator when new issues are labeled `factory`. This removes even the ignition step from your workflow.

---

## Troubleshooting

### Orca CLI fails to connect

**Problem:** `orca status --json` returns an error or hangs

**Solutions:**
1. Verify Orca Desktop App is running
2. Check Settings → Experimental → orchestration is enabled
3. Restart the Orca Desktop App
4. Ensure PATH points to the correct Orca CLI binary
5. Try `orca skills get orchestration --full` to verify CLI can talk to the app

**Classification:** If this persists, it's an `environment_fault` - the factory cannot run without a working runtime.

### Workers not starting

**Problem:** `worker-start` commands fail or workers never complete

**Solutions:**
1. Check `orca orchestration worker-list --include-remote --json` to see worker states
2. Verify `default_worker_agent` in config.json matches installed agents (e.g., `codex`, `claude`)
3. Check agent availability: `orca agent list`
4. Review worker logs: `orca orchestration worker-read --dispatch <id> --limit 50 --json`
5. Ensure sufficient resources (CPU, memory) on the machine

### Checks gate fails

**Problem:** Checks return `GATE: FAIL` or `GATE: BLOCKED`

**Solutions:**
- **FAIL:** The builder worker will be re-dispatched automatically. Review `checks.md` to see which command failed.
- **BLOCKED:** This is an `environment_fault` - the check command cannot execute (missing tool, wrong path). Fix the environment before retrying.

### GitHub CLI authentication issues

**Problem:** `gh pr create` fails with authentication errors

**Solutions:**
```bash
gh auth login
gh auth status
```

Ensure the same authentication is available in the environment where the orchestrator runs.

### Worktree issues

**Problem:** Git worktree creation or checkout fails

**Solutions:**
1. Verify the `base_branch` exists in the repository
2. Check for uncommitted changes that might block worktree creation
3. Ensure git is configured correctly in the environment
4. Manually test: `git worktree add --detach /tmp/test-worktree main`

---

## Useful Orca CLI Commands

### Monitoring and Debugging

```bash
# Check Orca runtime status
orca status --json

# List all workers (including remote)
orca orchestration worker-list --include-remote --json

# List workers for a specific run
orca orchestration worker-list --run <run_id> --json

# Read worker output/logs
orca orchestration worker-read --dispatch <dispatch_id> --limit 50 --json

# Check coordinator inbox (what the orchestrator sees)
orca orchestration check --wait --types "worker_done,escalation,question" --timeout-ms 900000 --json

# List available agents
orca agent list

# Get orchestration skill details
orca skills get orchestration --full
```

### Worktree Management

```bash
# List all worktrees
git worktree list

# Remove a worktree
git worktree remove <path>

# Prune stale worktrees
git worktree prune
```

### Understanding Worktree Types

| Type | Description | When Used |
|---|---|---|
| `current` | The main repository checkout | Intake, spec, approver (read-only analysis) |
| `new-child` | Creates a new git worktree/branch | Builder (writes code) |
| `isolated` | Separate read-only worktree for each worker | Reviewers (parallel, isolated) |

---

## Example: Handling Failures and Recovery

### Scenario: Builder fails checks

**What happens:**
1. Builder worker completes with `worker_done succeeded`
2. Checks worker runs test/lint commands
3. Checks.md shows `GATE: FAIL` - e.g., `npm test` failed with 2 broken tests

**Factory response (automatic):**
1. Orchestrator sees `GATE: FAIL`
2. Re-dispatches the builder worker with the same branch
3. Builder reads the checks failure, fixes the broken tests
4. Re-runs the checks gate
5. If PASS → proceeds to review wave
6. If FAIL again → another builder dispatch (up to policy limits)

**What you do:**
- Nothing. The factory loops automatically until checks pass or max retries reached.
- Review `jobs/<id>/checks.md` and `jobs/<id>/build.md` to understand what failed.

### Scenario: Reviewer requests changes

**What happens:**
1. Code reviewer returns `CHANGES` - e.g., "missing test for empty filter case"
2. Orchestrator routes this back to builder
3. Builder fixes the issue, re-runs checks
4. **Entire review wave runs again** (all reviewers in parallel)
5. This repeats up to `max_review_rounds` (default: 3)

**Factory response (automatic):**
- Loop: builder → checks → review wave → builder (if changes requested)
- After `max_review_rounds` exceeded: files as `factory_bug` in ledger

**What you do:**
- Review `jobs/<id>/review-*.md` to see what changes were requested
- If the factory cannot resolve after max rounds, intervene by fixing the prompt or config

### Scenario: Intake rejects request

**What happens:**
1. Intake worker reads the issue
2. Returns `REJECT` - e.g., request is out of scope, violates security policy, or is ambiguous
3. Orchestrator writes `decision.md` with REJECT
4. Run ends without starting any other workers

**Factory response (automatic):**
- Stops immediately, no resources wasted
- Evidence saved in `jobs/<id>/intake.md` and `decision.md`

**What you do:**
- Review why intake rejected. If incorrect, update the intake prompt in `agents/intake.md`
- If the issue truly needs clarification, the user should update the GitHub issue

---

## Common Issues and Solutions

| Issue | Cause | Solution |
|---|---|---|
| `orca status` fails | Orca app not running or orchestration disabled | Start Orca app, enable orchestration in Settings |
| Workers hang indefinitely | Network issue or agent stuck | Check with `worker-list`, try restarting Orca app |
| `gh pr create` fails | Missing GitHub auth or insufficient permissions | Run `gh auth login`, verify repo permissions |
| Checks timeout | Slow tests or missing dependencies | Optimize tests, install dependencies, increase timeout in config.json |
| Reviewers keep requesting changes | Insufficient spec or ambiguous requirements | Improve spec-writer prompt, clarify requirements in issue |
| Protected paths were modified | Builder ignored config.json policies | Strengthen builder prompt, add explicit constraints |
| Parallel reviewers conflict | Same issue found by multiple reviewers | Accept first finding, others will see the fix in next round |

---

## Configuring Different Agents

The factory can use different Orca agents for different roles by modifying `config.json`:

```json
{
  "orca": {
    "default_worker_agent": "codex",
    "agent_overrides": {
      "builder": "claude",
      "reviewer-security": "claude",
      "reviewer-ux": "codex",
      "reviewer-ui": "codex"
    }
  }
}
```

**When to use different agents:**
- **codex**: Fast, good for implementation and straightforward reviews
- **claude**: Better for complex reasoning, security analysis, UX evaluation
- **custom agents**: If you have specialized agents for your domain

**Example:** Use Claude for security-sensitive work and code reviews, Codex for implementation and standard reviews.

---

## Roles

| Role | Prompt | Edits product code |
|---|---|---|
| Orchestrator (ignite this one) | `agents/orchestrator.md` | No |
| Intake / spec / reviewers / approver | `agents/*.md` + `WORKER_CONTRACT.md` | No |
| Builder | `agents/builder.md` | Yes, assigned worktree only |

---

## Safety

Issue bodies are data, not overrides. No secrets in prompts or `jobs/`. No production. No stripping protected tests. Base branch stays untouched until a configured merge.

---

## Further reading

- Package copy (same contract, shorter): [`orca-software-factory/README.md`](orca-software-factory/README.md)
- Coordinator script: [`orca-software-factory/ORCA_FACTORY.md`](orca-software-factory/ORCA_FACTORY.md)
- Ledger: [`orca-software-factory/bugs/README.md`](orca-software-factory/bugs/README.md)
- Orca orchestration: https://www.onorca.dev/docs/cli/orchestration
- Version-matched skill: `orca skills get orchestration --full`
