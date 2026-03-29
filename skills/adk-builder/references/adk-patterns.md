# ADK Patterns — Quick Reference

## Installation

```bash
pip install google-adk          # core
pip install google-adk[vertexai] # + Cloud deployment
```

## Environment

```bash
# .env (project root)
GOOGLE_API_KEY="your-key-here"
```

---

## Model Selection Quick Reference

| Model | Use when the agent… |
|-------|---------------------|
| `gemini-2.5-flash` | Calls tools, routes/delegates, retrieves data, summarizes short content |
| `gemini-2.5-pro` | Generates code, reasons over long context, synthesizes multi-agent outputs, handles ambiguous multi-constraint tasks |

In multi-agent systems, assign each agent the model that matches its specific job.
Do not use the same model for every agent.

---

## Pattern 1 — Minimal Single Agent

```python
from google.adk.agents import LlmAgent

def greet(name: str) -> dict:
    """Returns a greeting for the given name.

    Args:
        name: The person's name.
    Returns:
        Dict with status and greeting message.
    """
    return {"status": "success", "message": f"Hello, {name}!"}

root_agent = LlmAgent(
    name="greeter",
    model="gemini-2.5-flash",   # simple tool-calling — flash is the right pick
    instruction="""
        You are a friendly greeter.
        - greet: use when the user tells you their name or wants a greeting.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[greet],
)
```

---

## Pattern 2 — Tool That Calls an External API

```python
import requests

def get_weather(city: str) -> dict:
    """Fetches the current weather for a city using Open-Meteo (free, no key needed).

    Args:
        city: The city name, e.g. "Tokyo" or "New York".
    Returns:
        Dict with status, city, temperature_f, and condition code.
    """
    try:
        geo = requests.get(
            "https://geocoding-api.open-meteo.com/v1/search",
            params={"name": city, "count": 1},
            timeout=5,
        ).json()
        if not geo.get("results"):
            return {"status": "error", "message": f"City '{city}' not found."}
        r = geo["results"][0]
        wx = requests.get(
            "https://api.open-meteo.com/v1/forecast",
            params={
                "latitude": r["latitude"],
                "longitude": r["longitude"],
                "current": "temperature_2m,weathercode",
                "temperature_unit": "fahrenheit",
            },
            timeout=5,
        ).json()["current"]
        return {
            "status": "success",
            "city": city,
            "temperature_f": wx["temperature_2m"],
            "weather_code": wx["weathercode"],
        }
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

---

## Pattern 3 — Stateful Tool (remembers data in the session)

```python
from google.adk.tools import ToolContext

def add_item(item: str, context: ToolContext) -> dict:
    """Adds an item to the user's in-session list.

    Args:
        item: The item to add.
        context: Injected automatically by ADK — do not pass manually.
    Returns:
        Dict with status and the updated list.
    """
    items = context.state.get("items", [])
    items.append(item)
    context.state["items"] = items
    return {"status": "success", "items": items}

def get_items(context: ToolContext) -> dict:
    """Returns all items currently in the session list.

    Args:
        context: Injected automatically by ADK.
    Returns:
        Dict with status and the list of items.
    """
    return {"status": "success", "items": context.state.get("items", [])}

def clear_items(context: ToolContext) -> dict:
    """Clears all items from the session list.

    Args:
        context: Injected automatically by ADK.
    Returns:
        Dict with status and confirmation.
    """
    context.state["items"] = []
    return {"status": "success", "message": "List cleared."}
```

Note: ADK automatically injects `ToolContext` — just declare it as a parameter.
The state persists for the life of the session.

---

## Pattern 4 — Coordinator / Orchestrator (recommended for multi-domain)

Use when the agent needs to handle multiple distinct areas of expertise.
The coordinator is an `LlmAgent` that delegates to specialist agents via `AgentTool`.
Each specialist has its own focused tools and its own model.

```python
from google.adk.agents import LlmAgent
from google.adk.tools.agent_tool import AgentTool

# Specialist: needs deep reasoning for code — use pro
code_specialist = LlmAgent(
    name="code_specialist",
    model="gemini-2.5-pro",
    instruction="""
        You are a Python expert. Write, review, and fix Python code as requested.
        Return the final code block and a brief explanation.
    """,
    tools=[],   # add run_linter, run_tests, etc. as needed
)

# Specialist: simple retrieval — flash is fine
docs_specialist = LlmAgent(
    name="docs_specialist",
    model="gemini-2.5-flash",
    instruction="""
        You are a documentation expert. Answer questions about APIs, libraries,
        and frameworks accurately and concisely.
    """,
    tools=[],   # add google_search or fetch_url as needed
)

# Coordinator: routes to the right specialist
# Use flash if just routing; upgrade to pro if synthesizing complex outputs
root_agent = LlmAgent(
    name="coordinator",
    model="gemini-2.5-flash",
    instruction="""
        You are a smart coordinator. Route every request to the right specialist:
        - code_specialist: for writing, reviewing, debugging, or refactoring code
        - docs_specialist: for questions about libraries, APIs, or documentation

        Do not answer directly — always delegate to the right specialist.
        Combine their outputs into a clear response for the user.
    """,
    tools=[
        AgentTool(agent=code_specialist),
        AgentTool(agent=docs_specialist),
    ],
)
```

Key rules:
- Import: `from google.adk.tools.agent_tool import AgentTool`
- Only `root_agent` (the top-level coordinator) uses that variable name
- Specialist variable names can be anything descriptive
- Each specialist gets the model that matches its actual job
- The coordinator instruction must name each specialist and when to use it

---

## Pattern 5 — Sequential Multi-Agent Pipeline

Use when steps must run in a fixed order and each step feeds into the next.

```python
from google.adk.agents import LlmAgent, SequentialAgent

researcher = LlmAgent(
    name="researcher",
    model="gemini-2.5-flash",   # retrieval task — flash is sufficient
    instruction="Research the given topic thoroughly and output your findings.",
    tools=[],  # add google_search here for real research
)

analyst = LlmAgent(
    name="analyst",
    model="gemini-2.5-pro",   # deep analysis of research output — use pro
    instruction="Analyze the research findings and identify key insights and gaps.",
    tools=[],
)

summarizer = LlmAgent(
    name="summarizer",
    model="gemini-2.5-flash",   # summarization of short content — flash is fine
    instruction="Take the analysis and write a concise 3-bullet executive summary.",
    tools=[],
)

root_agent = SequentialAgent(
    name="research_pipeline",
    sub_agents=[researcher, analyst, summarizer],
)
```

---

## Pattern 6 — Parallel Multi-Agent (runs sub-agents simultaneously)

Use when tasks are independent and can run at the same time to save latency.
All sub-agents receive the same input and run simultaneously.

```python
from google.adk.agents import LlmAgent, ParallelAgent

# Security review needs deep reasoning — use pro
security = LlmAgent(
    name="security",
    model="gemini-2.5-pro",
    instruction="Review the code for security vulnerabilities. Be thorough.",
    tools=[],
)

# Style and performance reviews are straightforward — flash is fine
style = LlmAgent(
    name="style",
    model="gemini-2.5-flash",
    instruction="Review the code for style, readability, and naming conventions.",
    tools=[],
)
perf = LlmAgent(
    name="performance",
    model="gemini-2.5-flash",
    instruction="Review the code for performance bottlenecks and inefficiencies.",
    tools=[],
)

root_agent = ParallelAgent(
    name="code_reviewer",
    sub_agents=[security, style, perf],
)
```

---

## Pattern 7 — LoopAgent (repeat until condition met)

Use when a task needs to repeat and self-improve until quality is good enough.

```python
from google.adk.agents import LlmAgent, LoopAgent

# Use pro — cheap models declare "DONE" prematurely
refiner = LlmAgent(
    name="refiner",
    model="gemini-2.5-pro",
    instruction="""
        You are a writing editor. Review the current draft.
        If it is clear, concise, and complete, respond with only: DONE
        Otherwise, rewrite it to be better and output the improved version.
    """,
    tools=[],
)

root_agent = LoopAgent(
    name="writing_loop",
    sub_agents=[refiner],
    max_iterations=4,  # always set a safety limit
)
```

You can combine a generator + checker in a loop for tighter control:

```python
from google.adk.agents import LlmAgent, SequentialAgent, LoopAgent

generator = LlmAgent(
    name="generator",
    model="gemini-2.5-pro",    # generation needs strong reasoning
    instruction="Generate or improve the current draft based on any feedback.",
    tools=[],
)
checker = LlmAgent(
    name="checker",
    model="gemini-2.5-flash",  # checking against a rubric is simpler
    instruction="""
        Check the draft against the requirements.
        If it passes all checks, respond with only: DONE
        Otherwise, list exactly what needs to be fixed.
    """,
    tools=[],
)

root_agent = LoopAgent(
    name="generate_and_check",
    sub_agents=[
        SequentialAgent(name="inner", sub_agents=[generator, checker])
    ],
    max_iterations=5,
)
```

---

## Pattern 8 — google_search (live web grounding)

```python
from google.adk.agents import LlmAgent
from google.adk.tools import google_search

root_agent = LlmAgent(
    name="search_agent",
    model="gemini-2.5-flash",
    instruction="""
        You are a research assistant with live web access.
        - google_search: use when you need current or factual information from the web.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[google_search],
)
```

Notes:
- `google_search` is a pre-built singleton — import and pass directly, no instantiation
- Requires Gemini 2.0+ model (`gemini-2.5-flash` works)
- Only `GOOGLE_API_KEY` needed — no extra setup
- Can be combined with regular Python tools: `tools=[google_search, my_tool]`

---

## Pattern 9 — BuiltInCodeExecutor (execute Python code)

```python
from google.adk.agents import LlmAgent
from google.adk.code_executors import BuiltInCodeExecutor

root_agent = LlmAgent(
    name="code_runner",
    model="gemini-2.0-flash",   # BuiltInCodeExecutor requires Gemini 2.0+
    instruction="""
        You are a Python assistant. Solve math and data problems by writing
        and executing Python code. Always show the code you ran and its output.
    """,
    code_executor=BuiltInCodeExecutor(),   # NOTE: code_executor= NOT tools=
)
```

Notes:
- Import: `from google.adk.code_executors import BuiltInCodeExecutor`
- Use `code_executor=` on `LlmAgent` — **NOT** `tools=`
- Cannot be combined with other tools in the same agent
- Requires Gemini 2.0+ (`gemini-2.0-flash` or `gemini-2.5-flash`)

---

## Pattern 10 — url_context (read any URL)

```python
from google.adk.agents import LlmAgent
from google.adk.tools import url_context

root_agent = LlmAgent(
    name="url_reader",
    model="gemini-2.5-flash",
    instruction="""
        You are a web content assistant.
        - url_context: use when the user shares a URL and wants you to read or summarize it.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[url_context],
)
```

Notes:
- `url_context` automatically fetches and reads any URL mentioned in the conversation
- Requires Gemini 2.0+ model
- Can be combined with `google_search`: `tools=[google_search, url_context]`

---

## Pattern 11 — VertexAiSearchTool (search your own data store)

Use when you have a Vertex AI Search data store with your own documents/knowledge base.

```python
from google.adk.agents import LlmAgent
from google.adk.tools import VertexAiSearchTool

DATA_STORE_ID = (
    "projects/YOUR_PROJECT/locations/global/"
    "collections/default_collection/dataStores/YOUR_DATASTORE"
)

root_agent = LlmAgent(
    name="doc_search_agent",
    model="gemini-2.5-flash",
    instruction="""
        You are a company knowledge assistant.
        Search the knowledge base to answer questions accurately.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[VertexAiSearchTool(data_store_id=DATA_STORE_ID)],
)
```

Requires Vertex AI `.env` (instead of `GOOGLE_API_KEY`):
```
GOOGLE_GENAI_USE_VERTEXAI=True
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
```

Notes:
- Unlike `google_search`, `VertexAiSearchTool` must be instantiated with a `data_store_id`
- Requires Vertex AI — not compatible with `GOOGLE_API_KEY` alone
- To combine with other tools: `VertexAiSearchTool(data_store_id=DATA_STORE_ID, bypass_multi_tools_limit=True)`

---

## Pattern 12 — Other Built-in Singletons (quick reference)

All imported from `google.adk.tools` and passed in `tools=[]` like `google_search`:

```python
from google.adk.tools import enterprise_web_search  # Vertex AI enterprise web grounding
from google.adk.tools import google_maps_grounding  # Google Maps grounding (requires Vertex AI)
from google.adk.tools import url_context            # fetch and read any URL

# Example: combine google_search + url_context
from google.adk.agents import LlmAgent
from google.adk.tools import google_search, url_context

root_agent = LlmAgent(
    name="research_agent",
    model="gemini-2.5-flash",
    instruction="""
        You are a research assistant.
        - google_search: search the web for current information.
        - url_context: read and summarize a specific URL the user provides.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[google_search, url_context],
)
```

| Singleton | Use case | Requires |
|-----------|----------|----------|
| `google_search` | Live web search | `GOOGLE_API_KEY`, Gemini 2.0+ |
| `url_context` | Read content at a URL | `GOOGLE_API_KEY`, Gemini 2.0+ |
| `enterprise_web_search` | Enterprise web grounding (VPC-SC safe) | Vertex AI, Gemini 2.0+ |
| `google_maps_grounding` | Location/map grounding | Vertex AI, Gemini 2.0+ |

---

## Pattern 13 — MCP Server as Tools (stdio)

```python
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioServerParameters

mcp_tools = MCPToolset(
    connection_params=StdioServerParameters(
        command="uvx",
        args=["mcp-server-filesystem", "/tmp"],
    )
)

root_agent = LlmAgent(
    name="filesystem_agent",
    model="gemini-2.5-flash",   # file ops are straightforward — flash is fine
    instruction="""
        You are a file assistant. Use your tools to read, write, and
        list files as the user requests.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[mcp_tools],   # pass the whole MCPToolset — ADK handles the rest
)
```

---

## Pattern 14 — MCP Server as Tools (SSE / remote server)

```python
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, SseServerParams

mcp_tools = MCPToolset(
    connection_params=SseServerParams(
        url="http://localhost:8080/sse",
        # headers={"Authorization": "Bearer token"}  # optional auth
    )
)

root_agent = LlmAgent(
    name="remote_mcp_agent",
    model="gemini-2.5-flash",
    instruction="""
        You are a helpful assistant with access to remote tools via MCP.
        Always use the appropriate tool rather than guessing.
    """,
    tools=[mcp_tools],
)
```

Common MCP servers to use with ADK:
| Server | Install | What it provides |
|--------|---------|-----------------|
| Filesystem | `uvx mcp-server-filesystem <path>` | Read/write local files |
| GitHub | `uvx mcp-server-github` | Repos, issues, PRs |
| PostgreSQL | `uvx mcp-server-postgres <conn>` | SQL queries |
| Slack | `uvx mcp-server-slack` | Read/send Slack messages |
| Fetch | `uvx mcp-server-fetch` | HTTP requests |

---

## Running Locally

```bash
# Interactive web UI (recommended for development)
adk web

# Terminal chat mode
adk run <agent-folder>

# Check ADK version
adk --version
```

`adk web` must be run from the **parent directory** of your agent folder,
not from inside it.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Agent variable not named `root_agent` | Always use exactly `root_agent` |
| `__init__.py` missing | Create with `from . import agent` |
| Tool returns a string instead of dict | Wrap: `return {"status": "success", "result": value}` |
| `adk web` run from inside agent folder | Run from the parent directory |
| API key not found | Check `.env` is in the agent folder root |
| Tool never called by agent | Improve the tool's docstring description |
| Every agent uses the same model | Assign each agent the model that fits its job |
| `SequentialAgent` sub-agent named `root_agent` | Only the top-level agent should be `root_agent` |
| `LoopAgent` never stops | Add `max_iterations` and a clear DONE signal in sub-agent instruction |
| Unpacking `MCPToolset` in tools list | Pass the whole object: `tools=[mcp_tools]`, not `tools=[*mcp_tools]` |
| `AgentTool` specialist never called | Name it explicitly in coordinator instruction with clear trigger conditions |
| `ImportError: cannot import name 'code_execution'` | There is no `code_execution` in `google.adk.tools`; use `BuiltInCodeExecutor` from `google.adk.code_executors` with `code_executor=` |
| `BuiltInCodeExecutor` not triggering | It goes on `code_executor=`, not inside `tools=[]` |
| Built-in tool "not supported for this model" | Built-in tools require Gemini 2.0+; switch to `gemini-2.0-flash` or `gemini-2.5-flash` |
| `VertexAiSearchTool` auth error | Requires Vertex AI env vars: `GOOGLE_GENAI_USE_VERTEXAI`, `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION` |
| Combining `VertexAiSearchTool` with other tools fails | Pass `bypass_multi_tools_limit=True` to `VertexAiSearchTool(...)` |
