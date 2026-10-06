# dev-talk

[← Back to lgh-skills](../README.md) · [Output style](../plugins/dev-talk/output-styles/dev-talk.md)

## What it does

`dev-talk` changes how Claude writes. It answers first, keeps replies short, answers only what was asked, reports current state rather than future plans, and reports results as facts with evidence. It disagrees directly when the user is wrong, and caps lists at 5 items. It removes praise, enthusiasm, exclamation marks, emoji, preamble and closing pleasantries. Grammar stays normal. The same tone applies to commit messages, PR descriptions and comments, Jira comments and code comments.

## Installation

`dev-talk` is a separate plugin in the lgh-skills marketplace, so installing `lgh-skills` doesn't turn it on:

```
/plugin marketplace add lukehillonline/lgh-skills
/plugin install dev-talk@lgh-skills
```

Restart Claude Code after installing. Output styles are read at startup.

## How to use it

It is on by default in every session.

```
/dev-talk:stop     # off for the rest of this session
/dev-talk:start    # back on
```

Saying "normal mode" or "dev-talk" does the same. A new session always starts with it on. To turn it off permanently, disable the plugin.

## How it works

- **Output style** ([`output-styles/dev-talk.md`](../plugins/dev-talk/output-styles/dev-talk.md)): the rules. It sets `force-for-plugin: true`, so it is active whenever the plugin is enabled and overrides your `outputStyle` setting. It sets `keep-coding-instructions: true`, so Claude Code's normal engineering behaviour is unchanged.
- **SessionStart hook** ([`hooks/hooks.json`](../plugins/dev-talk/hooks/hooks.json)): adds a one-line reminder at session start, resume, `/clear` and compaction.
- **`start` and `stop` skills** ([`skills/`](../plugins/dev-talk/skills)): user-only (`disable-model-invocation: true`). They tell Claude to ignore or follow the style for the rest of the session. The style is still loaded while it is off, so `stop` works by instruction, not by unloading it.

To use it without the plugin, copy `plugins/dev-talk/output-styles/dev-talk.md` to `~/.claude/output-styles/` and run `/output-style dev-talk`.
