---
name: beads-github-sync-testing
description: End-to-end testing recipes for the `bd github-sync` and `bd dolt push/pull --auth` authentication flow.
---

# beads github-sync end-to-end testing

## Devin Secrets Needed

- `BD_GITHUB_CLIENT_ID` / `BD_GITHUB_CLIENT_SECRET` — only needed for a live OAuth device-flow test against GitHub.
- `BD_GITLAB_CLIENT_ID` / `BD_GITLAB_CLIENT_SECRET` — only needed for GitLab OAuth.
- `GITHUB_TOKEN` or an OS-keyring-stored `gh`/`glab` credential — only if testing real delegated auth; prefer mocked binaries.

## Canonical build / test commands

```bash
source .buildflags
make build        # writes ./bd
make ci-pr-lint   # gofmt + golangci-lint linux + windows
make test         # runs scripts/test.sh with isolated BEADS_HOME
```

- Do **not** run `go build -o bd` directly; the Makefile sets `-tags gms_pure_go` and build SHA injection.
- `CGO_ENABLED=1` and `GOFLAGS=-tags=gms_pure_go` are loaded from `.buildflags`.

## CLI smoke tests

```bash
./bd github-sync --help
./bd github-sync status --provider oauth --host github.com --json
./bd github-sync status --provider oauth --host github.com
./bd github-sync git-credential --help   # hidden command
```

- `github-sync status` requires a beads workspace (run `bd init` in a temp directory first).
- With no `client_id` configured, OAuth status reports `"ready": false`.

## Mocking delegated `gh` / `glab` binaries

Use a temp `FAKE_BIN` directory on the front of `PATH` so `bd` resolves the fake binary instead of the system `gh`/`glab`.

- Fake `gh` must accept `auth status --hostname <host>` and exit 0, and `auth git-credential get` and print:
  ```
  username=oauth2
  password=<token>
  ```
- Fake `glab` must accept `auth status --hostname <host>` (exit 0) and `auth status --show-token` printing a line matching `token[:=]\\s*([A-Za-z0-9_\\-\\.]+)`.

Expected `status` JSON:

```json
{"host":"github.com","provider":"gh","ready":true}
{"host":"gitlab.com","provider":"glab","ready":true}
```

## Proving `GIT_CONFIG_PARAMETERS` wiring in `bd dolt push --auth`

`withRemoteAuth` builds a `GIT_CONFIG_PARAMETERS` string that installs a per-command credential helper and disables client hooks. You can verify it in a disposable workspace with an invalid remote and a `git` wrapper on `PATH`:

```bash
mkdir -p /tmp/fakebin
cat > /tmp/fakebin/git <<'EOF'
#!/bin/bash
echo "invoked git $*" >> /tmp/fakebin/git.log
echo "  GIT_CONFIG_PARAMETERS=$GIT_CONFIG_PARAMETERS" >> /tmp/fakebin/git.log
/usr/bin/git "$@"
EOF
chmod +x /tmp/fakebin/git

# create workspace and remote
rm -rf /tmp/bdtest && mkdir /tmp/bdtest && cd /tmp/bdtest
/home/ubuntu/repos/beads/bd init --prefix tst --skip-hooks --skip-agents --non-interactive
/home/ubuntu/repos/beads/bd dolt remote add origin https://invalidhost.invalid/repo.git

# run with fake gh and git wrapper
PATH="/tmp/fakebin:$PATH" /home/ubuntu/repos/beads/bd dolt push --auth gh --remote origin
```

Inspect `/tmp/fakebin/git.log`; it should contain an entry like:

```
credential.https://invalidhost.invalid.helper=!/tmp/fakebin/gh auth git-credential' 'core.hooksPath=/dev/null'
```

For `--auth glab` the string also contains `'http.https://...proactiveAuth=basic'`. For `--auth oauth` it points back to the `bd` binary with `github-sync git-credential`.

## Manual `git credential fill` verification

```bash
export GIT_CONFIG_PARAMETERS="'core.hooksPath=/dev/null' 'credential.https://github.com.helper=!/tmp/fakebin/gh auth git-credential'"
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
```

Expected output:

```
protocol=https
host=github.com
username=oauth2
password=<token>
```

## OAuth device flow

A real CLI OAuth login cannot run without a registered `client_id`. Fall back to the unit test:

```bash
source .buildflags
go test ./internal/syncauth -run TestOAuthLoginStoresToken -v
```

To exercise the CLI path without network:

```bash
bd github-sync login --provider oauth --host github.com --dry-run
bd github-sync login --provider oauth --host github.com
```

The first prints `Would log in to github.com using oauth`. The second errors cleanly with a missing `client_id` message.

## Lint and unit tests

```bash
make ci-pr-lint
go test ./internal/syncauth -v
```

## Full suite caveats

`make test` runs `scripts/test.sh` with a 25-minute package timeout. On git 2.34.1, `cmd/bd/worktree_remove_git_test.go` fails because it uses `git worktree list --porcelain -z`; the `-z` flag is only available in newer git. These failures are usually pre-existing and unrelated to syncauth. Check `internal/worktreeremove` itself separately:

```bash
go test ./internal/worktreeremove -v
```
