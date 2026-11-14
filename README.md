# paperclip-launch

A skill for installing, launching, and auto-starting a local [Paperclip](https://paperclip.ing) instance.

## What it covers

- Managed CLI install (`paperclipai`)
- Onboarding with the systemd user service and linger
- Health verification
- Repair for embedded Postgres `exit 127` (`libpq.so.5` / `libicuuc.so.60`)
- Port 3100 conflicts and missing service registration
- OpenCode `--dir` compatibility shim for `opencode_local` agents

## Install as a skill

Copy this folder to your skills directory, for example:

```bash
cp -r paperclip-launch ~/.agents/skills/
```

## Trigger phrases

- "install Paperclip"
- "start Paperclip"
- "auto-start on boot"
- "paperclip service won't start"
- "Postgres exit 127"
- "OpenCode --dir error"
