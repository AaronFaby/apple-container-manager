# acm — Apple Container Manager

`acm` is a single bash script that runs several [OpenClaw](https://github.com/AaronFaby/openclaw-docker-image) gateways on one Mac using [Apple's `container` CLI](https://github.com/apple/container). It is the Apple-container counterpart of [ocm](https://github.com/AaronFaby/openclaw-docker-manager). Same workflow, same volume layout, no Docker.

Each managed container is a small Linux VM. `acm` keeps a private env file for it, four named volumes for its data, and a published host port for the gateway. Upgrades replace the VM and keep the volumes.

The script is the whole program. There is no daemon of its own and no package install step.

## Features

- Create, start, stop, restart, and remove many OpenClaw containers
- Persistent home, config, workspace, and auth data across restarts and image upgrades
- Per-container environment variables, including a generated gateway token
- Automatic host-port assignment that skips ports already in use
- Logs, status, an interactive shell, and one-off commands
- Image pull and upgrade of one container or all of them

## Requirements

- A Mac with Apple silicon
- macOS 26 or later (what `container` supports)
- The [`container`](https://github.com/apple/container) CLI on `PATH`. Homebrew is the usual install: `brew install container`
- Bash 3.2 or newer (the bash that ships with macOS is enough)
- `jq` or `python3` if you want IP address and start time in `acm status`. Name, port, and state still print without either

`acm` does not start the `container` services for you. `container system start` can ask for an administrator password and install a kernel. Run it yourself once per login:

```bash
container system start
```

## Install

```bash
git clone https://github.com/AaronFaby/apple-container-manager.git
cd apple-container-manager
chmod +x acm
```

Config and env files live next to the `acm` script, not in the directory you happen to be in. The script follows symlinks to find itself, so this works:

```bash
ln -s "$(pwd)/acm" "$HOME/bin/acm"
```

Put `$HOME/bin` on your `PATH`. `acm.conf` and `containers/` stay in the clone.

## Quick start

```bash
container system start

acm create main
acm env main            # set API keys, then save
acm setup main          # interactive OpenClaw setup, once
acm start main

acm list
acm status main
acm logs main -f
```

Open the gateway at `http://127.0.0.1:<port>` using the port from `acm list`. The port inside the container is always `18789`.

The guest hostname is the container id, `acm-main`. Apple's CLI has no separate hostname flag.

Run `acm setup` before you expect the gateway to stay up. Without a config, OpenClaw exits on startup and, because `container` has no restart policy, the VM stays stopped. `acm start` waits a moment and warns you when that happens. `acm logs main` shows the reason.

## How a container is stored

`acm create main` does not boot a VM. It writes `containers/main.env` (mode `0600`) and creates four volumes. `acm start main` boots the VM.

| Piece | Name |
|---|---|
| Env file | `containers/main.env` |
| Container id | `acm-main` |
| Volumes | `acm-main-home`, `acm-main-config`, `acm-main-workspace`, `acm-main-auth` |
| Image | `ghcr.io/aaronfaby/openclaw-custom:latest` unless you set a tag |
| Publish | `<host port>:18789` |

The prefix `acm` comes from `ACM_CONTAINER_PREFIX`. Change it in `acm.conf` before you create containers if you want a different prefix. Names already created keep the prefix they were given.

## Commands

Run `acm help` for the short list. The sections below are the behavior behind each command.

### `create`

```bash
acm create <name> [--tag TAG] [--port PORT]
```

Creates the env file and the four volumes. Does not pull the image and does not start the VM.

- `<name>` is 1–64 characters: letters, digits, and hyphens, and it must start and end with a letter or digit.
- `--tag` defaults to `ACM_DEFAULT_TAG` (`latest`). It is stored as `ACM_TAG` and used by later start, setup, run, and upgrade commands.
- `--port` is the host port. Omit it and `acm` takes the highest `ACM_PORT` already in an env file, adds one (starting from `ACM_PORT_START`), then skips any port another process is listening on.
- An explicit port must be 1–65535 and must not belong to another managed container or a current listener (`lsof`, or `nc` if `lsof` is missing).
- A gateway token is written to `OPENCLAW_GATEWAY_TOKEN`.
- If a `container` named `acm-<name>` already exists, create refuses. It will reuse volumes that are already there, which is how a retry works after a partial failure.
- Create fails if `containers/<name>.env` already exists. Edit that file with `acm env` instead.

```bash
acm create main
acm create staging --port 19000
acm create pinned --tag oc-2026.5.22
```

### `setup`

```bash
acm setup <name>
```

Runs `node openclaw.mjs setup` in a new interactive VM (`container run -it --rm`) with this container's volumes, image tag, resource limits, and env file. The VM is deleted when setup exits. The volumes are not.

You need a terminal. Do this once before the first real start. Setup can take a minute while the VM boots.

### `start`

```bash
acm start <name>
```

- `running`: prints that it is already running and does nothing.
- `absent` (no VM yet): boots one.
- any other state, including `stopped`: stops it if needed, deletes the VM, and boots a new one. That is how a changed env file, tag, or port takes effect. Volumes are kept.

The boot command is fixed:

```bash
container run -d \
  --name acm-<name> \
  --user node \
  --cpus "$ACM_CPUS" \
  --memory "$ACM_MEMORY" \
  -p "$ACM_PORT:18789" \
  -v acm-<name>-home:/home/node \
  -v acm-<name>-config:/home/node/.openclaw \
  -v acm-<name>-workspace:/home/node/.openclaw/workspace \
  -v acm-<name>-auth:/home/node/.config/openclaw \
  --env-file <filtered env> \
  "$ACM_IMAGE:$ACM_TAG" \
  node openclaw.mjs gateway --bind "$ACM_GATEWAY_BIND"
```

Mounts are applied parent-first so the nested volumes overlay `/home/node`. The image entrypoint stays in place; those arguments replace the image's default command.

`container run -d` returns when the VM is up, which can be before OpenClaw has decided to stay running. `acm` waits two seconds and then checks. If the container is still `running` or `starting`, start succeeds. If it has already stopped, start prints a warning and points you at `acm logs`.

The image is pulled automatically when it is not local.

### `stop`

```bash
acm stop <name>
acm stop <name> --force
```

Sends `container stop --time 10` (SIGTERM, then kill after 10 seconds). If the container is already stopped or has never been created, stop tells you and exits 0. Volumes and the env file stay.

`--force` is for a guest that is no longer answering. `container stop` and `acm cli` both wait on the guest agent, so a wedged VM (the usual cause is the guest running out of memory) makes them hang past the 10 second limit. `--force` SIGKILLs the host `Virtualization.VirtualMachine` process that has this container's disk open, then records the stop. Other containers are left alone. Volumes and the env file stay. This is a hard power-off, so use it when a normal stop does not return.

### `restart`

```bash
acm restart <name>
acm restart <name> --force
```

Deletes the VM if one exists, then boots a new one with the current env file, tag, and port. Same warning as start if the process exits immediately. Use this after `acm env` if the container was running.

`--force` force-stops a wedged guest first, the same way `acm stop --force` does, then boots a new VM on the existing volumes. A container that is only slow is killed too. A stopped container is booted without a kill.

### `rm`

```bash
acm rm <name> [--volumes] [--force]
acm remove <name>
acm delete <name>
```

Deletes the VM and the env file. Volumes are kept unless you pass `--volumes`.

- A `running`, `starting`, or `stopping` container is refused unless you pass `--force` or `-f`.
- `--volumes` asks for confirmation on a terminal. `--force` skips the question. With no terminal and no `--force`, the volumes are left in place and the env file is still removed.
- Volume deletion is permanent.

```bash
acm stop main
acm rm main                  # keep data
acm rm main --volumes        # ask, then delete data
acm rm main --force --volumes
```

### `list` and `status`

```bash
acm list
acm ls
acm status
acm status <name>
```

`list` prints name, state, and host port for every env file.

`status` with no name adds the IPv4 address for containers that are up. `status <name>` also prints the container id, image, port mapping, CPU and memory settings from `acm.conf`, start time, and volume names.

State words:

| State | Meaning |
|---|---|
| `not created` | Env file exists, no VM |
| `running` | VM is up |
| `starting` / `stopping` | Transition |
| `stopped` | VM exists and the process has exited |

There is no health column. `container` does not report a Docker-style healthcheck. IP and start time need `jq` or `python3`. Without them, `acm` still prints name, port, and state, and says the parser is missing.

### `logs`

```bash
acm logs <name> [-f] [-n LINES]
acm logs <name> [--follow] [--tail LINES]
```

Shows the last 100 lines by default. `-n` is passed to `container logs -n`. The long flag `--tail` is accepted as an alias, but the underlying CLI has no `--tail`. `-f` follows until you interrupt it.

Logs work after the process has exited, as long as the VM has not been removed. A container that was never started has no logs.

### `run`

```bash
acm run <name> [command...]
```

Boots a throwaway interactive VM with the same volumes, env, image, CPUs, and memory as `start`. Nothing is published. The VM is removed on exit (`--rm`). Volumes stay.

With no command, you get `/bin/bash`. Otherwise the arguments are the command:

```bash
acm run main
acm run main node openclaw.mjs doctor
```

This needs a terminal because it allocates a TTY.

### `cli`

```bash
acm cli <name> [command...]
acm exec <name> [command...]
acm shell <name>
```

Runs inside the existing VM. The container must be `running`.

- No command: interactive `/bin/bash`. Requires a terminal.
- A command: `node openclaw.mjs <command...>`, so `acm cli main doctor --fix` runs OpenClaw.
- `--raw`: run the arguments as the process, with no `openclaw` prefix. `acm cli main --raw echo ok`.

Stdin stays open. A TTY is allocated only when both stdin and stdout are terminals, so pipes work.

### `env`

```bash
acm env <name>
```

Opens `containers/<name>.env` in `$EDITOR`, or `vi` if `EDITOR` is unset. `EDITOR` may include arguments (`code -w`).

After you save, `acm` removes one matching pair of surrounding quotes from each value and sets the file back to mode `0600`. If the container is `running` or `starting`, it tells you to restart. Env edits do not apply to a VM that is already running.

### `pull`

```bash
acm pull [tag]
```

Pulls `ACM_IMAGE:<tag>`, defaulting to `ACM_DEFAULT_TAG`. If you pass a different tag from a terminal, `acm` asks whether to store it as `ACM_DEFAULT_TAG` in `acm.conf`. A non-interactive shell leaves the default tag alone.

### `upgrade`

```bash
acm upgrade <name> [--tag TAG]
acm upgrade --all [--tag TAG]
```

Pulls the image first. With `--tag`, that tag is written into each selected env file (the `ACM_TAG=` line is added back if it was deleted).

Then:

- A container that was `running` is recreated and started on the new image.
- A container that was stopped, or never started, keeps the new tag and stays down. The next `acm start` uses it.

`--all` cannot be combined with names. Upgrade refuses to run when there is nothing to upgrade.

```bash
acm upgrade main
acm upgrade --all
acm upgrade main --tag oc-2026.5.22
```

### `help` and `version`

```bash
acm help
acm version
```

`help`, `--help`, and `-h` print the short help. `version` and `--version` print `acm v1.0.0`. An unknown command prints an error, the help, and exits 1.

## Configuration

### `acm.conf`

Sourced on every invocation. Values already exported in your shell are ignored, so the file is the source of truth. If the file is missing, `acm` writes the defaults below. Keys you delete fall back to those same defaults.

```bash
ACM_IMAGE=ghcr.io/aaronfaby/openclaw-custom
ACM_DEFAULT_TAG=latest
ACM_PORT_START=18789
ACM_CONTAINER_PREFIX=acm
ACM_MEMORY=4g
ACM_CPUS=4
ACM_GATEWAY_BIND=lan
ACM_VOLUME_HOME_SIZE=16g
ACM_VOLUME_CONFIG_SIZE=16g
ACM_VOLUME_WORKSPACE_SIZE=8g
ACM_VOLUME_AUTH_SIZE=1g
```

| Key | Used for |
|---|---|
| `ACM_IMAGE` | Registry image, without a tag. The tag is per container. |
| `ACM_DEFAULT_TAG` | Tag for new containers, and for `acm pull` when you omit one. |
| `ACM_PORT_START` | First host port auto-assignment will consider. |
| `ACM_CONTAINER_PREFIX` | Prefix for container ids and volume names. |
| `ACM_MEMORY` | `--memory` for every VM this tool starts. A number plus an optional `K`, `M`, `G`, `T`, or `P` suffix. |
| `ACM_CPUS` | `--cpus`. A positive integer. |
| `ACM_GATEWAY_BIND` | Passed as `openclaw gateway --bind`. Default `lan`. |
| `ACM_VOLUME_*_SIZE` | Cap passed to `container volume create -s` the first time that volume is created. Changing it later does not resize an existing volume. |

`container` itself defaults a new VM to 1 GiB and 4 CPUs, and a new volume to a 512 GiB cap. The memory and volume values above are there because those defaults are a poor fit for OpenClaw. Memory and CPU in `acm.conf` apply the next time a VM is created (`start` of a stopped container recreates it). They do not resize a VM that is already running until you restart it.

`ACM_GATEWAY_BIND=lan` is required for the published port to work. The image command listens on loopback unless told otherwise, and `container`'s port publishing reaches the VM's non-loopback address.

### `containers/<name>.env`

```bash
# --- ACM internal (do not remove) ---
ACM_PORT=18789
ACM_TAG=latest

# --- Gateway auth (auto-generated) ---
OPENCLAW_GATEWAY_TOKEN=...

# --- User environment variables ---
ANTHROPIC_API_KEY=sk-ant-...
XAI_API_KEY=xai-...
OPENAI_API_KEY=sk-...
TZ=America/Los_Angeles
```

Lines that are blank, start with `#`, or start with `ACM_` are not passed into the VM. Everything else is passed with `--env-file`.

Do not quote values. `--env-file` does not unquote, so `ANTHROPIC_API_KEY="sk-ant-..."` would send the quote marks as part of the key. `acm env` strips one surrounding pair of double or single quotes after you save. A lone quote, or mismatched quotes, is left as-is.

`ACM_PORT` and `ACM_TAG` are required for normal operation. If `ACM_PORT` is missing, start refuses and tells you to put it back. If `ACM_TAG` is missing, the container uses `ACM_DEFAULT_TAG`.

These files hold secrets. They are mode `0600`, and `containers/*.env` is gitignored.

## Data persistence

| Volume | Mount | Default cap | What it holds |
|---|---|---|---|
| `acm-<name>-home` | `/home/node` | 16g | Home directory for tools that do not have a more specific mount |
| `acm-<name>-config` | `/home/node/.openclaw` | 16g | OpenClaw config and auth profiles |
| `acm-<name>-workspace` | `/home/node/.openclaw/workspace` | 8g | Sessions and workspace files |
| `acm-<name>-auth` | `/home/node/.config/openclaw` | 1g | Keys that encrypt OAuth tokens |

The mounts nest: home contains config, config contains workspace, and home also contains auth. They are listed parent-first on `container run` so each narrower mount covers the directory inherited from the volume above it. A fresh `home` volume is filled from the image's `/home/node` the first time it is used.

`acm rm` does not delete volumes unless you pass `--volumes`. `acm upgrade` and `acm restart` never delete them.

The size is a cap on a sparse disk image, set only at `volume create`. It is not how much disk is full.

## Networking

Each VM joins the default `container` network and gets an address you can see with `acm status <name>`. The gateway port is published on all host interfaces:

```text
host ACM_PORT  ->  container 18789/tcp
```

Pick an explicit port when `18789` is already taken, which it often is if other OpenClaw gateways are running:

```bash
acm create dev --port 19000
```

Auto-assignment looks at env files first, then at listeners on this Mac. It does not look at ports reserved only inside other VMs.

## Upgrading the image

```bash
acm pull
acm upgrade --all
```

`pull` only downloads. `upgrade` downloads, then recreates VMs that were running. Stopped containers pick up the new tag on the next start.

Pin one container without moving the others:

```bash
acm upgrade main --tag oc-2026.5.22
```

## Several containers

```bash
acm create production          # first free port at or above 18789
acm create staging             # next free port
acm create dev --port 19000
acm list
acm upgrade --all
```

Each name gets its own env file, token, volumes, and VM. They do not share a network alias beyond what `container` gives every VM (the container id).

## Limits of Apple's container CLI

These are properties of `container` 1.7.1, not choices `acm` can paper over:

- No restart policy. A process that exits stays exited. A reboot or `container system stop` does not bring containers back. Run `acm start <name>` again. `acm` does not install a launchd agent.
- No `--hostname`. The guest hostname is the container id (`acm-main`).
- No Docker-style health status, and no pause.
- `container system status` can report the apiserver as unregistered while `container ls` still works. That happens with the Homebrew service. `acm` treats a successful `container ls -a` as "the service is up."
- `acm` will not call `container system start`.

## Containers created outside acm

`acm` only manages names that have an env file under its own `containers/` directory. VMs you started by hand, including any `ocm-*` containers already on the machine, are left alone.

To manage an existing name, set `ACM_CONTAINER_PREFIX` to the prefix those VMs and volumes already use, and add a `containers/<name>.env` whose `ACM_PORT` matches the published host port. Do this before `acm start`. Start deletes and recreates a stopped VM, and it will refuse to replace one that is running until you stop it. Volumes with the expected names are reused rather than recreated, so a matching prefix keeps the data.

## Troubleshooting

**`acm stop` or `acm cli` hangs**

The guest agent is not answering, so those commands wait. Run `acm restart <name> --force` to kill that container's host VM and boot a new one. `acm stop <name> --force` kills it and leaves it down. Volumes are kept either way.

**`container services are not running`**

`container ls -a` failed. Run `container system start`, then retry. Do not trust `container system status` alone on a Homebrew install.

**`Container 'name' is stopped after launch`**

The VM came up and the process exited. Read `acm logs <name>`. The usual first-run message is that OpenClaw has no config: run `acm setup <name>`, then `acm start <name>`.

**Port already in use**

Pass another `--port`, or edit `ACM_PORT` with `acm env` and restart. Listeners owned by other gateways on this Mac count.

**`ACM_PORT is missing`**

Put `ACM_PORT=<port>` back in the env file. `acm` will not guess one for a container that already exists.

**Changes to env or `acm.conf` seem ignored**

A running VM keeps the environment and resource limits it was booted with. `acm restart <name>` recreates it. Volume caps are the exception: they are fixed when the volume is first created.

**Quotes in an API key**

Run `acm env` and save. It strips one surrounding pair. Or edit the file so the value has no wrapping quotes.

**Status has no IP**

Install `jq` or `python3`. The address is also `container inspect acm-<name>` under `status.networks[].ipv4Address`.

## Security

- `containers/*.env` holds the gateway token and any API keys you add. Mode `0600`. Git ignores them.
- `acm.conf` has no secrets in the default file. Do not put keys there. Put them in the per-container env file.
- The gateway is published on `0.0.0.0` at the host port, with `--bind lan` inside the VM. That is what makes the port reachable. Do not expose that port beyond your machine unless you mean to.

## Project layout

```text
acm            the CLI
acm.conf       defaults; rewritten only if you delete the file
containers/    one <name>.env per container; not committed
README.md      this file
AGENTS.md      notes for people and agents changing the script
```

## Related

- Image: [openclaw-docker-image](https://github.com/AaronFaby/openclaw-docker-image), published as `ghcr.io/aaronfaby/openclaw-custom`
- Docker version of this workflow: [openclaw-docker-manager](https://github.com/AaronFaby/openclaw-docker-manager)
- Runtime: [apple/container](https://github.com/apple/container)
