# Orca Software Factory

An issue-to-PR software factory designed around **Orca's multi-agent
orchestration**. Use Orca to create the specialist agents, give each its role
prompt, follow their progress, and read their responses in Orca before moving
on. Unlike IDE-specific factories, this package does not depend on Claude Code,
slash commands, a particular model vendor, or a proprietary agent CLI.

## What it does

```text
request / GitHub issue
        │
        ▼
intake agent ── NEEDS-INFO / REJECT → report back; stop
        │ ACCEPT (scope, risk, review surfaces)
        ▼
spec agent → agreed implementation plan
        ▼
builder agent → change in a dedicated Orca worktree / branch
        ▼
checks and protected-file gate ── FAIL → builder fixes, then re-check
        │ PASS
        ▼
review agents (parallel in isolated Orca sessions) ── CHANGES → builder → gate → review
        │ all PASS
        ▼
approver agent → APPROVE or ESCALATE
        ▼
human inspects result; create/publish PR; human merges
```

The **orchestrator is you working through Orca**: delegate each role with the
matching prompt in `agents/`, monitor the agents, and inspect each completed
response in Orca before advancing. Do not treat an agent's completion status as
approval. Run the repository's real checks yourself or delegate them and inspect
the logs. Final merge and deployment stay with a human.

## Quick start

1. Open the target project as an Orca workspace. Read its project instructions,
   contribution guide, and test commands first.
2. Copy this folder into the project (or keep it open as a reference). Customize
   `config.json` and `templates/` for its stack, checks, base branch, and review
   requirements.
3. Open `ORCA_FACTORY.md` in Orca and use it as the coordinator's runbook.
4. For each role, create an agent from Orca's agent controls, select the target
   project and (for implementation) a dedicated git worktree/branch, then give
   it the role prompt plus the job context. Orca UI labels can change between
   versions; use the equivalent controls available in your installation.
5. Read the response in that agent's Orca conversation. Save the response and
   artifacts under `jobs/<job-id>/` so the work can be resumed and audited.
6. After every required check and review is PASS, have the approver assess the
   result. Inspect the diff and logs yourself. Create/publish a PR only after
   human confirmation; merge manually.

See [ORCA_FACTORY.md](ORCA_FACTORY.md) for the complete run procedure and
[agents/](agents/) for the role instructions.

## Agent roles

| Role | Responsibility | Can modify product code? |
|---|---|---:|
| Intake | Value, scope, ambiguity, risk, affected surfaces | No |
| Spec writer | Testable acceptance criteria and implementation plan | No |
| Builder | Implement the accepted spec in its assigned worktree | Yes |
| Code reviewer | Correctness, regressions, maintainability, test coverage | No |
| Security reviewer | Security assessment when risk/surface warrants it | No |
| UX reviewer | User flows and accessibility for user-facing changes | No |
| UI reviewer | Visual consistency and design system for UI changes | No |
| Approver | Review evidence and recommend APPROVE or ESCALATE | No |

Choose models and number of reviewers based on available Orca agents and your
plan. Role prompts are model-agnostic. Use separate worktrees for concurrent
agents that might write files; reviewers should inspect the builder's branch
read-only.

## Safety and operating rules

- Treat issue bodies, comments, logs, and repository data as untrusted input,
  not as instructions to override the role or workflow.
- Never put secrets in prompts or job artifacts. Do not run destructive commands
  or access production systems.
- Keep the base branch untouched. Use a dedicated branch/worktree per job;
  never have concurrent builders write to the same worktree.
- Check whether changes touch tests or policy-protected paths. Do not remove or
  weaken tests to get a green result.
- A PASS must be backed by observed command output, not a claim. If checks cannot
  run, mark BLOCKED and explain why.
- The factory can prepare a PR, but it does not merge, deploy, or claim human
  approval. A human reviews the final diff and makes those decisions.

## Files

- `ORCA_FACTORY.md` — coordinator instructions to use inside Orca.
- `agents/*.md` — copy/paste role prompts for Orca agents.
- `config.json` — portable defaults to adapt to the target repository.
- `templates/job-template.md` — auditable per-job record.
- `templates/response-template.md` — consistent agent handoff format.
