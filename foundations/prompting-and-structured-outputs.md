# Prompting and structured outputs

> **TL;DR**
> System prompt = role, rules, tone. User prompt = the task and the data.
> One good example in the prompt (one-shot) often beats a paragraph of instructions.
> When code consumes the output, use structured outputs (a schema) instead of asking nicely for JSON.

## Mental model

- **System prompt:** who the model is, what it must and must not do, and how it should sound.
- **User prompt:** the request plus the material to work on.
- **Assistant messages:** earlier model replies, resent to give the illusion of memory.

Tone and behaviour are cheap to change in the system prompt. The same summariser became snarky, Spanish-speaking or a humorous brochure writer with one line.

## Patterns

**1. Format instructions that prevent common mistakes**
```text
Respond in markdown. Do not wrap the markdown in a code block - respond just with the markdown.
```

**2. One-shot example inside the system prompt**
```text
You should respond in JSON as in this example:
{"links": [{"type": "about page", "url": "https://full.url/goes/here/about"}, ...]}
```
For style, a sample reply works the same way: *"if the customer says 'I'm looking to buy a hat', you could reply something like, 'Wonderful - we have lots of hats...'"*.

**3. Dynamic context injection:** add instructions only when they're relevant. This is RAG in its smallest form.
```python
relevant_system_message = system_message
if "belt" in message.lower():
    relevant_system_message += " The store does not sell belts; point out other items on sale."
```

**4. Multi-step chains:** one call extracts (pick relevant links as JSON), a second call generates (write the brochure). Splitting the work keeps each prompt simple and testable. The course calls this the first agentic pattern.

**5. Multi-party conversations:** for three or more bots, send one system prompt plus one user prompt containing the whole transcript. Mapping several speakers onto `user`/`assistant` roles gets messy.

## JSON mode vs structured outputs

| | JSON mode | Structured outputs |
|---|---|---|
| How | `response_format={"type": "json_object"}` + an example in the prompt | `response_format=<Pydantic model>` via `.parse()` |
| Guarantee | Valid JSON, any shape | Matches your schema |
| Use for | Quick prototypes | Anything code depends on |

```python
class Deal(BaseModel):
    product_description: str = Field(description="...")
    price: float = Field(description="The actual price. If described as $100 off "
                                     "the usual $300, respond with $200")
    url: str

class DealSelection(BaseModel):
    deals: list[Deal] = Field(description="The 5 deals with the most detailed descriptions and clear prices")

result = client.chat.completions.parse(model="gpt-5-mini", messages=messages,
                                       response_format=DealSelection)
deals = result.choices[0].message.parsed
```

**`Field(description=...)` is part of the prompt.** The model reads those descriptions, so edge-case rules ("$100 off isn't the price") belong there.

## Temperature and reproducibility

- `temperature=0` means "pick the most likely token". It **does not** guarantee identical outputs. You also need a `seed`, and even then it's not always reproducible.
- For creativity, change the system prompt rather than raising temperature.

## Seeing what the model "thinks"

Requesting `logprobs=True, top_logprobs=3` returns each generated token with its probability and the top alternatives. The week 3 visualiser draws this as a tree. It's a useful reminder that generation is just repeated next-token prediction.

## Gotchas

- Models wrap output in code fences anyway. Strip them, or say explicitly not to.
- Typos in templates leak into the model's job. One course prompt said "main.{language}", which rendered as `main.Rust` instead of `main.rs`.
- Mutating a global system prompt works in notebooks but causes bugs in apps. Build the prompt per request.

## Sources

- Course: [week1/day1.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week1/day1.ipynb), [week1/day5.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week1/day5.ipynb) (one-shot JSON, chaining), [week2/day3.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day3.ipynb) (context injection), [week3/visualizer.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week3/visualizer.py) (logprobs), [week8/agents/deals.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/agents/deals.py) (Pydantic schemas)

## Open questions

- How reliable is structured output through OpenAI-compatible endpoints of other providers?
- Few-shot vs fine-tuning: at what point do examples in the prompt stop scaling?
