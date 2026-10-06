# dev-talk

[← Back to lgh-skills](../README.md) · [Output style](../plugins/dev-talk/output-styles/dev-talk.md)

## What it does

`dev-talk` makes Claude's replies quick to scan. It's built for readers with ADHD, dyslexia or limited attention, and for anyone who wants the point fast.

**Part 1: easy to digest** (the main focus)

- **BLUF (bottom line up front):** the answer or the action needed is the first line.
- **Key word first:** every line starts with the word that carries its meaning.
- **Short and chunked:** one idea per sentence, at most 20 words, 2 sentences per paragraph.
- **Lists:** numbered for steps, bullets for the rest, 5 items at most.
- **Bold labels only:** `**Cause:**`, `**Fix:**`, never bold inside sentences.
- **One action, one question:** what you need to do is the last line, on its own.
- **Named things, plain words, no filler.**

**Part 2: tone**

- Facts with evidence. Current state only, no plans.
- No praise, emotion, exclamation marks or emoji.
- Disagrees directly. Hedges only with a reason.

The same rules apply to commit messages, PR descriptions and comments, Jira text and code comments.

## Where the rules come from

- **Short sentences, short paragraphs, front-loading, lists for 3+ items:** [W3C, Keep Text Succinct](https://www.w3.org/WAI/WCAG2/supplemental/patterns/o3p05-succinct-text/).
- **Scanning:** [Nielsen Norman Group](https://www.nngroup.com/videos/f-pattern-reading-digital-content/) found 79% of users scan a new page and only 16% read word by word.
- **Bottom line first:** the military's [BLUF](https://animalz.co/blog/bottom-line-up-front) standard.
- **Bold labels, sparingly:** Mayer's signalling principle. Cues reduce distraction, and overuse weakens them.
- **One action at a time:** ADHD reduces working memory, so guidance is to give discrete steps.
- **Plain, literal words:** W3C cognitive accessibility guidance and dyslexia style guides.

Bionic reading (bold first letters of words) is left out. A [Readwise test](https://qz.com/2182666/does-bionic-reading-work-typographers-and-scientists-weigh-in) with 2,074 people found no speed gain.

## Installation

`dev-talk` is a separate plugin in the lgh-skills marketplace, so installing `lgh-skills` doesn't turn it on:

```
/plugin marketplace add lukehillonline/lgh-skills
/plugin install dev-talk@lgh-skills
```

Restart Claude Code after installing. Output styles are read at startup.

## How to use it

It's on by default in every session.

```
/dev-talk:stop     # off for the rest of this session
/dev-talk:start    # back on
```

- **Same effect:** saying "normal mode" or "dev-talk".
- **New session:** always starts with it on.
- **Off permanently:** disable the plugin.

## How it works

- **Output style** ([`output-styles/dev-talk.md`](../plugins/dev-talk/output-styles/dev-talk.md)): the rules.
  - `force-for-plugin: true`: active whenever the plugin is enabled, overriding your `outputStyle` setting.
  - `keep-coding-instructions: true`: Claude Code's normal engineering behaviour is unchanged.
- **SessionStart hook** ([`hooks/hooks.json`](../plugins/dev-talk/hooks/hooks.json)): a one-line reminder at session start, resume, `/clear` and compaction.
- **`start` and `stop` skills** ([`skills/`](../plugins/dev-talk/skills)): user-only (`disable-model-invocation: true`). They work by instruction. The style stays loaded while it's off.

**Without the plugin:** copy `plugins/dev-talk/output-styles/dev-talk.md` to `~/.claude/output-styles/` and run `/output-style dev-talk`.
