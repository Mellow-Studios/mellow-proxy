# MELLOW.md

`mellow-proxy` is the Mellow Studios fork of
[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) and runs as the
studio's subscription proxy. It is one patch set carried on upstream release
tags. Nothing here is filed upstream.

- Version: `8.0.22-mellow.1`
- Upstream base: tag `v8.0.22`, commit `67465884`
- Last sync: 2026-10-08
- Configuration, service and deploy: the MellowBox record, repository
  `Mellow-Studios/mellowbox`, directory `proxy/`; MellowOps project record
  `content/projects/mellowbox/`

## Branches

- `main` mirrors upstream. Never commit to it.
- `mellow` is the default branch. It carries the patch set on the upstream
  base tag.

## Patch admission

A change enters only when it cannot be configuration, stays small, carries a
test, and has a row below with its origin. Flipped defaults (session affinity,
cloaking, websockets) stay in the MellowBox config, never in this fork.

## Patch set

1. `soonest-reset` routing strategy. Origin: upstream PR #6277,
   https://github.com/router-for-me/CLIProxyAPI/pull/6277, closed 2026-10-02,
   ported unchanged. Within the highest ready priority tier, it picks the
   credential whose weekly quota resets soonest. Files:
   `sdk/cliproxy/auth/selector_soonest_reset.go`,
   `sdk/cliproxy/auth/selector_soonest_reset_test.go`,
   `sdk/cliproxy/auth/scheduler.go`, `sdk/cliproxy/auth/conductor_selection.go`,
   `sdk/cliproxy/service_config.go`, and comments in
   `internal/config/config_types.go` and `config.example.yaml`.
2. Management API accepts `soonest-reset`. Origin: this fork. PR #6277 did not
   update `normalizeRoutingStrategy`, so `PUT /v0/management/routing/strategy`
   rejected the strategy the config file accepts. Files:
   `internal/api/handlers/management/config_basic.go`, test in
   `internal/api/handlers/management/config_basic_weight_test.go`.

3. Codex weekly reset with no secondary window. Origin: this fork,
   2026-10-02, found on the home node with five Codex accounts. Codex reports
   an absent secondary window as resetting in 0 seconds; patch 1 took that
   as a known reset and never read the primary window, so every Codex
   credential tied at "unknown" and the lowest id served every session.
   A reset that is not in the future now counts as unknown before the
   primary window is read. Files: `sdk/cliproxy/auth/selector_soonest_reset.go`,
   test `TestWeeklyQuotaResetAt_CodexSecondaryWindowAbsent`.

Fork files, no behaviour: this file, the first line of `AGENTS.md`, and
`runs/` in `.gitignore` for build and scratch output.

## Build

```bash
go build -ldflags "-s -w -X main.Version=8.0.22-mellow.1 -X main.Commit=$(git rev-parse --short HEAD) -X main.BuildDate=$(date -u +%Y-%m-%dT%H:%M:%SZ)" -o runs/scratch/cli-proxy-api ./cmd/server
```

Check: `runs/scratch/cli-proxy-api -h` prints
`CLIProxyAPI Version: 8.0.22-mellow.1, ...` on its first line and exits.

## Rebase onto a new upstream tag

Runs only in a MellowOps maintain run.

1. `git fetch --tags https://github.com/router-for-me/CLIProxyAPI.git`
2. `git rebase --onto <new tag> <old tag> mellow`
3. On a conflict, stop and report the file and the patch. Never resolve it
   silently.
4. `go test ./sdk/cliproxy/... ./internal/api/handlers/management/... ./internal/config/...`,
   then `go test ./...`.
5. Build with the version set to `<upstream>-mellow.N`. `N` starts at 1 on each
   upstream tag and rises with each patch-set change on that tag.
6. Update the version, upstream base and last sync lines in this file.
