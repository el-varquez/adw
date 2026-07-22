# Relay

**Relay** is a Claude Code plugin that takes a **task** and drives it through
**Plan → Build → Test → Engineer Review → Ship**, auto-looping on failure — a relay of
single-purpose agents passing the work down the line. Give it a task; the Planner drafts a plan
you approve, then it builds. One project-agnostic workflow that reads each project's own
`CLAUDE.md` to learn how *that* project builds and tests.

## Install

```
/plugin marketplace add el-varquez/relay
/plugin install relay@relay
```

Then, from inside any project:

```
/relay:run "<a task or problem>"
/relay:run "<a task or problem>" --max-rounds 5
```

Just describe the task — the Planner recons the code, drafts a plan, and you approve it before any building. (Want it to build on an existing plan or spec? Mention that file's path in the task.)

## The flow

```mermaid
flowchart LR
    E(["Engineer: /relay:run (task)"]) --> P["🧭 Planner (relay:plan)"]
    P -->|"sub-agent"| PS["🔍 Scout (relay:scout)<br/>recon"]
    PS -->|"context pack"| P
    P -->|"PLAN BLOCKED"| X(["⛔ clarify"])
    P -->|"PLAN READY"| PR{{"⏸ Plan Review (you)"}}
    PR -->|"reject"| P
    PR -->|"approve"| B["🔨 Build (relay:build)<br/>implement + compile"]
    B -->|"PASS"| T["✅ Test (relay:test)<br/>tests + lint"]
    T -->|"FAIL"| B
    T -->|"PASS"| R{{"⏸ Engineer Review + QA (you)"}}
    R -->|"reject"| B
    R -->|"approve"| SH["🚢 Ship (orchestrator)<br/>commit → push → PR → merge"]
    SH --> DONE(["🎉 Merged"])
```

*Renders as a diagram in Obsidian, GitHub, and most Markdown viewers. Scout (`relay:scout`) is the Planner's read-only recon sub-agent. Build↔Test loops at most 3 rounds (`--max-rounds N`), then escalates to you.*

<details>
<summary>Plain-text version</summary>

- **Engineer** runs `/relay:run "<task>"`.
- **Planner** (`relay:plan`) spawns the **Scout** (`relay:scout`) sub-agent to recon the code, then drafts a plan → you approve/edit it at **Plan Review** (or the Planner reports it's blocked → you clarify).
- **Build** (`relay:build`) implements + compiles → **Test** (`relay:test`) runs tests + lint.
- Test fail → back to Build. Test pass → **Engineer Review + QA** (you). Reject → back to Build.
- Approve → **Ship** (orchestrator): commit → push → PR → merge.
- Build↔Test loops at most 3 rounds (`--max-rounds N`), then escalates to you.

</details>

## How it works

- **Task-driven.** Give Relay a task — the **Planner** (`relay:plan`) recons the code via its Scout
  sub-agent (`relay:scout`) and drafts a grounded plan for you to approve before any building starts.
  It uses `superpowers:writing-plans` if installed, otherwise plans plainly.
- **Project-agnostic.** Nothing is hardcoded. The orchestrator (your Claude Code session)
  reads *this* project's `CLAUDE.md` for build/test/lint commands and git conventions, then
  delegates to subagents.
- **Read-only recon + verify around one writer.** `relay:scout` recons the code for the Planner;
  `relay:test` verifies the *result* after Build. Only `relay:build` edits code — enforced by
  tool scope.
- **QA is part of Review.** The Engineer Review gate is where QA runs the manual-verify
  checklist. Approval means QA + engineer signed off — that authorizes Ship.
- **Persistent Build.** The Build agent is continued across retries, so it remembers prior
  attempts. Both fail loop-backs feed it. Cap: 3 rounds, then it escalates to you.
- **Nothing commits until Ship.** Build only edits the working tree — it never commits, branches,
  or pushes. All changes are committed once, at Ship, after Engineer + QA approval.

## Stages

| Stage | Writes? | Job | Passes when |
|-------|---------|-----|-------------|
| Plan | no | Planner turns your task into a grounded plan (recons via Scout) | you approve the plan |
| Scout | no | recon sub-agent — explores code + discovers build/test/lint cmds for the Planner | context pack → `RECON DONE` |
| Build | yes | implement the plan, then compile | clean compile → `PASS` |
| Test  | no | run the plan's Verify — tests + lint | every gate green → `PASS` |
| Review| — | the mandatory human gate; QA tests here | you approve with QA sign-off |
| Ship  | git | commit → push → PR → merge | merged |

## Notes

- **Run logs** land in `./.relay/runs/<date>-<slug>.md` per project.
- **Git conventions** (branch naming, commit-message style, co-author trailers) follow *your*
  project or global `CLAUDE.md` — the workflow imposes none of its own.
- Requires Claude Code with plugin support.

## Credits

- Inspired by [IndyDevDan](https://www.youtube.com/@indydevdan)'s **Agentic Developer Workflow (ADW)** concept.

## License

MIT © 2026 el-varquez
