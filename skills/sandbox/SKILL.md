---
name: sandbox
description: Set up a Docker Sandboxes (sbx) microVM for the current project so Claude Code can be used as normal — multiple sessions, skills, plugins — with permissions skipped, safely. Use when the user runs /sandbox or asks to run Claude "in a sandbox", "in Docker", "safely on auto mode" or "with skip permissions".
---

# Sandbox

Sets up a persistent Docker Sandboxes microVM (`sbx`) for the current project. Inside it, Claude Code runs with `--dangerously-skip-permissions` (the `sbx` default). The microVM is the security boundary: its own kernel, filesystem, Docker daemon and network. Credentials stay on the host: `sbx` injects them through a proxy.

The user then works as normal: as many sessions as they like, their skills and plugins, any repository provider.

## Rules

- One step at a time. Ask, then stop and wait for the answer.
- `sbx run` and `sbx exec -it` are interactive. You can't drive them from Bash: give the user the command to run in a separate terminal.
- Never mount extra host paths without `:ro`.
- Never pass secrets with `-e` or mounted files. Use `sbx secret set`.

## 1. Preflight

Run these and stop at the first failure, telling the user what failed and how to fix it:

1. `uname -sm` to find the platform. Claude Code on Windows reports `MINGW64_NT-...` or `MSYS_NT-...` (Git Bash). This skill supports macOS and native Windows only: for anything else, including WSL, stop and say so.
   - **macOS:** run `sw_vers -productVersion` and `sysctl -n machdep.cpu.brand_string`. `sbx` needs macOS 14 or later and Apple silicon. On an Intel Mac, stop: `sbx` isn't supported there.
   - **Windows:** run `powershell.exe -NoProfile -Command "(Get-CimInstance Win32_OperatingSystem).Caption"`. `sbx` needs Windows 11 on a 64-bit Intel or AMD processor, so stop on Windows 10 or Arm. It also needs Windows Hypervisor Platform. Checking it needs admin, so tell the user to make sure it's on by running this in an elevated PowerShell (safe if it's already on; a restart may be needed):
     ```powershell
     Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All
     ```
2. `command -v sbx`. If it's missing, give the install commands for the platform and stop.
   - **macOS:**
     ```bash
     brew trust docker/tap
     brew install docker/tap/sbx
     sbx login
     ```
   - **Windows** (PowerShell, per-user, no admin needed). Open a new terminal afterwards so `sbx` is on `PATH`:
     ```powershell
     winget install -h Docker.sbx
     sbx login
     ```

   On first use, `sbx` asks for a global network policy. Tell the user to pick **Balanced** (default deny, common dev hosts allowed).
3. `sbx policy ls`. Show the user the rules. If the policy allows all traffic (Open), stop and tell them to switch to Balanced or Locked Down. If you can't tell from the output, ask.
4. `sbx ls`. If a sandbox for this project already exists (step 2 says how they're named), ask with AskUserQuestion whether to reuse it (go to **4**) or set up a new one.

## 2. Choose the workspace mode

Ask with AskUserQuestion, showing both descriptions:

- `Direct`: the agent edits your real checkout, and your IDE sees changes live. Works like normal. Risk: the agent can write files that later run on your host, outside the sandbox (`.git/hooks`, `.git/config`, `package.json` scripts, `.envrc`, `Makefile`, `.vscode/tasks.json`, `.ps1`/`.bat`/`.cmd` scripts). Check them before running anything on the host (step **5**).
- `Clone`: the agent works on a private Git clone inside the sandbox, and your checkout is mounted read-only. Your host can't be touched. You see the agent's work by fetching from the `sandbox-<name>` remote. Needs a Git repo.

The mode is fixed when the sandbox is created. Name the sandbox `<folder>` for Direct and `<folder>-clone` for Clone, where `<folder>` is the current directory's name.

For Clone:

- Run `git rev-parse --show-toplevel`. If this isn't a Git repo, stop and offer Direct instead.
- Run `git status --porcelain`. Uncommitted changes may not reach the clone. If there are any, ask with AskUserQuestion whether to stop so the user can commit, or carry on without them. Never commit, stash or discard them yourself.

## 3. Skills and plugins

The sandbox doesn't load the host's `~/.claude` (`%USERPROFILE%\.claude` on Windows, which is also `~/.claude` in Git Bash): user-level `CLAUDE.md`, settings, hooks, output styles and plugins stay on the host. Project-level `.claude/` in the workspace is available. Tell the user this.

**Skills:** give the user `sbx skills import` to copy skills from the host. Run `sbx skills import --help` first and use the syntax it shows.

**Plugins:** read `~/.claude/plugins/known_marketplaces.json` and `enabledPlugins` in `~/.claude/settings.json`. Build the commands to reinstall the enabled plugins inside the sandbox:

```
/plugin marketplace add <owner/repo or git URL>
/plugin install <plugin>@<marketplace>
```

Marketplaces with a local `directory` source aren't reachable from the sandbox. List them as skipped. Show the commands for the user to run inside the first session (step **4**), then `/reload-plugins`. The sandbox keeps them, so this is only needed once per sandbox.

**Other credentials:** if the user's work needs an API key (for example a model provider), have them store it with `sbx secret set`. Run `sbx secret ls` to show what's already stored.

## 4. Start sessions

Give the user the commands to run in separate terminals (Terminal on macOS, PowerShell on Windows), from the project root. They're the same on both:

```bash
# Create the sandbox and start the first session
sbx run --name <name> claude .            # Direct
sbx run --clone --name <name> claude .    # Clone

# Return to an existing sandbox
sbx run <name>

# More sessions in the same sandbox
sbx exec -it <name> claude --dangerously-skip-permissions
```

Tell them:

- Run `/login` in the first session if Claude asks for it. The token stays on the host.
- If a session needs a blocked host, check `sbx policy log` and allow it only if it's expected: `sbx policy allow network <host>`.
- In Clone mode, fetch the agent's work with `git fetch sandbox-<name>`. This only works while the sandbox is running.

## 5. Before running anything on the host (Direct mode)

When the user comes back after a Direct-mode session, or asks to check, look for files that would run on the host:

1. `ls -la .git/hooks` and flag every file not ending in `.sample`.
2. `git config --local --list`, flagging `core.hooksPath`, `core.fsmonitor`, `core.sshCommand`, `*.helper` and any `alias.*` starting with `!`.
3. `git status --porcelain` and `git diff`, flagging changes to `package.json` scripts, lockfiles, `.envrc`, `Makefile`, `.vscode/`, `.idea/`, Dockerfiles, CI config, and shell, `.ps1`, `.bat` or `.cmd` scripts.

Show what you found. Never delete or revert anything without an explicit "yes".

## 6. Stop or remove

Sandboxes persist, including installed plugins and packages. When the user is done:

- `sbx stop <name>` stops it and keeps everything.
- `sbx rm <name>` deletes it. In Clone mode, anything not fetched is lost, and the `sandbox-<name>` remote is removed from the host repo. Confirm before running it.
