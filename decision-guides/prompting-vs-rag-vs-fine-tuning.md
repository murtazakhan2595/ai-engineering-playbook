# Prompting vs RAG vs fine-tuning

> **TL;DR**
> Go in this order: **prompt → RAG → fine-tune**, and only move on when an eval says the current step isn't enough.
> RAG adds *knowledge*. Fine-tuning changes *behaviour*. Mixing up the two is the most common mistake.
> Course evidence: RAG lifted GPT-5.1 from $44.74 to $30.19 error; hosted fine-tuning made GPT-4.1 Nano worse; QLoRA on 800k examples made a 3B model beat GPT-5.1.

## Decision flow

```
Can a better prompt (instructions + 1–3 examples) hit the target on your eval?
├── Yes → stop. Ship it.
└── No → Is the gap missing or private/changing KNOWLEDGE?
        ├── Yes → RAG. Measure retrieval and answers separately.
        └── No, it's BEHAVIOUR (format, style, a narrow skill, cost/latency)
                ├── Little data (<1k examples) → better prompts, few-shot, structured outputs
                └── Lots of good labelled data (10k–1M) → fine-tune
                        ├── Need cheap/private/fast inference → QLoRA an open model
                        └── Want to stay on a hosted model → hosted fine-tuning (test vs base!)
```

## Comparison

| | Prompting | RAG | Fine-tuning |
|---|---|---|---|
| Adds new knowledge | No | **Yes** | Weakly, and it goes stale |
| Changes behaviour or format | Somewhat | No | **Yes** |
| Data needed | None | Your documents | Hundreds to millions of labelled examples |
| Time to first result | Minutes | Hours to days | Days |
| Keeps up with changing facts | n/a | Re-index | Re-train |
| Citations / traceability | No | **Yes** | No |
| Cost per call | Base | Base + context tokens | Can be **lower** (smaller model) |
| Main failure mode | Instructions ignored | Retrieval misses | Overfitting, lost general ability |

## Evidence from the course (Ed Donner's runs, price-prediction error in $)

| Approach | Model | Error |
|---|---|---|
| Prompt only | GPT-5.1 | 44.74 |
| **Prompt + RAG** (5 similar products) | GPT-5.1 | **30.19** |
| Prompt only | GPT-4.1 Nano | 62.51 |
| Hosted fine-tune (20k examples) | GPT-4.1 Nano | 75.91 (worse) |
| Prompt only | Llama 3.2 3B (4-bit) | 110.72 |
| **QLoRA** (800k examples) | Llama 3.2 3B | **39.85** |
| All combined (ensemble) | — | **29.90** |

**Reading it:**
1. When relevant examples exist, RAG gave the biggest single jump for the least effort.
2. Fine-tuning a capable hosted model on a knowledge-heavy task backfired.
3. Fine-tuning won when there was **a lot** of data, a **narrow** task and a **small** model to specialise.
4. The best system used all three, each where it fits.

## Questions to ask before choosing

- What does my eval say *today*? (No eval → build that first.)
- Is the failure "doesn't know" or "doesn't behave"?
- How often does the underlying knowledge change?
- How many good labelled examples do I actually have?
- What are the latency, cost and privacy constraints at production volume?

## Related notes

[RAG pipeline](../rag/rag-pipeline.md) · [Evaluating RAG](../rag/evaluating-rag.md) · [Fine-tuning frontier models](../fine-tuning/fine-tuning-frontier-models.md) · [QLoRA](../fine-tuning/qlora.md) · [Data curation and baselines](../fine-tuning/data-curation-and-baselines.md)
