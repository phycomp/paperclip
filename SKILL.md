---
name: paperclip-launch
description: >
  Install, start, and auto-start Paperclip on this machine. Use when asked to
  install Paperclip, launch it, keep it running after logout or reboot, or
  repair a local Paperclip service that will not start. Covers the managed CLI
  install, onboarding with the systemd user service, linger, health checks,
  embedded Postgres exit 127, and the OpenCode --dir startup failure.
---

# Paperclip Launch

Paperclip is two layers: the managed `paperclipai` CLI, and a background service that owns the API, embedded database, and heartbeat scheduler. A launch is complete when the service is active, enabled at boot, and healthy.

Run every command as the same desktop user that will own the instance.

## 1. Preflight

```bash
node --version        # engine requirement is >=24.11.0
npm --version
systemctl --user is-system-running
command -v opencode   # only needed for opencode_local agents
export PATH="$HOME/.local/bin:$PATH"
```

Done when Node meets the engine requirement and the systemd user manager reports `running` or `degraded` (both can host the service; `offline` cannot).

## 2. Install the managed CLI

```bash
npx --yes paperclipai@latest install -y
paperclipai --version
```

This writes the binary to `~/.local/bin/paperclipai` and the payload under `~/.paperclip/cli/installs/npm/<version>`. Use the managed install rather than a bare global npm install; `paperclipai service` and updates expect this layout.

Done when `paperclipai --version` prints a version and the payload path, and `~/.local/bin/paperclipai` exists.

## 3. Onboard and register the service

```bash
paperclipai onboard --yes --install-service
```

This writes `~/.paperclip/instances/default/config.json`, creates the embedded database, registers and starts `paperclipai.service`.

Do not also run `paperclipai run`; that is foreground mode and refuses to start while the service is active unless forced.

Done when `paperclipai service status` reports:

- `"installed": true`
- `"active": true`
- `"enabled": true`
- `"linger": true`
- `"health": { "ok": true }`

## 4. Verify the launch

```bash
paperclipai service status
curl -s http://127.0.0.1:3100/api/health
```

If `linger` is false, enable startup without a login session:

```bash
loginctl enable-linger "$USER"
```

Done when the service is active and enabled, `health.ok` is true, and the health endpoint returns `"status":"ok"` with the server version.

## If the service is not active

Read the unit log first:

```bash
journalctl --user -u paperclipai.service --no-pager -n 100
```

### Exit 127, or `libpq.so.5` / `libicuuc.so.60` not found

The embedded Postgres package was installed without its postinstall symlinks. Repair once, then restart:

```bash
node "$HOME/.paperclip/cli/current/node_modules/@embedded-postgres/linux-x64/scripts/hydrate-symlinks.js"
systemctl --user restart paperclipai.service
paperclipai service status
```

Run the same repair after a CLI update if the symptom returns.

### Port 3100 already in use

```bash
ss -ltnp | grep ':3100'
```

Stop the other process before restarting. Two instances sharing one data directory corrupt state.

### Service not registered

```bash
paperclipai service install --enable-linger
paperclipai service start
```

## OpenCode adapter shim

Only needed when agents use `opencode_local` and the installed OpenCode rejects `--dir`. Check:

```bash
opencode run --help 2>&1 | grep -q -- '--dir' && echo "supported" || echo "shim needed"
```

Paperclip always appends `--dir <workspace>` to `opencode run`. If the flag is unsupported, wrap the real binary and turn the flag into a `cd`:

```bash
REAL_OPENCODE="$(command -v opencode)"
mkdir -p "$HOME/.paperclip/bin"
cat > "$HOME/.paperclip/bin/opencode" <<'EOF'
#!/bin/bash
args=()
dir=""
while [[ $# -gt 0 ]]; do
  case "$1" in
    --dir) dir="${2:-}"; shift 2 ;;
    --dir=*) dir="${1#--dir=}"; shift ;;
    *) args+=("$1"); shift ;;
  esac
done
if [[ -n "$dir" ]]; then cd "$dir" || exit 1; fi
exec "@REAL_OPENCODE@" "${args[@]}"
EOF
sed -i "s|@REAL_OPENCODE@|$REAL_OPENCODE|" "$HOME/.paperclip/bin/opencode"
chmod +x "$HOME/.paperclip/bin/opencode"
```

Capture `REAL_OPENCODE` before `~/.paperclip/bin` is on `PATH`, so it points at the actual OpenCode binary rather than this shim.

Point each existing `opencode_local` agent at the shim:

```bash
paperclipai agent update <agentId> \
  --payload-json "{\"adapterConfig\":{\"command\":\"$HOME/.paperclip/bin/opencode\"}}"
```

Done when one heartbeat run reaches `succeeded`:

```bash
paperclipai agent heartbeat:invoke <agentId> --json
paperclipai run get <runId> --json
```

## Paths

| What | Where |
| --- | --- |
| CLI binary | `~/.local/bin/paperclipai` |
| Active CLI payload | `~/.paperclip/cli/current` |
| Instance config | `~/.paperclip/instances/default/config.json` |
| Embedded database | `~/.paperclip/instances/default/db` (Postgres port 54329) |
| Logs | `~/.paperclip/instances/default/logs` |
| Service unit | `~/.config/systemd/user/paperclipai.service` |
| API | `http://127.0.0.1:3100` |

This skill ends at a healthy launch. Company, agent, and task setup belong to the `paperclip` skill.
