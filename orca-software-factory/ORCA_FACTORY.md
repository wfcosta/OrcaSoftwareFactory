# Orca Software Factory — coordinator runbook

Use this document as the orchestration instructions in an Orca conversation
opened on the target repository. **Orca is the control plane:** create agents,
assign their prompts, select their worktrees, monitor them, and read their
responses in Orca. Do not assume a CLI or API for spawning agents. Orca UI
labels may differ by release; use the corresponding agent/session/worktree
controls available in the installed version.

## Mission

Take one approved request from intake through a tested, reviewed branch and a
human-reviewed PR recommendation. Use the role prompts in `agents/`. Record
state and complete agent responses in `jobs/<job-id>/` (create the directory if
needed). Keep one job per issue/request and resume from its recorded state.

The coordinator sequences work; specialist agents provide focused analysis or
implementation. Explicitly inspect each agent's final response in Orca. Do not
advance based solely on an agent saying it completed. When a decision is
ambiguous, checks fail, or evidence is unavailable, stop and report BLOCKED or
ESCALATE instead of guessing.

## Start a job

1. Confirm the target repository, base branch, and requested issue/feature. Read
   `config.json`, repo guidance (`AGENTS.md`, `CONTRIBUTING.md`, etc.), and
   discover the real validation commands. Do not assume this template's
   defaults match the project.
2. Make `jobs/<job-id>/` from `templates/job-template.md`. Use a stable
   identifier such as `001-short-feature-name`; record the request as quoted
   data in `intake.md` and identify the source/link.
3. Create an **Intake** agent through Orca. Give it `agents/intake.md`, the
   request, and relevant project context. Read its response in Orca and save
   it to `decision.md`.
4. Follow its verdict:
   - `NEEDS-INFO`: ask the requester the listed questions and pause this job.
   - `REJECT`: record the rationale and stop.
   - `ACCEPT`: record scope, risks, and surfaces, then continue.
5. Create a **Spec writer** agent. Give it `agents/spec-writer.md`, the accepted
   scope, project instructions, and relevant code context. Inspect and save its
   plan as `spec.md`. Resolve material questions with the requester before
   implementation.

## Build and gate

6. In Orca, start a **Builder** agent with `agents/builder.md`. Assign the
   repository and a dedicated git worktree/branch based on the configured base
   branch. Never let multiple builders modify the same worktree. Include the
   accepted spec, repo guidance, and checks. Ask for a concise build report and
   commits on that branch; do not ask it to merge or deploy.
7. Read the builder's response and inspect its branch diff in Orca. Save its
   report to `build.md`. Verify the branch is based on the intended base and
   that it has not modified or removed protected tests/policies. If unsafe or
   out of scope, return concrete findings to the builder and repeat the gate.
8. Run the configured checks from `config.json` in the builder worktree (or
   assign a separate validation agent). Inspect the actual output. Save command,
   exit status, and relevant output in `checks.md` with exactly one status:
   `GATE: PASS`, `GATE: FAIL`, or `GATE: BLOCKED`. Never start review on FAIL or
   BLOCKED; send the actionable failure to the builder and return to step 7.

## Independent review

9. After `GATE: PASS`, select reviewers from intake's affected surfaces and
   risk. Start each reviewer in Orca against the same branch/diff, preferably
   in an isolated/read-only worktree. They must not edit product files.
   Required minimum: code review. Add security for security-sensitive changes;
   UX and UI for user-facing changes. Run reviewers in parallel when Orca
   supports it, but wait for and inspect every required response.
10. Save each response as `review-<role>.md`. Any `CHANGES` or `BLOCKED` means
    send the findings to the builder, inspect the new diff, rerun the full gate,
    and repeat applicable reviews. Limit loops to `max_review_rounds` from
    config; when exceeded, escalate to a human. Never reinterpret a finding as
    PASS without evidence.

## Decision and handoff

11. When all required reviewers return PASS, ask an **Approver** agent using
    `agents/approver.md` to assess the spec, diff summary, check evidence, and
    review reports. Save the response in `approval.md`. The result must be
    `APPROVE` or `ESCALATE`; missing evidence always escalates.
12. On `ESCALATE`, report the unresolved questions and evidence to the human.
    Do not publish or merge automatically.
13. On `APPROVE`, prepare a PR description with the configured title/body
    template and link the issue. Have a human inspect the final diff, checks,
    and all reviews before publishing the PR. The human owns PR creation,
    approval, merge, and deployment. Never merge or deploy through this flow.
14. Update `job.md` with final status and links. Keep all evidence in its job
    directory. Do not include secrets or unnecessary personal data.

## Orca orchestration checklist

For every delegated role, perform these actions using Orca's available UI:

1. Create an agent/session and select the intended project/worktree.
2. Paste the matching role prompt from `agents/` followed by the task context.
3. For implementers, use a dedicated branch/worktree; for reviewers, inspect
   the implementation branch without editing it.
4. Monitor progress and handle questions; do not start conflicting writers.
5. Open/read the completed response in Orca, verify claims against the workspace,
   and save the response in the job folder before proceeding.

If Orca cannot provide the required isolation or evidence, serialize the work or
stop and report the limitation. The workflow is intentionally not tied to
specific Orca button names or automation APIs.
