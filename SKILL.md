---
name: paperclip-launch
description: >
  Install, start, and auto-start Paperclip on this machine. Use when asked to
  install Paperclip, launch it, keep it running after logout or reboot, or
  repair a local Paperclip service that will not start. Covers the managed CLI
  install, onboarding with the systemd user service, linger, health checks,
  embedded Postgres exit 127, the OpenCode --dir startup failure, switching
  agent models/providers, and debugging "Free usage limit reached" errors.
---

# Paperclip Launch

Paperclip is two layers: the managed `paperclipai` CLI, and a background service that owns the API, embedded database, and heartbeat scheduler. A launch is complete when the service is active, enabled at boot, and healthy.

Run every command as the same desktop user that will own the instance.

## Optional prerequisite

If you need the env-profile repository first, run the `repo-pull` skill before
installing Paperclip.

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

## Free usage limit and switching models

When an `opencode_local` agent fails with `lastError: "Free usage limit reached. Connect a provider or subscribe to OpenCode Go to keep going, or switch models."`, the OpenCode Console free tier for the account has been exhausted. The agent's model string is fine; the account needs a different provider or a paid model.

### Diagnose

```bash
paperclipai agent runtime-state <agentId> --json | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d.get("lastError")); print("model:", d.get("sessionParamsJson",{}).get("__paperclipConfiguredModel"))'
paperclipai agent configuration <agentId> --json | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d["adapterType"], d["adapterConfig"].get("model"))'
```

Check the OpenCode runtime Paperclip uses (the shim or real binary):

```bash
"$HOME/.paperclip/bin/opencode" --version
"$HOME/.paperclip/bin/opencode" auth list
"$HOME/.paperclip/bin/opencode" models | grep -i beam
```

### Switch every agent to a different model

Update each agent's `adapterConfig.model`. The adapter type stays `opencode_local`; only the model string changes. Use the `INSTANCE/MODEL` form that `opencode models` prints (e.g. `opencode-go/iceman-contributor-free`, `opencode/iceman-free`, `opencode-go/mimo-v2.6-pro`).

```bash
COMPANY_ID=<company-id>
for id in $(paperclipai agent list -C "$COMPANY_ID" | sed -n 's/^id=\([^ ]*\).*/\1/p'); do
  paperclipai agent update "$id" \
    --payload-json '{"adapterConfig":{"model":"opencode-go/iceman-contributor-free"}}'
done
```

Or one agent:

```bash
paperclipai agent update <agentId> \
  --payload-json '{"adapterConfig":{"model":"opencode-go/iceman-contributor-free"}}'
```

Verify:

```bash
for id in <agentId1> <agentId2> <agentId3>; do
  paperclipai agent configuration "$id" --json \
    | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d["name"], "->", d["adapterConfig"].get("model"))'
done
```

### Restart after a model change

```bash
paperclipai service restart
paperclipai service status
```

### Clear stale session state and retry

A changed model leaves the old session id in `runtimeState`. Reset it, then invoke a heartbeat:

```bash
paperclipai agent runtime-state:reset-session <agentId>
paperclipai agent heartbeat:invoke <agentId> --json
```

### Provider options

- `opencode/...` — OpenCode Console free/paid models (free tier has usage limits).
- `opencode-go/...` — OpenCode Go subscription models (separate quota; needs OpenCode Go).
- `claude_local` adapter — requires Claude Code logged in (`claude auth login`) or `ANTHROPIC_API_KEY`.
- `codex_local` adapter — requires Codex logged in (`codex login`) or `OPENAI_API_KEY`.

Switching adapter type requires changing `adapterType` plus `adapterConfig` (command path, model). Only switch when the corresponding CLI is installed and authenticated.

### Common errors

| Error | Cause | Fix |
|---|---|---|
| `Free usage limit reached...` | OpenCode free tier exhausted | Switch model/provider or subscribe to OpenCode Go |
| `opencode run ... --dir` unsupported | Old OpenCode binary | Apply the OpenCode adapter shim in this skill |
| Agent stuck in `error` after model change | Stale session | `runtime-state:reset-session` then `heartbeat:invoke` |

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
