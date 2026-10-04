# The RAG pipeline

> **TL;DR**
> RAG = find the relevant pieces of your data, paste them into the prompt, let the model answer from them.
> The basic version is genuinely simple: load → chunk → embed → store → retrieve → answer.
> Most quality problems are retrieval problems, so measure retrieval separately (see [evaluating RAG](evaluating-rag.md)).

## Mental model

The model doesn't know your company's data. You can't paste all of it into every prompt, so for each question you **retrieve** the few chunks most similar in meaning and **augment** the prompt with them.

```
Ingest (offline):   documents → chunks → embeddings → vector store
Query (online):     question → embed → top-k similar chunks → prompt(context + question) → LLM
```

**Step zero needs no vectors.** The course's first version matched words in the question against employee surnames and product names, then added those files to the prompt. It's crude, but it shows the whole idea, and keyword matching still has a place in hybrid search.

## The pipeline (course defaults)

| Stage | Course choice | Notes |
|---|---|---|
| Load | `DirectoryLoader(glob="**/*.md")`, folder name → `doc_type` metadata | Metadata pays off later (filtering, plots, citations) |
| Chunk | `RecursiveCharacterTextSplitter(chunk_size=1000 or 500, chunk_overlap=200)` | Overlap keeps facts that straddle a boundary |
| Embed | `all-MiniLM-L6-v2` (free, local, 384-d) or `text-embedding-3-large` (API) | Pick one and evaluate. Don't assume. |
| Store | Chroma, persisted to disk | Rebuild deletes the old collection |
| Retrieve | `k = 10` | More context isn't free: cost and distraction |
| Answer | `gpt-4.1-nano`, `temperature=0` | Cheap model; retrieval quality matters more |

```python
SYSTEM_PROMPT = """You are a knowledgeable, friendly assistant representing the company Insurellm.
If you don't know the answer, say so.
Context:
{context}"""

docs = retriever.invoke(question, k=10)
context = "\n\n".join(d.page_content for d in docs)
```

The test corpus was a fictional insurance company "Insurellm": 76 markdown files across company, contracts, employees and products.

## Conversation and retrieval

A follow-up like "and what about his salary?" retrieves badly on its own. The course tried three approaches:
1. Ignore history: simple, breaks on follow-ups.
2. **Retrieve with all prior user messages concatenated** with the new question, and send the full history to the LLM (the `implementation/` version).
3. Rewrite the question into a standalone query with an LLM (the advanced version; see [advanced RAG](advanced-rag-techniques.md)).

## Looking at your embeddings

Project the vectors to 2D/3D with t-SNE (`TSNE(n_components=2, random_state=42)`) and colour them by document type. If the types don't form clusters, your embedding model or chunking isn't separating the concepts you care about.

## Framework or not

The course does it both ways: LangChain 1.0 (`langchain_chroma`, `ChatOpenAI`, `retriever.invoke`) and then **"No LangChain! Just native for maximum flexibility"** (OpenAI SDK + `chromadb` + Pydantic). With the native version every step is visible and swappable. Frameworks earn their place when you need many loaders or integrations.

## Gotchas

- `vectorstore.as_retriever()` with no `k` uses a library default you didn't choose. Set it explicitly.
- `temperature=0` makes answers more stable, not reproducible.
- Embedding everything in one API call works for 76 files, not for 76,000. Batch it.
- RAG answers are only as current as the last ingest. Plan re-indexing.

## Sources

- Course: [week5/day1.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/day1.ipynb) (no-vector RAG), [week5/day2.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/day2.ipynb) (chunk, embed, t-SNE), [week5/day3.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week5/day3.ipynb) (LangChain pipeline), [week5/implementation/](https://github.com/murtazakhan2595/llm_engineering/tree/main/week5/implementation)

## Open questions

- MiniLM vs `text-embedding-3-large` on the Insurellm test set: how big is the gap in MRR?
- At what corpus size does Chroma stop being the right default (vs pgvector, Pinecone)?
