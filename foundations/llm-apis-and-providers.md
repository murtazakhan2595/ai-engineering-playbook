# LLM APIs and providers

> **TL;DR**
> Almost every provider speaks the OpenAI Chat Completions format, so one client plus a `base_url` covers most of the market.
> LLM calls are stateless: "memory" is just you resending the conversation, and you pay for it every turn.
> Provider choice is mostly a cost/latency/privacy decision; abstraction layers should stay thin.

## Mental model

An LLM API call is an HTTP POST with a list of messages. The model predicts what comes next in that conversation and returns it. That's the whole contract.

- The `openai` Python package is just a client library around that HTTP call. It contains no model code.
- Because OpenAI's format won, most providers (Anthropic, Gemini, DeepSeek, Groq, xAI, OpenRouter, Ollama) expose an **OpenAI-compatible endpoint**.
- **Every call is stateless.** If the model "remembers" your name, it's because the earlier turns were sent again in this request.

```python
# the raw call the SDK wraps
requests.post("https://api.openai.com/v1/chat/completions",
    headers={"Authorization": f"Bearer {api_key}"},
    json={"model": "gpt-5-nano", "messages": [{"role": "user", "content": "Tell me a fun fact"}]})
```

## One client, many providers

```python
from openai import OpenAI

anthropic  = OpenAI(api_key=ANTHROPIC_API_KEY, base_url="https://api.anthropic.com/v1/")
gemini     = OpenAI(api_key=GOOGLE_API_KEY,    base_url="https://generativelanguage.googleapis.com/v1beta/openai/")
deepseek   = OpenAI(api_key=DEEPSEEK_API_KEY,  base_url="https://api.deepseek.com")
groq       = OpenAI(api_key=GROQ_API_KEY,      base_url="https://api.groq.com/openai/v1")
grok       = OpenAI(api_key=GROK_API_KEY,      base_url="https://api.x.ai/v1")
openrouter = OpenAI(api_key=OPENROUTER_API_KEY, base_url="https://openrouter.ai/api/v1")
ollama     = OpenAI(api_key="ollama",           base_url="http://localhost:11434/v1")  # key is a dummy
```

A model-to-client dict then lets you switch providers with a dropdown (used for the Python→C++ comparison in week 4).

**Native SDKs** are still worth knowing when you need provider-specific features:
- Anthropic's native `messages.create` **requires `max_tokens`** and returns `response.content[0].text`.
- Gemini's `google.genai` client uses `models.generate_content(...)` and returns `response.text`.

## Abstraction layers

| Layer | What it is | When I'd use it |
|---|---|---|
| OpenAI client + `base_url` | Thin, no extra dependency | Default |
| LiteLLM | Thin wrapper, `provider/model` strings, reports cost per call | When you want cost tracking or many providers with one call signature |
| OpenRouter | Hosted router, one key for hundreds of models | Quick model comparisons, models you can't easily get elsewhere |
| LangChain | Heavy framework | When you need its ecosystem (loaders, retrievers), not just to call a model |

```python
from litellm import completion
r = completion(model="openai/gpt-4.1", messages=messages)
r._hidden_params["response_cost"]                      # dollars
r.usage.prompt_tokens_details.cached_tokens           # cache hits
```

## Cost levers

**Prompt caching:** the biggest cost lever for long, repeated context. The course demo used all of Hamlet (~190 KB) as context.

| Provider | How it works | Economics (per the course) |
|---|---|---|
| OpenAI | Automatic, **exact prefix match only** | Cached input ~4x cheaper |
| Anthropic | Explicit: you mark what to cache | Priming costs 25% more; cache reads ~10x cheaper |
| Gemini | Implicit and explicit | n/a |

The design rule that follows: **put static content (instructions, examples, documents) first and variable content (the user's question) last**, so the prefix stays cacheable.

**Reasoning effort:** `reasoning_effort="minimal" | "low" | "medium" | "high"` on reasoning models trades latency and cost against quality. This is inference-time scaling. Use minimal effort for easy tasks and save high effort for the hard ones.

**Other levers:** use the smallest model that passes your evals, truncate inputs (the course truncated scraped pages to 2–5k characters), and use local models where quality allows.

## Local models (Ollama)

| | Local (Ollama) | Frontier API |
|---|---|---|
| Cost | No API charges | Pay per token |
| Privacy | Data never leaves the machine | Leaves your network |
| Capability | Noticeably weaker | Strongest available |
| Hardware | `llama3.2` / `llama3.2:1b` on a laptop; `gpt-oss:20b` needs ~16 GB RAM | None |

## Gotchas

- **Statelessness costs money.** Resending history means every turn costs more than the last. You need summarisation or truncation strategies for long chats.
- **`.env` changes need `load_dotenv(override=True)`** in a running notebook, or you'll keep using the old key.
- **Scraping is fragile:** JS-rendered sites return empty content, and CloudFront-style protection returns 403. You need Playwright or Selenium for those.
- **Model names get retired.** The course had to swap Claude Haiku 3.5 for 4.5 mid-run. Keep model names in config, not code.
- Asking a model for today's date is a quick way to probe its training cutoff.

## Sources

- Course: [week1/day2.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week1/day2.ipynb) (raw HTTP, Ollama), [week1/day4.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week1/day4.ipynb) (tokens, statelessness), [week2/day1.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day1.ipynb) (providers, LiteLLM, caching, reasoning effort)

## Open questions

- How much does caching actually save on a real RAG workload, where retrieved context changes per query?
- Where does LiteLLM's abstraction leak (tool calling, structured outputs) across providers?
