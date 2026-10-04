# Evaluating RAG

> **TL;DR**
> "People are often focused on RAG architecture… even more important: evaluations!"
> Evaluate retrieval (did we fetch the right chunks?) and answers (was the reply right?) **separately**. They fail for different reasons.
> Know the blind spots of your metrics before trusting a green dashboard.

## The test set

The course uses 150 questions in JSONL, each with keywords, a reference answer and a category:

```json
{"question": "Who won the prestigious IIOTY award in 2023?",
 "keywords": ["Maxine", "Thompson", "IIOTY"],
 "reference_answer": "Maxine Thompson won the prestigious Insurellm Innovator of the Year (IIOTY) award in 2023.",
 "category": "direct_fact"}
```

| Category | Count | Tests |
|---|---|---|
| direct_fact | 70 | Single lookup |
| temporal | 20 | Dates, "when" |
| spanning | 20 | Facts across several documents |
| comparative | 10 | X vs Y |
| numerical | 10 | Numbers, totals |
| relationship | 10 | Links between entities |
| holistic | 10 | Whole-corpus questions |

The categories matter. Per-category scores show *where* the system fails: spanning and holistic questions usually expose chunking and k limits.

## Retrieval metrics

Relevance is approximated by keywords: a chunk is "relevant" if it contains the keyword.

| Metric | Question it answers | Computation |
|---|---|---|
| **MRR** | How high is the first relevant chunk? | Per keyword `1/rank` of the first hit, averaged |
| **nDCG@10** | Are relevant chunks near the top? | Binary relevance, `rel_i / log2(i+2)`, normalised |
| **Keyword coverage** | Did we retrieve the facts at all? | % of keywords found in any retrieved chunk |

## Answer metrics: LLM as judge

A judge model returns structured scores (1–5) with written feedback:

- **Accuracy:** "any wrong answer must score 1… an acceptable answer would score 3"
- **Completeness:** "only 5 if ALL information from the reference answer is included"
- **Relevance:** "5 = directly addresses the question and gives no additional information"
- Instruction to the judge: "Only give 5/5 scores for perfect answers."

## Thresholds used in the course dashboard

| Metric | Green | Amber |
|---|---|---|
| MRR / nDCG | ≥ 0.90 | ≥ 0.75 |
| Keyword coverage | ≥ 90% | ≥ 75% |
| Accuracy / completeness / relevance | ≥ 4.5 | ≥ 4.0 |

A single saved example shows why both halves matter. For the IIOTY question, retrieval scored **MRR 0.17**, but the answer still scored **accuracy 5/5**: the model got lucky with poorly-ranked context. Bad retrieval can hide behind good answers until it can't.

## Blind spots I found reading the evaluator

These are weaknesses in the course's evaluation design, worth fixing in my own:

1. **Substring keyword matching** gives false positives (a keyword like "5" matches almost anything) and false negatives (synonyms, different formatting).
2. **nDCG is normalised over the retrieved list only**, so it measures ordering, not recall against the whole corpus. Coverage partly fills that gap.
3. **The judge is the same small model as the generator** (`gpt-4.1-nano`). Use a stronger, different model as judge to reduce self-preference.
4. **Reference answers can be wrong.** A contributor fixed a holistic answer's total ($3,504,000 → $3,510,000). Review test sets like code.
5. The CLI entry point calls `load_tests("tests.jsonl")`, but `load_tests()` takes no arguments, so `uv run eval.py <n>` would crash. The dashboard path works.

## What I'd do in a real project

- Build the test set **before** tuning the pipeline, and keep it versioned.
- Track metrics per category and per change (chunk size, embedding model, k, reranker) in a results table.
- Add cost and latency per query as first-class metrics.
- Spot-read judge feedback weekly. Judges drift too.

## Sources

- Course: [week5/day4.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/day4.ipynb), [week5/evaluation/eval.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/evaluation/eval.py), [week5/evaluation/tests.jsonl](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/evaluation/tests.jsonl), [week5/evaluator.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/evaluator.py)

## Open questions

- Run the full eval on `implementation/` vs `pro_implementation/`: what's the real gain per category?
- Replace keyword relevance with LLM-judged chunk relevance and compare the rankings.
