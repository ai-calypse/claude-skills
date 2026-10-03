---
name: project-plugins
description: Pick which yaksh-skills plugins a project needs (clerk, cloudflare, flutter, mobile, web-design, media, ai-dev, rust, pinokio, productivity) and install them at project scope. Use at the start of work in a project, when the user asks which skills or plugins to install, or when a task needs a skill that is not loaded.
---

# Project plugins

The plugin guide is the README of the `yaksh-skills` marketplace. Read it first and follow its "For Claude" section:

- Local copy: `~/Documents/Development/claude-skills/README.md`
- If that file does not exist: https://raw.githubusercontent.com/ai-calypse/claude-skills/main/README.md

In short:

1. Check the project's files and dependencies against the README's Signals column.
2. Run `claude plugin list` and skip plugins that are already installed.
3. Tell the user which plugins match and why. Wait for their OK.
4. Install each one with `claude plugin install <plugin>@yaksh-skills --scope project`.
5. Tell the user to restart Claude Code so the new skills load.

Never install anything except `core` at user scope. If nothing matches, say so and install nothing.
