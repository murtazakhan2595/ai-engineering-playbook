# Tools and function calling

> **TL;DR**
> The model never runs your code. It *asks* you to run a function, and you run it and send back the result.
> Write the loop to handle multiple and sequential tool calls from day one.
> Tool descriptions are prompts; guardrails belong in code.

## Mental model

1. You describe functions (name, purpose, JSON-schema parameters).
2. The model replies either with text or with `finish_reason == "tool_calls"` and a list of calls.
3. You execute each call, append the results as `role: "tool"` messages, and call the model again.
4. Repeat until it answers in text.

"Giving the LLM the power to run code" really means *giving it the opportunity to tell us it wants a tool run.*

## Tool schema

```python
price_function = {
    "name": "get_ticket_price",
    "description": "Get the price of a return ticket to the destination city.",
    "parameters": {
        "type": "object",
        "properties": {
            "destination_city": {"type": "string",
                                 "description": "The city that the customer wants to travel to"}
        },
        "required": ["destination_city"],
        "additionalProperties": False,
    },
}
tools = [{"type": "function", "function": price_function}]
```

## The loop

The course builds this in three steps: one call → several calls in one response → calls in sequence. The final version:

```python
response = client.chat.completions.create(model=MODEL, messages=messages, tools=tools)
while response.choices[0].finish_reason == "tool_calls":
    message = response.choices[0].message
    messages.append(message)
    messages.extend(handle_tool_calls(message))   # one {"role":"tool", ...} per call
    response = client.chat.completions.create(model=MODEL, messages=messages, tools=tools)
return response.choices[0].message.content
```

```python
def handle_tool_calls(message):
    responses = []
    for call in message.tool_calls:
        args = json.loads(call.function.arguments)
        result = TOOLS[call.function.name](**args)          # name -> function dict
        responses.append({"role": "tool", "content": str(result), "tool_call_id": call.id})
    return responses
```

## Design rules

- **Dispatch through an explicit dict**, not `globals()` (one course notebook does the latter). An explicit map is an allow-list.
- **Validate arguments before using them.** The course's SQLite tool uses parameterised queries (`WHERE city = ?`), which is the minimum.
- **Put hard limits in code, not just in the description.** The week 8 planner's `notify_user_of_deal` tool says "only call this one time" in its description, *and* the code ignores a second call. The description guides the model; the code enforces the rule.
- **Cap the loop.** A `while` loop around a model needs a maximum number of steps.
- **Tools can trigger side effects outside the chat.** In the multimodal assistant, a tool call also decided which city image to generate.

## Gotchas

- The first naive version forgets to pass `tools=` on the follow-up call, so the model can't chain tools.
- Tool results are strings to the model. Return compact, well-labelled data, not huge dumps.
- Tool output can contain text an attacker controls (web pages, user records). Treat it as data, not instructions.

## Sources

- Course: [week2/day4.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day4.ipynb) (FlightAI tools + SQLite), [week2/day5.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day5.ipynb) (multimodal), [week8/agents/autonomous_planning_agent.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/agents/autonomous_planning_agent.py) (planner tools + notify-once guard)

## Open questions

- Parallel tool calls: when does the model batch them, and does ordering ever matter?
- Best way to surface tool errors back to the model so it can recover?
