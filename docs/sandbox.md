# sandbox

[← Back to lgh-skills](../README.md) · [SKILL.md](../skills/sandbox/SKILL.md)

## What it does

`sandbox` sets up a [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/get-started/) (`sbx`) microVM for the current project, so you can use Claude Code as normal (multiple sessions, your skills and plugins, any repository provider) with permissions skipped.

What makes that safe:

- **microVM isolation:** the agent gets its own kernel, Docker daemon, filesystem and network.
- **Restrictive network policy:** Balanced or Locked Down, never Open.
- **Credentials on the host:** `sbx` injects them through a proxy, so they never enter the sandbox.

You choose the workspace mode each time you create a sandbox:

- **Direct:** the agent edits your real checkout and your IDE sees changes live. The agent can write files that later run on your host (Git hooks, `package.json` scripts, `.envrc`), so the skill checks them before you run anything.
- **Clone:** the agent works on a private Git clone and your checkout is read-only. You fetch its work from the `sandbox-<name>` remote.

Remaining risk: hosts the network policy allows can still be used to send data out. Use Locked Down plus a short allowlist for untrusted code.

## Requirements

- Apple silicon Mac with macOS 14 or later. Intel Macs are not supported by `sbx`.
- `sbx` CLI and a Docker account (`brew install docker/tap/sbx`, `sbx login`).

## How to use it

```
/sandbox
```

1. **Preflight:** checks the platform, `sbx`, the network policy and existing sandboxes.
2. **Workspace mode:** Direct or Clone.
3. **Skills and plugins:** the sandbox doesn't load `~/.claude`, so it gives you the `sbx skills import` command and the `/plugin` commands to reinstall your enabled plugins. Needed once per sandbox.
4. **Sessions:** gives you the commands to start the sandbox, return to it, and open more sessions in it.
5. **Host check (Direct mode):** flags Git hooks, Git config and changed scripts that would run on your host.
6. **Stop or remove:** `sbx stop` keeps the sandbox, `sbx rm` deletes it.

## Status

Untested: written from the Docker Sandboxes docs on 2026-10-06, without access to an Apple silicon Mac. Unverified:

- Opening extra sessions with `sbx exec -it <name> claude`.
- Plugins installed inside a sandbox persisting across restarts.
- Uncommitted changes being left out of the clone in Clone mode.
