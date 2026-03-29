# adk-builder-ext

A Gemini CLI extension that gives you an expert ADK (Agent Development Kit)
builder skill. Install it once, then vibe-code any ADK agent through natural
language — no manual scaffolding required.

## Install

```bash
gemini extensions install https://github.com/cuppibla/adk-builder-ext
```

## Usage

Start Gemini CLI inside your project folder:

```bash
gemini
```

Then just describe what you want to build:

```
gemini> I want to build an ADK agent that helps users track their daily tasks.
        It should be able to add tasks, list them, and mark them as done.
```

Gemini CLI will detect the ADK context, activate the `adk-builder` skill, and
generate all project files for you. No manual coding needed.

## What the skill does

When activated, the skill instructs Gemini CLI to:

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
gemini> Add a tool that sets a due date on a task
gemini> Add error handling to all tools
gemini> Make the coordinator smarter about which sub-agent to use
gemini> @./my-agent/agent.py — write a docstring for every function missing one
```

## Requirements

- [Gemini CLI](https://github.com/google-gemini/gemini-cli)
- [Google ADK](https://github.com/google/adk-python) (`pip install google-adk`)
- Python 3.11+
- A free Gemini API key from [Google AI Studio](https://aistudio.google.com)
