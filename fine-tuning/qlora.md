# QLoRA: fine-tuning open-source models cheaply

> **TL;DR**
> QLoRA = load the base model in 4-bit, freeze it, train small low-rank adapter matrices on top.
> It turns "fine-tune a model" from a cluster job into a single-GPU job.
> In the course, a QLoRA fine-tuned Llama 3.2 **3B** beat GPT-5.1 at price prediction (39.85 vs 44.74 average error).

## Mental model

- **Full fine-tuning** updates every weight. That's expensive in memory and storage.
- **LoRA** freezes the original weights. For chosen layers (usually the attention projections), it learns a small update written as two thin matrices `B·A` of rank `r`. Only those matrices train: typically well under 1% of the parameters.
- **Q**LoRA adds 4-bit quantization of the frozen base, so the base takes ~¼ of the memory.
- The result is an **adapter**: a small set of weights you load on top of the base model at inference time.

## The knobs

| Knob | What it controls | Typical starting point |
|---|---|---|
| `r` (rank) | Adapter capacity | 8–64 |
| `lora_alpha` | Scale of the update (often 2×r) | 16–128 |
| `lora_dropout` | Regularisation | 0.05–0.1 |
| `target_modules` | Which layers get adapters | `q_proj, k_proj, v_proj, o_proj` (+ MLP for more capacity) |
| Learning rate, epochs, batch size, scheduler, warmup | Standard training settings | Tune on the lite dataset first |
| Quantization | `nf4`, double quant, fp16/bf16 compute | See below |

The course's exact training hyperparameters live in its Colab notebooks, not the repo. Re-running them is on my list (see open questions).

## The course setup

- **Base model:** `meta-llama/Llama-3.2-3B`
- **Data:** prompt/completion pairs, completions rounded to whole dollars for training
  ```text
  What does this cost to the nearest dollar?

  <product summary, max 110 tokens>

  Price is $
  ```
  Completion: `"123.00"`. Datasets are pushed to the Hugging Face Hub as `items_prompts_lite` (20k) and `items_prompts_full` (800k).
- **Training:** on a Google Colab GPU. The training notebooks are linked from the repo but live in Colab; the adapter is a PEFT adapter, loaded as shown below.
- **Serving** (week 8): base model in 4-bit + adapter, on a T4 via Modal:
  ```python
  quant_config = BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_use_double_quant=True,
                                    bnb_4bit_compute_dtype=torch.float16, bnb_4bit_quant_type="nf4")
  base = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.2-3B", quantization_config=quant_config)
  model = PeftModel.from_pretrained(base, "ed-donner/price-2025-11-28_18.47.07", revision=REVISION)
  # generate(max_new_tokens=5), then parse the number after "Price is $"
  ```
  Pinning the adapter `revision` makes the deployment reproducible.

## Results (Ed Donner's runs, avg error in $)

| Model | Error |
|---|---|
| Llama 3.2 3B base, 4-bit | 110.72 |
| + QLoRA on 20k examples | 65.40 |
| + QLoRA on 800k examples | **39.85** |
| GPT-5.1 zero-shot (for reference) | 44.74 |

The same tiny model went from worst on the leaderboard to beating every frontier model zero-shot, and the main lever was **data volume and quality**. Compare this with hosted fine-tuning of GPT-4.1 Nano, which made things worse ([notes](fine-tuning-frontier-models.md)).

## When QLoRA is worth it

- You have **lots** of labelled examples for a narrow task.
- You need a cheap, fast, private model at inference time.
- The task has a crisp metric you can evaluate before and after.
- You'd otherwise pay frontier prices for millions of calls.

## Gotchas

- Set `tokenizer.pad_token = tokenizer.eos_token` for Llama tokenizers that have no pad token.
- Parse outputs defensively. A 5-token generation can still produce junk; regex out the number.
- Evaluate the **base** model first. Without that 110.72, the 39.85 would mean much less.

## Sources

- Course: [week7/day2.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week7/day2.ipynb) (prompt data), [week7/](https://github.com/murtazakhan2595/llm_engineering/tree/main/week7) (Colab training notebooks), [week7/results.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week7/results.ipynb), [week8/pricer_service2.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/pricer_service2.py)
- Hugging Face LLM course, chapter 11 (LoRA/QLoRA/PEFT): https://huggingface.co/learn/llm-course/

## Open questions

- Re-run the Colab training and record the exact r / alpha / target modules / LR used, plus the trainable-parameter count.
- How does the 800k fine-tune do with RAG context added to its prompt?
