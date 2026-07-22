# Detecting build / test / lint commands for any project

The ADW is project-agnostic. This is the procedure the Scout agent follows to learn how the
CURRENT project builds, tests, and lints. Resolve in priority order and STOP at the first
source that gives a confident answer; fall through only when it doesn't.

## Priority order

1. **`./CLAUDE.md` or `./AGENTS.md`** (primary). Well-run repos document exact commands. Read
   them and extract the build / test / lint invocations verbatim, including any required
   environment setup and platform/config flags.
2. **The plan/handoff's own "Verify" / "Build" section.** If the plan names the commands to run
   (or says tests are deferred / manual), that governs.
3. **Ecosystem heuristics** (below), inferred from files present in the repo.
4. **Unresolved → report it.** If none of the above yields a confident command set, say so in
   the verdict so the orchestrator can ask the Engineer. Never guess.

## Ecosystem heuristics

| Detect (file present)       | Build                      | Test                   | Lint / typecheck         |
|-----------------------------|----------------------------|------------------------|--------------------------|
| `package.json`              | `<pm> run build`           | `<pm> run test`        | `<pm> run lint`; `tsc`   |
| `*.dproj` / `*.groupproj`   | `msbuild <proj> /t:Build`  | build + run test runner| (compiler warnings)      |
| `*.sln` / `*.csproj`        | `dotnet build`             | `dotnet test`          | `dotnet format --verify` |
| `Cargo.toml`                | `cargo build`              | `cargo test`           | `cargo clippy`           |
| `pyproject.toml`/`setup.py` | `python -m build`          | `pytest`               | `ruff check` / `flake8`  |
| `go.mod`                    | `go build ./...`           | `go test ./...`        | `go vet ./...`           |
| `Makefile`                  | `make` / `make build`      | `make test`            | `make lint`              |

**Node package manager (`<pm>`)** — pick from the lockfile: `pnpm-lock.yaml` → `pnpm`;
`yarn.lock` → `yarn`; `package-lock.json` → `npm`. Only run scripts that actually exist in
`package.json` `scripts`.

**Delphi note:** many Delphi repos build only for a specific platform/config and require an env
script first (e.g. a repo may need `call rsvars.bat`, then
`msbuild … /p:Config=Release /p:Platform=Win64`). Read CLAUDE.md — do not assume defaults.

## Output

Return a command set: `{ build, test, lint/typecheck, env-setup, target repo(s) }`, plus any
**fragile-path caveats** the project's CLAUDE.md calls out (e.g. "the full EXE cmdline link is
fragile; build the packages instead"). Downstream agents run exactly these; they don't re-derive.
