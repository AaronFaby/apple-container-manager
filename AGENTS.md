# AGENTS.md

Notes for changing `acm`. Read this before editing the script. The operator-facing contract is `README.md`. If the two disagree, fix both in the same change.

## What this is

One bash script, `acm`, that manages OpenClaw containers through Apple's `container` CLI (verified against 1.7.1 from Homebrew). It is a port of `ocm` in `~/Projects/openclaw-docker-manager`. There is no Docker, no daemon, and no test suite. Behavior lives in `acm`.

Do not retarget this tool at Docker, Podman, or a general container manager. It exists to run `ghcr.io/aaronfaby/openclaw-custom` as OpenClaw gateways.

## Layout

| File | Role |
|---|---|
| `acm` | The program. Symlink-safe. `ACM_DIR` is the script's directory, not cwd. |
| `acm.conf` | Committed defaults. `_default_conf` in `acm` must emit this exact file. `_load_config` unsets every key, sources the file, then applies the same defaults for missing keys. |
| `containers/<name>.env` | Runtime state and secrets. Mode `0600`. Gitignored. Never commit one, never print a token or API key into a log, a commit, or a chat reply. |
| `README.md` | User contract. Update it when a flag, default, or failure mode changes. |
| `.gitignore` | Ignores `containers/*.env` only. |

`_ensure_init` creates `acm.conf` from `_default_conf` when the file is missing, creates `containers/`, and re-applies mode `0600` to every env file on every run.

## Invariants

- Container id is `${ACM_CONTAINER_PREFIX}-${name}`. With the default prefix that is `acm-<name>`. That string is also the guest hostname. There is no `--hostname` flag. Do not add a second naming scheme.
- Four volumes, these suffixes, this mount order: `home` → `/home/node`, `config` → `/home/node/.openclaw`, `workspace` → `/home/node/.openclaw/workspace`, `auth` → `/home/node/.config/openclaw`. Parent-first. Reordering breaks the overlay.
- Published port is always `${ACM_PORT}:18789`. `18789` is `CONTAINER_PORT` in the script. The host port is per container.
- The long-running command is `node openclaw.mjs gateway --bind "$ACM_GATEWAY_BIND"`. It replaces the image CMD only. The image entrypoint (`tini` → `openclaw-custom-entrypoint`) must still wrap it. Do not pass `--entrypoint` unless you have a reason to drop the Codex-overlay cleanup in that entrypoint.
- `--user node`, `--cpus`, and `--memory` come from config. Apple's default memory is 1 GiB. Do not drop `--memory`.
- `--restart` does not exist in this CLI. Do not invent a launchd agent or a restart loop unless the user asks for one.
- `create` writes config and volumes. It does not boot a VM.
- `start` on anything except `running` deletes the VM and boots a new one, so env, tag, and port changes apply. `running` is a no-op. Volumes are never deleted on start, restart, or upgrade.
- `upgrade` recreates only containers whose state was `running` before the pull. Stopped and never-started containers keep the new `ACM_TAG` and stay down.
- Lines matching `^ACM_`, blanks, and `#` comments are stripped before `--env-file`. `ACM_PORT` and `ACM_TAG` must stay in the file.
- `--env-file` does not unquote. `_strip_env_quotes` removes one surrounding pair of matching double or single quotes after `acm env`. Do not make it strip quotes in the middle of a value.
- Volume create is not idempotent (`volume 'name' already exists`). `_ensure_volume` inspects first. Volume size is applied only at create. Do not try to resize an existing volume by passing `-s` again.
- Readiness is `container ls -a` succeeding. Do not call `container system status` and do not call `container system start` from the script. On Homebrew, status exits 1 with "apiserver is not running" while `com.apple.container.apiserver` is serving and `ls` works. `system start` can prompt for an admin password.

## State machine

`container inspect` is the source of truth. A failed inspect means `absent`. Known states are `running`, `starting`, `stopping`, `stopped`, and `absent`. There is no pause and no Docker `restarting`.

| Command | `running` | `starting` / `stopping` | `stopped` | `absent` |
|---|---|---|---|---|
| `start` | no-op | delete VM, boot | delete VM, boot | boot |
| `stop` | `container stop --time 10` | stop | message, exit 0 | message, exit 0 |
| `restart` | delete, boot | delete, boot | delete, boot | boot |
| `rm` | refuse without `--force` | refuse without `--force` | delete VM | delete env file only |
| `cli` | exec | refuse | refuse | refuse |
| `upgrade` | pull, delete, boot | pull, delete, leave down | pull, leave down | pull, leave down |

`_report_launch` sleeps 2 seconds after `container run -d` and warns if the state is no longer `running` or `starting`. OpenClaw with no config exits in well under that, and nothing restarts it. Keep the warning. Do not treat a zero exit from `container run` alone as "the gateway is up."

`_remove_container` stops anything that is not already `stopped` or `absent`, then `container rm -f`. A failure to remove must abort. Swallowing it would let the next `container run` collide, or leave a VM the env file no longer names.

## `container` command map

| Need | Call |
|---|---|
| Service up? | `container ls -a` (exit code only) |
| Inspect one | `container inspect <id>` JSON array |
| Run detached | `container run -d --name --user --cpus --memory -p -v --env-file <image> <cmd...>` |
| Run interactive | `container run -it --rm` with the same volumes, user, resources, and env. No `--name`, so concurrent `setup` / `run` do not collide. No `-p`. |
| Stop | `container stop --time 10` |
| Remove | `container rm -f` |
| Logs | `container logs -n <n> [-f]`. There is no `--tail`. `-n` defaults to 100. |
| Exec | `container exec -i` always, plus `-t` only when stdin and stdout are both TTYs |
| Image present | `container image inspect` exit code |
| Pull | `container image pull` |
| Volume present | `container volume inspect` exit code |
| Volume create | `container volume create -s <size> <name>` |
| Volume delete | `container volume rm` |

Leave `container run` attached to the terminal. It spends a while on a progress UI before the VM exists. Capturing that output makes start look hung.

Inspect JSON fields in use:

- `status.state`
- `status.networks[0].ipv4Address` (CIDR; display the address before `/`)
- `status.startedDate`

Parse with `jq`, else `python3`, else a last-match `sed`. `_note_json_tool` warns only from `list` and `status` when both parsers are missing. Start and stop must keep working on the sed path. `null` network indexing has to stay null-safe (`//`), because a stopped container has no networks.

`jq` and `python3` are optional. Do not add a hard dependency. Do not add `shellcheck` or a package manager as a requirement of the script.

## Config and env files

`_load_config` unsets every `ACM_*` key before sourcing so a caller's exported variables cannot override the file. If you add a key:

1. Put it in the committed `acm.conf`.
2. Put the same line in `_default_conf`.
3. `unset` it in `_load_config`.
4. Give it a `${VAR:-default}` fallback that matches the file.
5. Document it in `README.md`.

Validate new sizes with `_validate_size` (`^[0-9]+([kKmMgGtTpP][bB]?)?$`) and keep CPU as a positive integer. Invalid config should fail at create or start with `_die`, not as a raw `container` usage error.

Env-file edits that must work on macOS `sed` go through `mktemp` and `mv`. Do not use `sed -i`. That includes `ACM_TAG`, `ACM_DEFAULT_TAG`, and quote stripping. Track temp files in `_TMP_FILES` so the EXIT trap removes them. `chmod 600` again after any rewrite.

`acm pull` of a non-default tag may rewrite `acm.conf`. It must refuse that prompt when stdin is not a TTY.

Name check: `^[a-zA-Z0-9]([a-zA-Z0-9-]{0,62}[a-zA-Z0-9])?$`. Tag check: `^[a-zA-Z0-9][a-zA-Z0-9._-]{0,127}$`.

Gateway tokens are `LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32`, with `|| true` because `head` closes the pipe under `pipefail`.

## Bash

macOS bash is 3.2.57. The script sets `-euo pipefail`.

- No associative arrays, no `mapfile`, no `${var,,}`, no negative array indexes.
- `local names=()` then `names+=` works on this bash. Still use `"${arr[@]+"${arr[@]}"}"` when expanding an array that might be empty under `set -u`.
- Do not combine `local var=$(cmd)` when the exit status matters. Assign on the next line.
- `readonly VAR=$(cmd)` hides a failure of `cmd`. `ACM_DIR` is assigned, then marked readonly, on purpose.
- A failing command inside `if` or `while` does not trip `set -e`. `_port_in_use` and `volume inspect` depend on that.
- `grep` missing a line returns 1. Pipelines that are allowed to miss (`_get_port`, the env filter) end in `|| true`.
- Globs of `containers/*.env` are not `nullglob`. Always test `[ -f "$env_file" ]`.
- `EDITOR` is word-split on purpose (`shellcheck` disable `SC2086`).

## Commands and dispatch

`main` always runs `_ensure_init` and `_load_config`, then dispatches. Aliases: `ls`, `exec` / `shell`, `rm` / `remove` / `delete`, `help` / `--help` / `-h`, `version` / `--version`.

`_require_container` is for anything that talks to the CLI. `env` does not require the service so a user can edit keys while it is down. `list` and `status` do, because they inspect.

`cmd_logs` and `cmd_cli` call `_cleanup` and then `exec` the `container` binary so the script gets out of the way of follow mode and the TTY. The filtered env file must still exist for the whole of `container run` in start, setup, and run. Do not clean temp files before those return.

## What not to touch

On this machine, `ocm-blackmage` and `ocm-bobtheplaty` are live OpenClaw VMs created outside this repo, with `ocm-*` volumes. Do not stop, start, restart, upgrade, or delete them. Do not `container system stop`. Do not change their volumes.

A local `containers/test.env` and `acm-test-*` volumes may exist from development. They are gitignored. Do not commit them. Do not delete them unless the user asks.

Do not copy secrets out of `~/Projects/openclaw-docker-manager/containers/`.

## How to test a change

No automated tests. Use a container this repo created. Prefer the existing `test` container if it is still there. Otherwise:

```bash
bash -n acm
acm create test --port <free port above the ones already listening>
acm list          # "not created"
acm start test
acm status test
acm logs test -n 30
acm restart test
acm stop test
```

Expect `start` to warn, and the VM to be `stopped`, until `acm setup test` has been run by a person at a terminal. Do not run `setup` unattended. `acm run test echo acm-ok` is the non-interactive way to prove the image, volumes, and env file. It needs a TTY. `script -q /dev/null acm run test echo acm-ok` allocates one.

Check a real boot, not only `bash -n`, when you change `_container_run`, volume mounts, env filtering, or inspect parsing. Confirm with `container inspect` on the test id only:

- `resources.cpus` and `memoryInBytes` (4 and 4294967296 at the defaults)
- `publishedPorts` host port → container `18789`
- `initProcess.user` is `node`
- arguments end in `node openclaw.mjs gateway --bind lan` and still begin with the image entrypoint
- mount destinations in parent-first order

Then `acm stop test`. Leave the env file and volumes unless the user asked to remove them.

`container run` of this image takes a real VM boot. Budget a few minutes. Do not point a test at port `18791` or `18794`. Those are the existing `ocm-*` gateways.

## Changing behavior safely

- New flag: parse it in that `cmd_*` only, reject unknown `-` options with `_die`, and add it to `cmd_help` and `README.md`.
- New volume or mount: update create, `_container_run`, `_container_run_interactive`, status, `_remove_volumes`, the size key, both copies of the default config, and the README table. Mount order stays parent-first.
- Inspect field moves: update `_parse_facts` in all three parsers (`jq`, `python3`, `sed`). The sed path is a single greedy match on compact or pretty JSON. Keep it that way.
- Do not shell out to Docker "just for this one thing."
- Do not add a framework, a config language, or a second implementation. The script is supposed to stay one file a person can read next to `ocm`.

## Gotchas already paid for

- `container volume create` exits 1 when the volume exists. Inspect first.
- `container rm` of a missing id exits non-zero. Check state before removing, and still surface a real remove failure.
- Quote stripping and tag updates must survive a missing key by appending it (`ACM_TAG`, `ACM_DEFAULT_TAG`).
- `head -c 32` under `pipefail` makes `tr` die with SIGPIPE. The token pipeline has `|| true` and then checks the result is non-empty.
- A progress spinner on stderr plus exit 0 is success. Do not treat spinner text as an error.
- `acm env` suggests a restart only for `running` and `starting`. Inspect failure while the service is down is reported as `absent`, which skips the hint. That is intentional.
