# Advanced RAG techniques

> **TL;DR**
> Spend LLM calls at ingest time (better chunks) and at query time (better queries, better ranking).
> The course's "pro" pipeline: LLM chunking + enriched chunks + query rewriting + multi-query retrieval + LLM reranking + source-labelled context.
> Every technique adds latency and cost, so keep only what your evals show is worth it.

## The techniques

| # | Technique | Where it runs | What it fixes |
|---|---|---|---|
| 1 | **LLM-based semantic chunking** | Ingest | Fixed-size splits cut facts in half |
| 2 | **Chunk enrichment** (headline + summary + original) | Ingest | Raw text embeds poorly against short questions |
| 3 | **Query rewriting** | Query | Vague, conversational or follow-up questions |
| 4 | **Multi-query retrieval + merge** | Query | One phrasing misses relevant chunks |
| 5 | **LLM reranking** | Query | Vector similarity ≠ relevance to *this* question |
| 6 | **Source-labelled context** | Query | Model can't cite or tell documents apart |
| 7 | **Evaluation-aware system prompt** | Query | Rambling or partial answers |

### 1–2. Let an LLM chunk and enrich

Each whole document goes to a small model with a structured-output schema:

```python
class Chunk(BaseModel):
    headline: str       = Field(description="A brief heading most likely to be surfaced in a query")
    summary: str        = Field(description="A few sentences summarizing the content to answer common questions")
    original_text: str  = Field(description="The original text of this chunk, exactly as is")

# embedded text = headline + "\n\n" + summary + "\n\n" + original_text
```
The prompt suggests a chunk count (`len(text) // AVERAGE_CHUNK_SIZE + 1`), asks for ~25% overlap, and says *"don't leave anything out"*. The summary and headline bring a chunk's embedding closer to the way people ask questions.

### 3–4. Rewrite, then search twice

```python
rewritten = rewrite_query(question)      # "a VERY short specific question most likely to surface content"
chunks = merge_chunks(fetch(question, k=20), fetch(rewritten, k=20))   # dedupe on content
```
This is **not** hybrid search. Both searches are dense-vector queries. Adding BM25/keyword search is a separate, complementary step.

### 5. Rerank with an LLM, against the *original* question

```python
class RankOrder(BaseModel):
    order: list[int]    # chunk ids, most relevant first; "include all the chunk ids"

top = reranked[:10]     # retrieve 20+20 → rerank → keep FINAL_K = 10
```
A cross-encoder reranker is the cheaper, faster alternative to an LLM for the same job.

### 6–7. Label sources and set the bar

```python
context = "\n\n".join(f"Extract from {c.metadata['source']}:\n{c.page_content}" for c in chunks)
# system prompt: "Your answer will be evaluated for accuracy, relevance and completeness,
#                 so make sure it only answers the question and fully answers it."
```

## Production details from the pro version

- `tenacity` retries with exponential backoff (`wait_exponential(min=10, max=240)`) on every LLM call.
- Parallel ingestion with `multiprocessing.Pool(3)`; drop to 1 worker on rate-limit errors.
- LiteLLM, so the answer model can be swapped (the pro version answered with `gpt-oss-120b` on Groq).

## Gotchas

- **The pro module's query rewrite ignores conversation history.** `rewrite_query(question, history=[])` accepts history, but the pipeline calls it with the question only, so follow-ups lose context. (Found while reading `pro_implementation/answer.py`.)
- LLM chunking is slow and costs tokens per document. Fine for 76 files; budget it for large corpora.
- The course demo shows reranking moving the "Manchester University" chunk up, but there's no saved before/after metric. Measure your own.

## Sources

- Course: [week5/day5.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/day5.ipynb), [week5/pro_implementation/ingest.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/pro_implementation/ingest.py), [week5/pro_implementation/answer.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/pro_implementation/answer.py)

## Open questions

- Which single technique gives the biggest MRR gain on the Insurellm test set? (Ablation: add one at a time.)
- Does hybrid search (BM25 + dense) beat query rewriting for names and IDs?
