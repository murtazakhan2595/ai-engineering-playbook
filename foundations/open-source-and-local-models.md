# Open-source and local models

> **TL;DR**
> Open-weight models are no longer "the weak option". In the course benchmarks a local 20B model beat GPT-5 on one task, and a fine-tuned 3B model beat every frontier model on another.
> 4-bit quantization (QLoRA-style) is what makes them fit on cheap GPUs.
> Choose them for cost, privacy, control or specialisation, and verify with your own evals.

## Ways to run them

| Option | What it's for | Course usage |
|---|---|---|
| **Ollama** (local) | Free, private, OpenAI-compatible endpoint at `localhost:11434/v1` | `llama3.2`, `gpt-oss:20b`, `deepseek-r1:1.5b` |
| **Hugging Face `transformers`** on a GPU | Full control: tokenizers, quantization, fine-tuning | Google Colab with a T4 (free or cheap tier) |
| **Hosted open models** (Groq, OpenRouter) | Open weights, someone else's GPUs, fast | `openai/gpt-oss-120b`, `qwen3-coder-30b` |
| **Serverless GPU** (Modal) | Your fine-tuned model behind an API | See [serving models with Modal](../agents/serving-models-with-modal.md) |

## The Hugging Face stack

- **Pipelines:** the high-level API. One line per task (sentiment, summarisation, translation, image generation).
- **Tokenizers:** turn text into token IDs. Each model family has its own vocabulary, special tokens and **chat template**. `tokenizer.apply_chat_template(messages)` produces the exact prompt format the model was trained on.
- **Models:** `AutoModelForCausalLM.from_pretrained(name, quantization_config=..., device_map="auto")`.

## 4-bit quantization

The config used to load Llama 3.2 3B in the course's deployed price model:

```python
quant_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_use_double_quant=True,        # quantize the quantization constants too
    bnb_4bit_compute_dtype=torch.float16,  # math happens in fp16
    bnb_4bit_quant_type="nf4",             # "normal float 4", suited to normally-distributed weights
)
```

The weights take ~1/4 of their fp16 memory, with a small quality hit. That's what makes a T4 enough.

## Evidence from the course (Ed Donner's runs)

**Python → C++ port, simple task (pi series), speedup vs Python:**

| Model | Speedup |
|---|---|
| Gemini 2.5 Pro | 1440× |
| Grok 4 | 1060× |
| **gpt-oss-20b (local)** | **238×** |
| GPT-5 | 233× |
| Claude Sonnet 4.5 | 184× |
| Qwen3 Coder 30B / DeepSeek Coder v2 | 168× |
| gpt-oss-120b | 14× |
| Qwen 2.5 Coder | failed |

**Harder task (O(n²) max-subarray with large numbers, ported to Rust):** GPT-5, Claude Sonnet 4.5, Gemini 2.5 Pro and most others **failed**. gpt-oss-120b won at ~111,000× faster than Python.

**Lesson:** rankings flip between tasks. Gemini went from 1st to fail, gpt-oss-120b from 8th to 1st. Leaderboards are a starting point; your own task-specific eval decides.

**Price prediction:** a QLoRA fine-tuned Llama 3.2 3B ($39.85 average error) beat GPT-5.1 ($44.74). The base model before fine-tuning was the worst entry of all ($110.72). See [QLoRA](../fine-tuning/qlora.md).

## Gotchas

- Don't pull models too big for your machine (`llama3.3`, `llama4` on a laptop). Start with `llama3.2:1b`.
- If Ollama "isn't responding", run `ollama serve` and check `http://localhost:11434`.
- The wrong chat template silently degrades a model. Always use the tokenizer's template.
- Compiler and runtime flags generated for one machine (`-mcpu=native`, absolute paths) don't port. Regenerate them per environment.

## Sources

- Course: [week1/day2.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week1/day2.ipynb), [week3/](https://github.com/murtazakhan2595/llm_engineering/tree/main/week3) (Colab notebooks), [week4/day4.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week4/day4.ipynb), [week4/day5.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week4/day5.ipynb), [week8/pricer_service2.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/pricer_service2.py)

## Open questions

- Re-run the week 4 benchmark myself: how stable are those rankings across runs?
- Where's the break-even between a hosted open model (Groq) and a self-served one (Modal) for steady traffic?
