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

- One of:
  - macOS 14 or later on Apple silicon. Intel Macs are not supported by `sbx`.
  - Windows 11 on a 64-bit Intel or AMD processor, with Windows Hypervisor Platform turned on. WSL is not covered.
- `sbx` CLI and a Docker account: `brew install docker/tap/sbx` on macOS, `winget install -h Docker.sbx` on Windows, then `sbx login`.

## How to use it

```
/sandbox
```

1. **Preflight:** checks the platform (macOS or Windows), `sbx`, the network policy and existing sandboxes.
2. **Workspace mode:** Direct or Clone.
3. **Create, then skills and plugins:** creates the sandbox with `sbx create`. The sandbox doesn't load `~/.claude`, so it imports your skills with `sbx skills import` and installs your enabled plugins inside the sandbox with `claude plugin install`. Needed once per sandbox.
4. **Sessions:** detects the terminal app you're in, works out how that app opens a new tab running a command (its CLI, scripting or docs), shows you the exact command and, once you agree, opens the sandbox there. Falls back to a new window, then to printing the commands. Can open more sessions the same way.
5. **Host check (Direct mode):** flags Git hooks, Git config and changed scripts that would run on your host.
6. **Stop or remove:** `sbx stop` keeps the sandbox, `sbx rm` deletes it.

## Status

Untested: written from the Docker Sandboxes docs on 2026-10-06, without access to an Apple silicon Mac or a Windows 11 machine. Unverified:

- Opening extra sessions with `sbx exec -it <name> claude`.
- `sbx create` accepting `--clone`, and `sbx exec` working on a sandbox that was created but not yet run.
- Opening tabs: worked out at run time per terminal, so untested by design.
- Plugins installed inside a sandbox persisting across restarts.
- Uncommitted changes being left out of the clone in Clone mode.
