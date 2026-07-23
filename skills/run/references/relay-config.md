# `relay.config.json` — the project config

An **optional** per-project file. It declares the facts Relay would otherwise have to discover or
infer on every run.

**Where it lives:** the repo root for a single repo; for a poly-repo solution, the directory that
*contains* the repos. Relay looks in the working directory and then walks **up** to find it, so a
run started from inside one repo of a solution still picks up the shared config. A project with no config behaves exactly as Relay always
has — the config is an override layer, never a requirement.

## The boundary rule

> **Config holds what the harness EXECUTES. CLAUDE.md holds what the agent must UNDERSTAND.**

Exact command strings, branch patterns, gate lists, and stopping points go in the config.
Domain language, architecture rules, "why this build path is fragile", and any other judgment
stays in `CLAUDE.md`. The config does **not** replace `CLAUDE.md` and is **never** global — each
project has its own, sitting beside its own `CLAUDE.md`.

When a piece of prose has a *deterministic consequence*, encode the consequence here and leave the
reasoning in `CLAUDE.md`. Both reach the agents; only one is executed.

**Conflict:** config wins for executable facts (specific, machine-checked). `CLAUDE.md` wins for
judgment.

## Schema

Every field is optional.

```json
{
  "version": 1,

  "commands": {
    "setup": "nvm use",
    "build": "pnpm build",
    "test":  "pnpm test",
    "lint":  "pnpm lint"
  },

  "gates": {
    "required": ["build", "lint"],
    "advisory": ["test"]
  },

  "git": {
    "defaultBranch": "main",
    "branchPattern": "<type>/<ticket>-<slug>",
    "coAuthor": false
  },

  "ship": { "stopAfter": "pr" },

  "repos": [
    { "name": "api", "path": "packages/api" },
    { "name": "web", "path": "packages/web", "dependsOn": ["api"] }
  ]
}
```

Drop `repos` and `ship` and the common case is eight lines.

## Fields

| Field | Type | Default when absent |
|---|---|---|
| `version` | int | `1` |
| `commands.setup` | string | none — no env prep step |
| `commands.build` | string | Scout discovers it |
| `commands.test` | string | Scout discovers it |
| `commands.lint` | string | Scout discovers it |
| `gates.required` | string[] | every declared command is required |
| `gates.advisory` | string[] | `[]` |
| `git.defaultBranch` | string | resolved from the remote |
| `git.branchPattern` | string | `<type>/<slug>` |
| `git.coAuthor` | bool | defer to `CLAUDE.md` |
| `ship.stopAfter` | `commit`\|`push`\|`pr`\|`merge` | `pr` |
| `repos` | array | single repo at the working directory |

**`gates`** — names in `required` / `advisory` refer to keys in `commands`. A red **required**
gate fails the run and sends Build back around. A red **advisory** gate is reported but does not
block; use it for a flaky suite, or a project whose tests can't cover the change.

**`branchPattern` tokens** — `<type>` (feature/fix/hotfix/chore), `<ticket>` (the key if the task
has one; dropped along with its trailing separator if not), `<slug>` (short kebab-case summary).

**`ship.stopAfter`** — where Ship stops:

| Value | Ship does |
|---|---|
| `commit` | commit locally, stop |
| `push` | commit + push the branch, stop |
| `pr` | commit + push + open the PR, stop **(default)** |
| `merge` | all of the above, then merge |

## Stale config: fall back, never hard-fail

Config that is out of date gets *executed*, which is worse than prose that gets reinterpreted.
If a declared command **fails to launch** — command not found, bad interpreter, missing script —
that is not a build failure:

1. Fall back to normal discovery for that command.
2. Continue the run.
3. Report in the verdict that the config entry is stale, naming the field and the command.

Only treat a command as a genuine gate failure when it *ran* and came back red.

## Poly-repo

Projects that omit `repos` never encounter any of this.

1. **Absent `repos`** → a single repo at the working directory; top-level `commands` apply.
2. **Per-repo `commands` override top-level.** Top-level is the default; a repo declares only what
   differs.
3. **`dependsOn` gives a topological order**, used for build order during Build and merge order at
   Ship.
4. **Identical commands run ONCE per pass, not once per repo.** If several repos resolve to the same
   command string for a gate, deduplicate it. Some projects have one solution-level build that
   covers several repos at once (a Delphi group project, a Gradle multi-project, a workspace
   build) — running it once per repo is pure waste, and its output is the same either way.
5. **A gate whose key a repo has no command for is SKIPPED for that repo — not failed.** If `lint`
   is required and one repo declares no lint command, that repo simply has no lint gate. Report it
   as skipped; do not fail the run and do not invent a command.
6. **`dependsOn` does not imply each repo builds alone.** It states ordering. When repos share a
   build command (rule 4), that command owns its own internal ordering and `dependsOn` only governs
   merge order at Ship.

Build creates the **same branch name in every repo it touches**, so the set is correlatable.
Ship opens one PR per touched repo and walks them in dependency order.

**Cross-repo merges are not atomic.** If one repo merges and another's PR is rejected, the tree is
in a split state. Relay cannot prevent this — it must record exactly which repos merged and which
did not, and never report a clean ship.

## Reading it

The orchestrator loads the file once and passes the relevant slice into each dispatch prompt.
Agents do not each re-read it.

| Stage | Uses |
|---|---|
| Scout | `commands.*`, `repos` — skips discovery for what's declared, recons only the gaps |
| Planner | `commands.*`, `gates.*` — writes the plan's Verify section from the declared gates |
| Build | `git.defaultBranch`, `git.branchPattern`, `commands.setup`, `commands.build`, per-repo overrides |
| Test | `gates.required`, `gates.advisory`, `commands.*` |
| Ship | `ship.stopAfter`, `git.coAuthor`, `repos[].dependsOn` |
