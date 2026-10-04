# Data curation and baselines

> **TL;DR**
> "R&D with your dataset can often have a greater impact on performance than the fashionable hyper-parameter optimization."
> Before any LLM, build an evaluation harness and cheap baselines. They tell you what "good" means.
> In the course's price-prediction project, the curated data plus a fine-tuned 3B model beat every frontier model.

## The project used as the example

**Task:** predict a product's price from its text description. It's a regression problem dressed as language, so results are easy to score: **average absolute error in dollars**.

## Curation pipeline

| Step | What was done | Why |
|---|---|---|
| Source | `McAuley-Lab/Amazon-Reviews-2023`, 8 product categories | Real, messy, large |
| Filter | ≥ 600 chars of text; price $0.50–$999.49; parseable price | Remove junk and outliers |
| Clean | Drop "Part Number", "Best Sellers Rank" etc.; regex out 7+ char codes mixing caps and digits | Part numbers leak noise, not meaning |
| Dedupe | Shuffle (seed 42), drop duplicate titles, then duplicate full texts | Duplicates inflate scores |
| Rebalance | Weighted sample of 820k by `price²`; down-weight over-represented categories (Automotive ×0.05) | Raw data is dominated by cheap items |
| Rewrite | `gpt-oss-20b` on the Groq **Batch API** turns each item into a fixed Title/Category/Brand/Description/Details summary | Consistent input format; cost < $1 (lite) / < $30 (full, 800k items) |
| Truncate | 110 tokens of summary (Llama 3.2 tokenizer) | Fixed, small context for fine-tuning |
| Split | 800k train / 10k val / 10k test (lite: 20k / 1k / 1k) | Lite for iteration, full for final runs |

The **lite / full split** is a good habit: develop and debug on 20k rows, then run the full set once.

## Evaluation harness first

One `evaluate(predictor)` function:
- runs the first 200 test items
- reports mean absolute error, MSE and r²
- colours each prediction (green: error < $40 or < 20%; orange: < $80 or < 40%; red otherwise)

Every model in the course, from a constant guess to GPT-5.1, goes through the same function. That's what makes the leaderboard below meaningful.

## The full leaderboard (Ed Donner's course runs, avg error in $, lower is better)

| Model | Error | Kind |
|---|---|---|
| **Ensemble** (RAG frontier + fine-tuned + DNN) | **29.90** | System |
| GPT-5.1 with RAG (5 similar products as context) | 30.19 | Frontier + RAG |
| **Fine-tuned Llama 3.2 3B (QLoRA, 800k)** | **39.85** | Open-source fine-tune |
| GPT-5.1 (zero-shot) | 44.74 | Frontier |
| Deep neural net (289M params, residual MLP) | 46.49 | Classic DL |
| Claude 4.5 (Opus called; chart label says Sonnet) | 47.10 | Frontier |
| Gemini 3 Pro | 50.54 | Frontier |
| Grok 4.1 Fast | 57.62 | Frontier |
| GPT-4.1 Nano | 62.51 | Frontier (small) |
| Neural net (8 layers) | 63.97 | Classic DL |
| Fine-tuned Llama 3.2 3B (20k) | 65.40 | Open-source fine-tune |
| XGBoost | 68.23 | Classic ML |
| Random forest | 72.28 | Classic ML |
| GPT-4.1 Nano **fine-tuned** | 75.91 | Frontier fine-tune |
| Bag-of-words + linear regression | 76.81 | Classic ML |
| Human (Ed, 100 items) | 87.62 | Human |
| Linear regression (3 features) | 101.56 | Classic ML |
| Constant (training mean) | 106.18 | Baseline |
| Base Llama 3.2 3B, 4-bit, not fine-tuned | 110.72 | Open-source base |

## What the table teaches

1. **Baselines anchor everything.** Without "constant = $106", you can't tell whether $60 is good.
2. **Classic ML is strong and cheap.** XGBoost on bag-of-words beat a human and was close to GPT-4.1 Nano. A random forest on all 800k rows reached $56.40, but took ~15 hours.
3. **Data volume transforms small models.** Same 3B model: base 110.72 → 20k examples 65.40 → 800k examples 39.85.
4. **Inference-time context beats training for frontier models.** GPT-5.1 went from 44.74 to 30.19 just by adding 5 similar products to the prompt.
5. **Combining different models wins.** The ensemble edges out its best member.

## Gotchas

- The leaderboard comes from 200 test items (100 for the human, 50 for Gemini 3 Pro), so small differences are within noise.
- `trust_remote_code` errors when loading the dataset: pin `datasets==3.6.0` and restart.
- XGBoost on macOS may need `brew install libomp`.

## Sources

- Course: [week6/day1.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week6/day1.ipynb), [week6/day2.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week6/day2.ipynb), [week6/day3.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week6/day3.ipynb), [week6/day4.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week6/day4.ipynb), [week6/pricer/](https://github.com/murtazakhan2595/llm_engineering/tree/main/week6/pricer), [week7/results.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week7/results.ipynb), [week8/results.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/results.ipynb)

## Open questions

- How much of the fine-tune's gain comes from the LLM rewrite step vs the data volume?
- Bootstrap confidence intervals on the 200-item test: which leaderboard gaps are real?
