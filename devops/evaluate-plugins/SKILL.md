---
name: evaluate-plugins
description: Systematic evaluation of third-party plugins, MCP servers, or tool integrations for Hermes Agent. Use when auditing repos like hermes-plugins for compatibility with current infrastructure.
---

# Evaluate Plugins / MCP Servers / Tool Integrations

Systematic evaluation of third-party plugins, MCP servers, or tool integrations for Hermes Agent.

## When to Use
- User shares a GitHub repo of plugins/MCP servers to evaluate
- Considering whether to install a new tool or integration
- Auditing existing tools for relevance and compatibility

## Workflow

### Step 1: Gather Source Material
Clone or download the repo to a temporary location:
```bash
git clone <repo-url> /tmp/<name>/
# or use browser_navigate to read README first
```

### Step 2: Read Source Code (not just README)
For each plugin/candidate:
- Read `plugin.yaml` or `__init__.py` — look for `register()` function, tool schemas
- Identify `provides_tools` — what tools does it expose?
- Check `requires` or dependencies — what services/env vars does it need?

### Step 3: Dependency Check
For each candidate, scan for external dependencies:
- Environment variables: `os.environ.get("LANGFUSE_HOST")`, `DASHBOARD_URL`, etc.
- External services: Qdrant, Langfuse, MQTT broker, Docker containers, specific ports
- Local models: references to Ollama model names (qwen35-4b, etc.)
- Python imports: `from evey_utils import ...` — shared utility modules

Flag anything that requires services NOT present in the current environment.

### Step 4: Compare Against Built-in Capabilities
Hermes already has: `memory`, `session_search`, `todo`, `delegate_task`, `cronjob`, `skill_manage`, `terminal`, `browser_*`, `execute_code`, MCP tools.

Check each candidate against this list — skip anything that duplicates a built-in tool.

### Step 5: Categorize
- 🟢 **Recommended**: zero or minimal new dependencies, fills a gap in built-in tools
- 🟡 **Needs adaptation**: useful concept but requires removing hardcoded dependencies
- 🔴 **Not recommended**: duplicates built-in, requires unavailable infrastructure, or irrelevant domain

### Step 6: Report
Present findings as a table with columns: Plugin name | Purpose | Dependencies/Blockers. Include a comparison table showing built-in vs plugin capabilities. Always respond in the user's language (Russian for Alexey).

## Pitfalls
- README descriptions can be aspirational — always read `__init__.py` to see what's actually implemented
- Plugins may share a common utility module (`evey_utils.py`) — check if it exists in the repo
- Docker-dependent plugins (Qdrant, Langfuse) are usually non-starters unless those services are already running
- Some plugins wrap Hermes built-in tools (e.g., `cached_delegate` wraps `delegate_task`); evaluate whether the wrapper adds enough value to justify the dependency overhead
- **Default models in plugins are rarely available** — plugins hardcode model names (e.g., `qwen35-4b`) that don't exist on the server. After installation, always test and replace models using the `ollama-model-testing` skill
- **Ollama via `/v1` endpoint leaks thinking traces** — models like `qwen3.5:cloud` inject CoT blocks into `content`, breaking structured JSON output. Test candidates for clean output before committing to a model
