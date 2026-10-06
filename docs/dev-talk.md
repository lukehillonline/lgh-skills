# dev-talk

[← Back to lgh-skills](../README.md) · [Output style](../output-styles/dev-talk.md)

## What it does

`dev-talk` changes how Claude writes. It answers first and reports results as facts with evidence. It disagrees directly when the user is wrong, and caps lists at 5 items. It removes praise, enthusiasm, exclamation marks, emoji, preamble and closing pleasantries. Grammar stays normal. The same tone applies to commit messages, PR descriptions and comments, Jira comments and code comments.

## How it works

- **Output style** ([`output-styles/dev-talk.md`](../output-styles/dev-talk.md)): the rules. It sets `force-for-plugin: true`, so it is active whenever the plugin is enabled and overrides your `outputStyle` setting. It sets `keep-coding-instructions: true`, so Claude Code's normal engineering behaviour is unchanged.
- **SessionStart hook** ([`hooks/hooks.json`](../hooks/hooks.json)): adds a one-line reminder at session start, resume, `/clear` and compaction.

Say "normal mode" to turn it off for the rest of a session, and "dev-talk" to turn it back on. To turn it off permanently, disable the plugin.

The manual symlink install doesn't include it. To use it without the plugin, copy `output-styles/dev-talk.md` to `~/.claude/output-styles/` and run `/output-style dev-talk`.
