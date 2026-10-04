# Agents and workflows

> **TL;DR**
> A **workflow** is code that decides the steps and calls LLMs along the way. An **agent** is an LLM that decides the steps by calling tools in a loop.
> The course capstone ships both planners and runs the *workflow* in production. That's often the right call.
> Agents are built from parts you already know: prompts, structured outputs, tools, memory.

## Mental model

| | Workflow | Agent |
|---|---|---|
| Who decides the next step | Your code | The model |
| Predictability | High | Lower |
| Debugging | Read the code | Read the trace |
| Good for | Known, repeatable processes | Open-ended tasks where the path varies |
| Course example | `PlanningAgent` | `AutonomousPlanningAgent` |

Chaining two LLM calls (week 1: pick links → write brochure) is already the simplest agentic pattern.

## The capstone: "The Price Is Right" deal finder

An autonomous system that scans deal feeds, estimates each product's true value, and sends a push notification for the best bargain.

```
DealAgentFramework  (memory.json, Chroma collection, logging, Gradio UI with 5-min timer)
└── PlanningAgent
    ├── ScannerAgent     RSS feeds → GPT-5-mini + structured outputs → 5 best-described deals
    ├── EnsembleAgent    preprocess text, then combine three pricers:
    │   ├── FrontierAgent        RAG: 5 similar products from Chroma → GPT-5.1   (weight 0.8)
    │   ├── SpecialistAgent      QLoRA fine-tuned Llama 3.2 3B on Modal          (weight 0.1)
    │   └── NeuralNetworkAgent   week 6 deep neural net                          (weight 0.1)
    └── MessagingAgent   Claude writes a short message → Pushover notification
```

**Workflow planner:** scan → price the 5 deals → sort by discount → alert if the best discount > $50.

**Autonomous planner:** `gpt-5.1` gets three tools and a goal:
```text
tools:  scan_the_internet_for_bargains, estimate_true_value, notify_user_of_deal
prompt: scan, estimate each deal, pick the single most compelling one, notify the user. Then reply OK.
```
It loops while `finish_reason == "tool_calls"`. The code refuses a second notification even if the model asks.

## Building blocks, and what each one taught

| Block | Lesson |
|---|---|
| **Structured outputs** (`DealSelection` Pydantic model) | Field descriptions carry the business rules ("$100 off $300 means the price is $200") |
| **RAG inside an agent** | The FrontierAgent is the strongest single pricer because of retrieved context |
| **Ensembles** | Different models make different mistakes. Fixed weights 0.8/0.1/0.1 gave **29.90** vs 30.19 for the best single model |
| **Memory** | A JSON file of past opportunities; the scanner skips URLs it has already seen |
| **Specialist models as tools** | A fine-tuned small model behind an API is just another tool |
| **Observability** | Each agent logs in its own colour; the UI streams logs live |
| **Guardrails in code** | "Notify once" is enforced in code, not only in the prompt |

## Design lessons

- **Start with a workflow.** Promote a step to agent control only when a fixed sequence can't handle the variation.
- **Give agents few, well-described tools** with tight schemas, and cap the loop.
- **Make side effects idempotent or guarded** (notifications, payments, writes).
- **Keep docs in sync with code.** The ensemble's docstring still mentions a linear-regression combiner that no longer exists; the weights are hard-coded.

## Gotchas

- The scanner only checks a few RSS feeds and truncates pages, so the quality of the input caps everything downstream.
- Memory as a growing JSON file is fine for a demo. For anything real, use a database with dedup keys.
- A course notebook dispatches tool calls through `globals()`. Use an explicit name-to-function map.

## Sources

- Course: [week8/agents/](https://github.com/murtazakhan2595/llm_engineering/tree/main/week8/agents), [week8/deal_agent_framework.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/deal_agent_framework.py), [week8/day3.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/day3.ipynb), [week8/day4.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/day4.ipynb), [week8/results.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/results.ipynb)

## Next

The Agentic Track goes deeper: OpenAI Agents SDK, CrewAI, LangGraph, AutoGen, MCP. Notes on those will land in this folder.
