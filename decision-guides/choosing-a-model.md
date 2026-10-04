# Choosing a model

> **TL;DR**
> There's no "best model", only the best model **for this task, at this cost, under these constraints**.
> Rankings flip between tasks. In the course, the winner of one coding benchmark failed the next.
> Shortlist from leaderboards, decide with your own eval, and keep the model name in config so you can switch.

## Step 1: Hard constraints first

| Constraint | Pushes you toward |
|---|---|
| Data can't leave your network | Local / self-hosted open models (Ollama, your own GPU, Modal/SageMaker) |
| Tight latency | Small models, fast hosts (e.g. Groq), low reasoning effort |
| High volume, low margin | Small models, fine-tuned specialists, prompt caching |
| Hardest reasoning or coding | Frontier models with higher reasoning effort |
| Long documents | Large context windows plus caching. Or RAG instead of stuffing. |
| Multimodal (image, audio) | Providers with native support for that modality |

## Step 2: Shortlist 3–5 candidates

Mix tiers: one frontier, one cheap frontier, one open-weight. Leaderboards help shortlist; they don't decide.

## Step 3: Evaluate on *your* task

What the course showed (Ed Donner's runs):

**Python → C++, simple numeric loop (speedup vs Python)**
Gemini 2.5 Pro 1440× · Grok 4 1060× · gpt-oss-20b (local) 238× · GPT-5 233× · Claude Sonnet 4.5 184× · Qwen 2.5 Coder failed

**Python → Rust, harder task with large numbers**
GPT-5, Claude Sonnet 4.5, Gemini 2.5 Pro and most others **failed** · gpt-oss-120b won (~111,000×)

**Price prediction (avg error $, zero-shot)**
GPT-5.1 44.74 · Claude 4.5 47.10 · Gemini 3 Pro 50.54 · Grok 4.1 Fast 57.62 · GPT-4.1 Nano 62.51

**Lessons:**
- The two coding tasks ranked the models completely differently. One benchmark isn't enough. Include the hard and edge cases in your eval.
- Open-weight models compete. A local 20B model beat GPT-5 on one task, and a fine-tuned 3B model beat every zero-shot frontier model on another.
- Correctness comes before speed. A fast wrong answer (the large-number trap) is a fail.

## Step 4: Price it at real volume

`cost per call × calls per month`, including context tokens (RAG and chat history grow them) and retries. Then check the levers:
- smaller model
- lower reasoning effort
- prompt caching (static content first)
- batch APIs for offline jobs (the course rewrote 800k product descriptions for under $30 with Groq's batch API)
- a fine-tuned small model

## Step 5: Keep it swappable

- Model names in config, behind one client interface (OpenAI-compatible `base_url` or LiteLLM).
- Re-run the eval when a new model ships or an old one is retired. The course had to swap a retired Claude model mid-run.

## Related notes

[LLM APIs and providers](../foundations/llm-apis-and-providers.md) · [Open-source and local models](../foundations/open-source-and-local-models.md) · [Prompting vs RAG vs fine-tuning](prompting-vs-rag-vs-fine-tuning.md)
