# claude-onshape-skill

An agent skill for working in [Onshape](https://cad.onshape.com) through the [Claude in Chrome](https://claude.com/chrome) browser tools.

It's a set of field notes, not code. It covers keyboard shortcuts, sketch and dimension recipes, feature dialogs, assemblies and mates, animation, sheet metal and drawings. It also explains how to read exact geometry from Onshape's REST API inside the page, so the agent doesn't have to guess numbers from screenshots.

## Install

```sh
npx skills add Sergej-Popov/claude-onshape-skill
```

Or copy `SKILL.md` into `~/.claude/skills/claude-onshape-skill/`.

## Requirements

- Claude Code (or another agent that supports skills) with the Claude in Chrome extension connected.
- An Onshape account, signed in to Chrome. The skill uses only your browser session. It needs no API keys, and its API calls are read-only.

## Use

Open an Onshape document in Chrome and ask the agent to model, change or inspect something. The skill loads by itself when the task involves Onshape.

## Limitations

- UI coordinates in the notes are approximate and depend on the window size.
- Onshape changes its UI over time. Some recipes may need adjusting.

## About

Claude Opus 5.5 wrote this skill entirely, from notes it took while working in real Onshape documents. It's shared as is: check the agent's work in your documents before you rely on it.