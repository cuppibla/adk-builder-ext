# adk-builder-ext

An expert ADK (Agent Development Kit) builder skill for vibe-coding Google ADK
agents through natural language. Works with both Gemini CLI and Claude Code —
install it once, then describe what you want to build.

## Gemini CLI

### Install

```bash
gemini extensions install https://github.com/cuppibla/adk-builder-ext --auto-update
```

To update manually (if installed without `--auto-update`):

```bash
gemini extensions update https://github.com/cuppibla/adk-builder-ext
```

To uninstall:

```bash
gemini extensions uninstall https://github.com/cuppibla/adk-builder-ext
```

### Usage

Start Gemini CLI inside your project folder:

```bash
gemini
```

Then just describe what you want to build:

```
I want to build an ADK agent that helps users track their daily tasks.
It should be able to add tasks, list them, and mark them as done.
```

Gemini CLI will detect the ADK context, activate the `adk-builder` skill, and
generate all project files for you. No manual coding needed.

---

## Claude Code

### Option 1 — Permanent install (recommended)

Run this once to make `/adk-builder` available in every project on your machine:

```bash
mkdir -p ~/.claude/commands && curl -o ~/.claude/commands/adk-builder.md \
  https://raw.githubusercontent.com/cuppibla/adk-builder-ext/main/.claude/commands/adk-builder.md
```

Then start Claude Code in any project folder and invoke the skill:

```
/adk-builder
```

Describe what you want to build and Claude will generate all project files for you.

### Update

To pick up the latest version, re-run the same curl command above.

To remove the skill:

```bash
rm ~/.claude/commands/adk-builder.md
```

### Option 2 — Load for current session (no setup)

Paste this into any Claude Code chat to load the skill without installing anything:

```
Fetch https://raw.githubusercontent.com/cuppibla/adk-builder-ext/main/.claude/commands/adk-builder.md and use it as your instructions. Then help me build an ADK agent.
```

The skill is active for the rest of that session.

---

## What the skill does

When activated, the skill instructs the AI to:

- Choose the right agent type (single agent, sequential, parallel, loop, or coordinator)
- Select the right model per agent based on what each agent actually needs to do
- Generate a correctly structured ADK project (`agent.py`, `__init__.py`, `.env`)
- Write tools as properly typed Python functions with full docstrings
- Define the `root_agent` with an instruction that lists all tools
- Follow consistent return format (`{"status": "success/error", ...}`)
- Validate the output and tell you how to run it with `adk web`

## Vibe-coding workflow

After the initial generation, keep iterating through conversation:

```
Add a tool that sets a due date on a task
Add error handling to all tools
Make the coordinator smarter about which sub-agent to use
```

## Requirements

- [Google ADK](https://github.com/google/adk-python) (`pip install google-adk`)
- Python 3.11+
- A free Gemini API key from [Google AI Studio](https://aistudio.google.com)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) (Gemini CLI only)
- [Claude Code](https://claude.ai/code) (Claude Code only)
