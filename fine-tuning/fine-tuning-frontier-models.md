# Fine-tuning frontier models (via API)

> **TL;DR**
> Hosted fine-tuning is easy to run and cheap at small scale: upload JSONL, start a job, call your model by name.
> It's a tool for **format, style and consistency**, not for teaching a model new knowledge.
> In the course it made GPT-4.1 Nano *worse* at price prediction (62.51 → 75.91 average error).

## How it works

1. Write training and validation files as JSONL, one chat per line.
2. Upload them, start a job, wait.
3. Call the resulting model ID like any other model.

```json
{"messages": [
  {"role": "user", "content": "Estimate the price of this product. Respond with the price, no explanation\n\n<summary>"},
  {"role": "assistant", "content": "$123.00"}
]}
```

```python
job = openai.fine_tuning.jobs.create(
    training_file=train_file.id,
    validation_file=val_file.id,
    model="gpt-4.1-nano-2025-04-14",
    seed=42,
    hyperparameters={"n_epochs": 1, "batch_size": 1},
    suffix="pricer",
)
```

## What happened in the course (Ed Donner's runs)

| Setup | Avg error ($) |
|---|---|
| GPT-4.1 Nano, base, zero-shot | 62.51 |
| GPT-4.1 Mini fine-tuned, 200 examples | 96.58 |
| GPT-4.1 Mini fine-tuned, 2,000 examples | 79.29 |
| GPT-4.1 Nano fine-tuned, 2,000 examples | 82.26 |
| GPT-4.1 Nano fine-tuned, 20,000 examples (~$3.42) | 67.75 (one run) / 75.91 (leaderboard) |

The fine-tuned versions never beat the untuned base model. The next notebooks open with "just to redeem ourselves from the disappointing result".

## Why it disappointed (my reading)

- **Price estimation needs world knowledge** about products. Fine-tuning on a few thousand examples doesn't add much knowledge, and it can **narrow** what the model already knew.
- The examples teach the *format* ("$123.00") well. But the base model already followed the format from the prompt.
- More data helped (Nano 2k → 20k: 82 → 68) but didn't change the conclusion.
- The win came from elsewhere: **RAG** (GPT-5.1 + similar products: 30.19) and **full fine-tuning of a small open model on 800k examples** (39.85).

## When hosted fine-tuning *is* the right tool

| Good fit | Poor fit |
|---|---|
| Strict output format or tone the prompt can't hold | Teaching new facts (use RAG) |
| Shortening very long prompts that repeat the same instructions | Small data and a task the base model already does well |
| Distilling a big model's behaviour into a cheaper one | Fast-changing knowledge |
| Consistent classification labels | When you can't evaluate before and after |

OpenAI suggests starting with 50–100 good examples. Always compare against the **untuned** model with the same eval.

## Sources

- Course: [week6/day5.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week6/day5.ipynb), [week6/results.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week6/results.ipynb)

## Open questions

- Would adding a system prompt plus a few RAG examples *into the fine-tuning data* change the result?
- Distillation test: fine-tune Nano on GPT-5.1-with-RAG outputs instead of true prices.
