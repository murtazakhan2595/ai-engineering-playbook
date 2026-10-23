# AI Engineering Playbook

Practical notes on building with LLMs: how things work, when to use them, what goes wrong, and the numbers behind the decisions.

These are organised **by topic, not by course**. Each note follows the same structure (TL;DR → mental model → when to use → how → gotchas → sources → open questions) and links to the code it came from. I add to it as I go deeper into AI engineering.

## Decision guides

Start here. These turn the notes into choices.

- [**Prompting vs RAG vs fine-tuning**](decision-guides/prompting-vs-rag-vs-fine-tuning.md): a decision flow, plus evidence from a 19-entry leaderboard
- [**Choosing a model**](decision-guides/choosing-a-model.md): constraints first, then your own eval; why rankings flip between tasks

## Notes

### Foundations
| Note | In one line |
|---|---|
| [LLM APIs and providers](foundations/llm-apis-and-providers.md) | One OpenAI-compatible client for every provider; statelessness; prompt caching economics |
| [Prompting and structured outputs](foundations/prompting-and-structured-outputs.md) | System vs user prompts, one-shot examples, JSON mode vs Pydantic schemas |
| [Tools and function calling](foundations/tools-and-function-calling.md) | The tool loop, schemas, and why guardrails belong in code |
| [Gradio for AI prototypes](foundations/gradio-for-ai-prototypes.md) | Three UI types, streaming, live logs for long-running agents |
| [Open-source and local models](foundations/open-source-and-local-models.md) | Ollama, Hugging Face, 4-bit quantization; when open models win |

### RAG
| Note | In one line |
|---|---|
| [The RAG pipeline](rag/rag-pipeline.md) | Load → chunk → embed → store → retrieve → answer, with the defaults that matter |
| [Advanced RAG techniques](rag/advanced-rag-techniques.md) | LLM chunking, enrichment, query rewriting, multi-query, reranking |
| [Evaluating RAG](rag/evaluating-rag.md) | MRR, nDCG, coverage, LLM-as-judge, and the blind spots in each |

### Fine-tuning
| Note | In one line |
|---|---|
| [Data curation and baselines](fine-tuning/data-curation-and-baselines.md) | Curating 800k examples; why baselines come first; the full leaderboard |
| [Fine-tuning frontier models](fine-tuning/fine-tuning-frontier-models.md) | Hosted fine-tuning, and why it made the model worse here |
| [QLoRA](fine-tuning/qlora.md) | How a fine-tuned 3B open model beat GPT-5.1 |

### Agents
| Note | In one line |
|---|---|
| [Agents and workflows](agents/agents-and-workflows.md) | Workflow vs agent, and a 7-agent deal-finding system |
| [Serving models with Modal](agents/serving-models-with-modal.md) | Serverless GPU inference, cold starts, keeping models warm |
| [Shipping an LLM workflow to production](agents/shipping-an-llm-workflow.md) | Short queued steps, side effects guarded in code, real costs from a hosted app |

## Learning log

| Course | Status |
|---|---|
| [AI Engineer Core Track: LLM Engineering, RAG, QLoRA, Agents](https://www.udemy.com/course/llm-engineering-master-ai-and-large-language-models/) (Ed Donner) | ✅ Completed |
| AI Engineer Agentic Track (Ed Donner) | ⏭️ Next |

Details, the code behind each week, and what's planned: [courses.md](courses.md).

## Conventions

- **Benchmark numbers** come from the course runs by the instructor (Ed Donner) unless a note says otherwise. They're labelled where they appear.
- **"Open questions"** at the end of each note are experiments I want to run, not things I've verified.
- New notes start from [TEMPLATE.md](TEMPLATE.md).

## License

[CC BY 4.0](LICENSE): reuse freely with attribution.
