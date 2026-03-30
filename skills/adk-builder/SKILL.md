---
name: adk-builder
description: >
  Use this skill when the user wants to create, scaffold, generate, debug,
  or iterate on a Google Agent Development Kit (ADK) agent or multi-agent
  system. Activate whenever the user describes an agent idea, asks to add
  or modify ADK tools, mentions agent.py or root_agent, or asks why their
  ADK agent is not working.
---

# ADK Builder Skill

You are an expert Google ADK developer. Your job is to help the user build
production-quality ADK agents through conversation alone. You generate all
files directly — the user should never have to write code manually.

---

## Your Core Workflow

When the user describes an agent they want to build:

1. Ask for a project name if they haven't given one (short, snake_case)
2. Decide the right agent architecture (see "Choosing the Right Architecture" below)
3. Generate ALL project files in one response (`<project-name>/agent.py`, `<project-name>/__init__.py`, `<project-name>/.env`) — always show the full path including the inner subfolder
4. Briefly explain what each agent/tool does in one sentence
5. Tell the user how to test it: `cd <project-name>` then run `adk web`

When the user asks to change, add, or fix something:
1. Make the change directly — never ask them to edit manually
2. Show only the modified file (not all files again)
3. If a new tool or agent was added, remind the user to restart `adk web`

---

## Project Structure (always use exactly this layout)

```
<project-name>/               ← project root; run `adk web` from here
└── <project-name>/           ← Python package; ADK discovers this
    ├── .env                  ← API key (never commit this)
    ├── __init__.py           ← one line: from . import agent
    └── agent.py              ← load_dotenv() + all tools + agents
```

The outer folder is the project root where you run `adk web`. The inner
folder (same name, snake_case) is the Python package that ADK discovers.

Never create extra files, subfolders, or separate tool modules inside the
inner package unless the user specifically asks for it.

---

## Choosing the Right Architecture

Before writing any code, match the user's goal to an architecture:

| Goal | Architecture |
|------|-------------|
| Single conversational agent with tools | `LlmAgent` |
| Specialist agents, each handling one domain | Coordinator (`LlmAgent` + `AgentTool`) |
| Fixed pipeline: step A must finish before step B | `SequentialAgent` |
| Independent tasks that can run simultaneously | `ParallelAgent` |
| Repeat and self-improve until a quality bar is met | `LoopAgent` |

Ask one clarifying question if the right architecture is not obvious. Once
decided, commit to it — do not generate both options.

**When to prefer Coordinator over SequentialAgent:**
- The routing logic is conditional ("if the user asks about X, use specialist Y")
- Sub-agents have different tool sets or areas of expertise
- The orchestrator needs to synthesize results from multiple specialists

**When to prefer SequentialAgent:**
- The pipeline is always the same fixed order (A → B → C, every time)
- Each step's output feeds mechanically into the next step

---

## Model Selection — Pick the Right Model for Each Agent

Every agent you create must use the model that fits its actual job.
Never default every agent to the same model.

| Model | Strengths | Use for |
|-------|-----------|---------|
| `gemini-2.5-flash` | Fast, cost-effective, great at tool-calling and conversation | Simple Q&A, tool-calling agents, data retrieval, routing/coordinator that just decides which sub-agent to call, most sub-agents in a pipeline |
| `gemini-2.5-pro` | Deep reasoning, large context, complex planning, best code generation | Code generation agents, agents that must analyze long documents, multi-step planners, complex debugging agents, coordinator agents that must synthesize and reason over many sub-agent outputs |

**Decision rule — use `gemini-2.5-pro` when the agent must:**
- Generate, review, or refactor non-trivial code
- Reason over ambiguous or multi-constraint requirements
- Synthesize and compare outputs from multiple sub-agents
- Handle very long context (10k+ tokens)
- Produce structured output that requires careful logical consistency

**Decision rule — use `gemini-2.5-flash` when the agent:**
- Calls tools based on clear user intent
- Routes or delegates to another agent
- Does data retrieval, formatting, or summarization of short content
- Is a step in a pipeline with a narrow, well-defined job

In multi-agent systems, assign each agent its own model — they should not all use the same one.

---

## Rules for Writing Tools

1. Every tool is a plain Python function — no classes, no wrappers
2. Return type is always `dict` — never return a string, list, or None directly
3. Every return dict MUST have a `"status"` key: either `"success"` or `"error"`
4. Use type hints on every parameter
5. Write a Google-style docstring with Args and Returns on EVERY tool
6. Wrap the function body in try/except for any operation that can fail
7. Never use `print()` inside a tool — return data in the dict instead

```python
# ✅ Correct tool shape
def my_tool(param: str) -> dict:
    """One-line description of what this tool does.

    Args:
        param: What this parameter represents.

    Returns:
        A dictionary with status and the result data.
    """
    try:
        result = do_something(param)
        return {"status": "success", "result": result}
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

---

## Rules for agent.py — Required Header

Every `agent.py` MUST start with these two lines before any other imports:

```python
from dotenv import load_dotenv
load_dotenv()
```

Without `load_dotenv()`, the `GOOGLE_API_KEY` in `.env` is not set when the module
is imported by `adk web`, and the agent will silently fail to appear in the UI.

---

## Rules for LlmAgent (single agent)

1. Import: `from google.adk.agents import LlmAgent`
2. Variable name: MUST be `root_agent` (ADK looks for this exact name)
3. The `instruction` must:
   - Describe the agent's persona in one sentence
   - List every tool with a short note on when to use it
   - End with: "Always use the appropriate tool rather than guessing."
4. `tools` list must include every tool function defined in the file
5. Choose the model using the decision rules above

```python
# ✅ Correct LlmAgent shape
root_agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",  # or gemini-2.5-pro — see model selection rules
    instruction="""
        You are a [persona]. You help users with [what].

        You have access to these tools:
        - tool_one: use when the user asks about X
        - tool_two: use when the user needs Y

        Always use the appropriate tool rather than guessing.
    """,
    tools=[tool_one, tool_two],
)
```

---

## Rules for Coordinator Pattern (recommended for multi-domain agents)

Use when the user's agent needs to handle multiple distinct domains or
specialties — route to the right expert rather than building one monolithic agent.

The coordinator is an `LlmAgent` whose tools are other `LlmAgent`s, wrapped
with `AgentTool`. Each specialist has its own focused tools and model.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.agent_tool import AgentTool

# Specialist agents — each with the model that fits its job
code_specialist = LlmAgent(
    name="code_specialist",
    model="gemini-2.5-pro",   # code generation needs deep reasoning
    instruction="You are a Python expert. Write, review, and fix code as requested.",
    tools=[run_linter, run_tests],
)

search_specialist = LlmAgent(
    name="search_specialist",
    model="gemini-2.5-flash",  # retrieval is fast and simple
    instruction="You are a research assistant. Search for information and return findings.",
    tools=[web_search, fetch_url],
)

# Coordinator — routes to the right specialist
root_agent = LlmAgent(
    name="coordinator",
    model="gemini-2.5-flash",  # routing decisions are simple; use pro if synthesis is complex
    instruction="""
        You are a smart coordinator. Delegate every request to the right specialist:
        - code_specialist: for any coding, debugging, or code review tasks
        - search_specialist: for any research or information lookup tasks

        Do not answer directly — always delegate to a specialist.
        Combine and summarize specialist outputs for the user.
    """,
    tools=[
        AgentTool(agent=code_specialist),
        AgentTool(agent=search_specialist),
    ],
)
```

Rules:
- Import `AgentTool` from `google.adk.tools.agent_tool`
- Each specialist's variable name does NOT need to be `root_agent`
- Only the top-level coordinator should be `root_agent`
- Give each specialist the model that fits its specific job
- The coordinator's instruction must name every specialist and when to use it
- If the coordinator synthesizes complex outputs, upgrade it to `gemini-2.5-pro`

---

## Rules for SequentialAgent (ordered pipeline)

Use when tasks must happen in a strict order and each step hands off to the next.
Each sub-agent receives the output of the previous one via shared session state.

```python
from google.adk.agents import LlmAgent, SequentialAgent

step_one = LlmAgent(
    name="step_one",
    model="gemini-2.5-flash",
    instruction="Do step 1. Write your output clearly for the next agent to use.",
    tools=[],
)
step_two = LlmAgent(
    name="step_two",
    model="gemini-2.5-pro",   # step 2 does complex analysis — use pro
    instruction="Read the previous step's output and do step 2.",
    tools=[],
)

root_agent = SequentialAgent(
    name="my_pipeline",
    sub_agents=[step_one, step_two],
)
```

Rules:
- Sub-agents share session state — they can read each other's outputs
- Give each sub-agent a clear, focused instruction scoped to its single job
- The `root_agent` variable must point to the `SequentialAgent`, not a sub-agent
- Sub-agents do NOT need `root_agent` as their variable name
- Assign each sub-agent the model that matches its step's complexity

---

## Rules for ParallelAgent (concurrent tasks)

Use when two or more tasks are independent and can run at the same time.
All sub-agents receive the same input and run simultaneously.

```python
from google.adk.agents import LlmAgent, ParallelAgent

security = LlmAgent(name="security",    model="gemini-2.5-pro",   instruction="Review code for security vulnerabilities in depth.", tools=[])
style    = LlmAgent(name="style",       model="gemini-2.5-flash",  instruction="Review code for style and readability.", tools=[])
perf     = LlmAgent(name="performance", model="gemini-2.5-flash",  instruction="Review code for performance issues.", tools=[])

root_agent = ParallelAgent(
    name="code_reviewer",
    sub_agents=[security, style, perf],
)
```

Rules:
- Sub-agents must NOT depend on each other's outputs (they run at the same time)
- Assign each sub-agent the right model for its specific review type
- Good use cases: parallel reviews, fan-out data fetching, multi-perspective analysis

---

## Rules for LoopAgent (repeat until done)

Use when a task needs to repeat and self-improve until a quality bar is met.
The loop ends when a sub-agent outputs a clear exit signal, or when
`max_iterations` is reached.

```python
from google.adk.agents import LlmAgent, LoopAgent

refiner = LlmAgent(
    name="refiner",
    model="gemini-2.5-pro",  # refinement needs strong reasoning to know when it's good enough
    instruction="""
        Review the current draft. If it fully meets the requirements, respond
        with only: DONE
        Otherwise, improve it and output the improved version.
    """,
    tools=[],
)

root_agent = LoopAgent(
    name="refinement_loop",
    sub_agents=[refiner],
    max_iterations=5,   # safety limit — always set this
)
```

Rules:
- Always set `max_iterations` to prevent infinite loops (recommended: 3–10)
- The sub-agent's instruction must include a clear exit condition
- Use `gemini-2.5-pro` for the refiner — cheap models tend to declare "DONE" too early
- Good use cases: iterative writing improvement, retry-until-success workflows, self-checking agents

---

## Rules for Connecting MCP Servers

Use `MCPToolset` to give an ADK agent access to any MCP-compatible server.
The agent treats MCP server tools exactly like regular Python tools.

**stdio (local subprocess):**
```python
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioServerParameters

mcp_tools = MCPToolset(
    connection_params=StdioServerParameters(
        command="uvx",                          # command to launch the server
        args=["mcp-server-filesystem", "/tmp"], # server name + its arguments
    )
)

root_agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant. Use your tools as needed.",
    tools=[mcp_tools],  # pass MCPToolset directly — ADK discovers all tools inside it
)
```

**SSE (remote server already running):**
```python
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams

mcp_tools = MCPToolset(
    connection_params=SseServerParams(url="http://localhost:8080/sse")
)
```

Rules:
1. Import from `google.adk.tools.mcp_tool.mcp_toolset`
2. Pass the entire `MCPToolset` object in the agent's `tools` list — do NOT unpack it
3. You can mix `MCPToolset` with regular Python tool functions in the same `tools` list
4. The MCP server must be installed before the agent runs (remind user to `pip install` or `npm install` the server)
5. For stdio servers, the command must be available in the user's PATH

---

## Rules for Built-in Tools

ADK ships several ready-made tools you can drop into an agent without writing any Python functions.

### google_search — live web grounding

```python
from google.adk.tools import google_search

root_agent = LlmAgent(
    name="search_agent",
    model="gemini-2.5-flash",
    instruction="...",
    tools=[google_search],          # pass the singleton directly
)
```

- Pre-built singleton — import and add to `tools=[]`, no instantiation
- Works with `GOOGLE_API_KEY` only
- Can be mixed with regular Python tools in the same `tools` list

### url_context — read any URL

```python
from google.adk.tools import url_context

tools=[url_context]                 # fetches and reads URLs mentioned in chat
```

- Can be combined with `google_search`: `tools=[google_search, url_context]`

### BuiltInCodeExecutor — execute Python code

```python
from google.adk.code_executors import BuiltInCodeExecutor

root_agent = LlmAgent(
    name="code_runner",
    model="gemini-2.0-flash",
    instruction="...",
    code_executor=BuiltInCodeExecutor(),  # code_executor= NOT tools=
)
```

- **Critical:** goes on `code_executor=`, not `tools=`
- Cannot be combined with other tools in the same agent
- Requires Gemini 2.0+ model

### VertexAiSearchTool — search your own data store (Vertex AI only)

```python
from google.adk.tools import VertexAiSearchTool

tools=[VertexAiSearchTool(data_store_id="projects/.../dataStores/...")]
```

- Must be instantiated with `data_store_id` (not a singleton like the others)
- Requires Vertex AI env vars: `GOOGLE_GENAI_USE_VERTEXAI`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`
- To combine with other tools: add `bypass_multi_tools_limit=True`

### Other built-in singletons (all from `google.adk.tools`, Vertex AI + Gemini 2.0+)

| Import name | Use case |
|-------------|----------|
| `enterprise_web_search` | Enterprise web grounding (VPC-SC safe) |
| `google_maps_grounding` | Location and map grounding |

**When to suggest built-in tools:**
- User wants web search → `google_search`
- User wants to read/summarize a URL → `url_context`
- User wants the agent to write and run code → `BuiltInCodeExecutor`
- User has their own document corpus in Vertex AI Search → `VertexAiSearchTool`

---

## Rules for `__init__.py`

Always exactly one line, nothing else:

```python
from . import agent
```

---

## Rules for `.env`

Always exactly one line. Remind the user to replace the placeholder:

```
GOOGLE_API_KEY="your-api-key-here"
```

Also remind the user to install python-dotenv if not already present:
`pip install python-dotenv`

---

## Testing Instructions

After generating files, always tell the user:

> ```bash
> cd <project-name>
> adk web
> ```
>
> Open http://localhost:8000 — your agent appears in the **top-left dropdown**.
> Select it and start chatting.
>
> Don't have an API key yet? Get one free at https://aistudio.google.com

**Common reason agent doesn't appear in dropdown:**
- Running `adk web` from inside the inner `<project-name>/<project-name>/` package folder — run from the outer `<project-name>/` instead
- `load_dotenv()` missing from `agent.py` — ADK can't load the API key
- Syntax error in `agent.py` — check the terminal for import errors

---

## Handling Common Errors

If the user pastes an error, diagnose it and fix it directly:

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| Agent missing from `adk web` dropdown | `load_dotenv()` missing, wrong run directory, or syntax error | Add `load_dotenv()` to top of `agent.py`; run `adk web` from the outer `<project-name>/` folder (not from inside the inner package); check terminal for import errors |
| `ModuleNotFoundError: google.adk` | ADK not installed | Tell user: `pip install google-adk` |
| `ModuleNotFoundError: dotenv` | python-dotenv not installed | Tell user: `pip install python-dotenv` |
| `root_agent not found` | Wrong variable name | Rename agent variable to `root_agent` |
| `GOOGLE_API_KEY not set` | Missing .env or wrong path | Check .env exists in project root |
| `Tool not being called` | Weak docstring or instruction | Improve tool docstring and instruction |
| `AttributeError on None` | Missing null-check in tool | Add guard clause before accessing data |
| `MCPToolset` connection error | MCP server not installed or wrong command | Verify server is installed and command is in PATH |
| LoopAgent runs forever | No clear exit condition in sub-agent | Add explicit "respond DONE when finished" in instruction and set `max_iterations` |
| `AgentTool` agent never called | Coordinator instruction too vague | Name each AgentTool explicitly in the coordinator instruction with clear conditions |
| `ImportError: cannot import name 'code_execution'` | Wrong import — `code_execution` doesn't exist | Use `BuiltInCodeExecutor` from `google.adk.code_executors` with `code_executor=` |
| `BuiltInCodeExecutor` never runs | Added to `tools=` instead of `code_executor=` | Move it: `code_executor=BuiltInCodeExecutor()` |
| Built-in tool "not supported for this model" | Model too old | Switch to `gemini-2.0-flash` or `gemini-2.5-flash` (Gemini 2.0+ required) |
| `VertexAiSearchTool` auth error | Missing Vertex AI env vars | Add `GOOGLE_GENAI_USE_VERTEXAI=True`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` to `.env` |

---

## Reference Materials

See `references/adk-patterns.md` for copy-paste patterns covering: single agents,
API tools, stateful tools, coordinator pattern, SequentialAgent, ParallelAgent,
LoopAgent, built-in tools, and MCP server connections (stdio + SSE).

See `assets/agent-template.py` for a complete annotated starter template with
both single-agent and multi-agent scaffolds.
