# Course log

What I've studied, in order, and where the code lives.

## Completed

### AI Engineer Core Track: LLM Engineering, RAG, QLoRA, Agents
**Ed Donner, Udemy** · [course page](https://www.udemy.com/course/llm-engineering-master-ai-and-large-language-models/) · studied Jan–Oct 2026
Code: [my fork of the course repo](https://github.com/murtazakhan2595/llm_engineering) · my exercises: [`community-contributions/ahmed_sohail`](https://github.com/murtazakhan2595/llm_engineering/tree/main/community-contributions/ahmed_sohail), [`week1/community-contributions/ahmed_sohail`](https://github.com/murtazakhan2595/llm_engineering/tree/main/week1/community-contributions/ahmed_sohail)

| Week | Topic | Built in the course | Notes |
|---|---|---|---|
| 1 | LLM APIs, prompting, tokens | Website summariser, brochure generator | [APIs & providers](foundations/llm-apis-and-providers.md), [prompting](foundations/prompting-and-structured-outputs.md) |
| 2 | Frontier models, Gradio, tools, multimodal | FlightAI assistant with tools, images and voice | [tools](foundations/tools-and-function-calling.md), [Gradio](foundations/gradio-for-ai-prototypes.md) |
| 3 | Hugging Face, open-source models | Pipelines, tokenizers, quantised models, meeting-minutes generator | [open-source models](foundations/open-source-and-local-models.md) |
| 4 | Model selection, code generation | Python → C++/Rust converter, frontier vs open-source comparison | [choosing a model](decision-guides/choosing-a-model.md) |
| 5 | RAG and RAG evaluation | Insurellm knowledge assistant, basic and advanced, with eval dashboard | [pipeline](rag/rag-pipeline.md), [advanced](rag/advanced-rag-techniques.md), [evaluation](rag/evaluating-rag.md) |
| 6 | Data curation, baselines, frontier fine-tuning | Product price predictor: classic ML → frontier → fine-tuned GPT-4.1 Nano | [data & baselines](fine-tuning/data-curation-and-baselines.md), [frontier fine-tuning](fine-tuning/fine-tuning-frontier-models.md) |
| 7 | QLoRA | Llama 3.2 3B fine-tuned to beat GPT-5.1 on price prediction | [QLoRA](fine-tuning/qlora.md) |
| 8 | Agents, serverless deployment | "The Price Is Right": 7-agent autonomous deal finder | [agents](agents/agents-and-workflows.md), [Modal](agents/serving-models-with-modal.md) |

## Up next

| Course | Focus | Status |
|---|---|---|
| AI Engineer Agentic Track (Ed Donner) | OpenAI Agents SDK, CrewAI, LangGraph, AutoGen, MCP | Next |
| MCP: Build Rich-Context AI Apps (DeepLearning.AI) | Model Context Protocol | Planned |
| A2A: The Agent2Agent Protocol (DeepLearning.AI) | Agent interoperability | Planned |
| LangChain & LangGraph (Eden Marco) | LangGraph in depth, LangSmith | Planned |
| AI Engineer Production Track (Ed Donner) | Deploying LLM systems on AWS and other clouds | Planned |
