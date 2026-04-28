---
name: ollama-model-testing
description: Test and select Ollama models for Hermes plugins that call LLMs (reflect, validate, etc.). Use when picking a model for structured output tasks or when a plugin's default model fails.
---

# Ollama Model Testing for Hermes Plugins

## When to Use
- Installing a plugin that calls LLM via OpenAI-compatible endpoint
- Plugin's default model doesn't exist or produces broken output
- Need to find the best model for critique/validation/structured output

## The Core Problem
Ollama models accessed via `/v1` (OpenAI-compatible) endpoint may leak **CoT/reasoning traces** into the `choices[0].message.content` field. This breaks plugins that expect clean, structured output (JSON scores, flags, structured critique).

Example of leaked thinking:
```
Let me analyze this step by step...

 response {"score": 0, "flag": "REJECT", ...}
```

The `response` marker separates thinking from real output — but both end up in `content`.

## Workflow

### Step 1: Install & Identify Default Model
Read the plugin's `__init__.py` — find `MODEL = "..."` or `model` parameter in `call_llm()`.

### Step 2: Check Model Availability
```bash
ollama list | grep <model-name>
```
If the model doesn't exist, pick candidates.

### Step 3: Test Candidates for Clean Output
For each candidate model:
```python
from hermes_tools import terminal
# Test with a simple structured-output request
import requests
resp = requests.post(
    "http://localhost:11434/v1/chat/completions",
    json={
        "model": "candidate-model:cloud",
        "messages": [{"role": "user", "content": "Ответь ТОЛЬКО JSON: {\"score\": 5, \"reason\": \"...\"}"}],
        "temperature": 0.1,
        "max_tokens": 300
    },
    headers={"Authorization": "Bearer ollama"}
)
content = resp.json()["choices"][0]["message"]["content"]
# Check: does it start with JSON? Or with thinking?
print("FIRST 200 CHARS:", content[:200])
print("HAS THINKING:", "thinking" in content[:500] or "推理" in content[:500])
```

### Step 4: Selection Criteria
| Criteria | Good | Bad |
|---|---|---|
| Output starts with expected format | `{"score": ...}` | `Let me analyze...` |
| Language matches task | Russian for Russian critique | Mixed or wrong language |
| No thinking traces | Clean content |  <malthink blocks present |
| Fast enough | <5s for short prompt | >30s |

### Step 5: Apply Fallback Fixes in `evey_utils.py`
Add thinking-trace stripping as safety net:
```python
import re
THINKING_PATTERN = re.compile(
    r'(?:.*?<thinking>.*?</thinking>)|'
    r'(?:.*?thinking.*?response)|'
    r'(?:^[\s\S]*?</think>\s*)',
    re.DOTALL
)

def clean_llm_response(content: str) -> str:
    """Strip leaked thinking traces from Ollama responses."""
    cleaned = THINKING_PATTERN.sub('', content, count=1)
    return cleaned.strip()
```

Call `clean_llm_response()` on every `call_llm()` return value.

### Step 6: Integration Test
Run the plugin's handler with a known input and verify structured output:
```python
# Test reflect: ask it to critique a known-bad answer
# Test validate: ask it to validate a hallucination
```

## Known Working Models (Alexey's Server)
| Model | Clean output | Speed | Best for |
|---|---|---|---|
| `qwen3-coder-next:cloud` | ✅ Yes | Fast | reflect, validate |
| `qwen3.5:cloud` | ❌ Leaks thinking | Fast | Avoid for structured output |
| `deepseek-v4-pro:cloud` | ✅ Yes | Medium | General (default) |
| `glm-5:cloud` | ✅ Yes | Medium | Analytics tasks |
| `nemotron-3-nano:30b-cloud` | ✅ Yes | Fast | Lightweight tasks |

## Pitfalls
- **Never trust the model name in the plugin** —  always verify availability first
- **Ollama `/v1` endpoint != Ollama native API** — thinking traces leak ONLY through the OpenAI-compatible endpoint; native API handles them differently
- **`max_tokens` must be generous for structured output** — 300+ for JSON with explanation
- **`temperature: 0.1` is important** — reduces creative drift in structured output
- **Russian-language tasks need models that actually understand Russian** — test with a Russian prompt; some models respond in English despite Russian input
- **Do NOT use `qwen3.5:cloud` for any structured output** regardless of the task
