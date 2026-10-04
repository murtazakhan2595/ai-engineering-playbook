# Gradio for AI prototypes

> **TL;DR**
> Gradio is the fastest way to put a UI on a model: demos, MVPs and internal power-user tools.
> Pick the right one of three UI types, stream by yielding the *cumulative* text, and keep secrets out of `share=True` demos.
> For a customer-facing product, graduate to a real frontend.

## Mental model

Gradio builds a Svelte frontend, starts a Starlette server, and creates a backend route for every Python callback. You write Python functions; it handles the web app.

| UI type | Use it for |
|---|---|
| `gr.Interface` | One function, inputs → outputs |
| `gr.ChatInterface` | A standard chatbot |
| `gr.Blocks` | Anything custom: several components, your own event wiring |

## Patterns

**Streaming: yield the whole text so far, not the new chunk**
```python
def stream(prompt):
    result = ""
    for chunk in client.chat.completions.create(model=MODEL, messages=..., stream=True):
        result += chunk.choices[0].delta.content or ""
        yield result
```
Yield `""` first if the function does slow work before generating, so the old output clears immediately.

**Chatbot: strip Gradio's extra fields from history**
```python
def chat(message, history):
    history = [{"role": h["role"], "content": h["content"]} for h in history]
    messages = [{"role": "system", "content": system_message}] + history + \
               [{"role": "user", "content": message}]
    ...
gr.ChatInterface(fn=chat, type="messages").launch()
```

**Blocks: chain events with `.then()`**
```python
message.submit(put_message_in_chatbot, inputs=[message, chatbot], outputs=[message, chatbot]) \
       .then(chat, inputs=chatbot, outputs=[chatbot, audio_output, image_output])
```
The first step clears the textbox and shows the user's message instantly; the second calls the model.

**Long-running agents with live logs (week 8):** run the agent framework in a background thread, send its log records into a `queue.Queue` through a logging `QueueHandler`, and have a generator yield the latest log lines to an HTML component. A `gr.Timer(value=300)` re-runs the agent every 5 minutes.

## Gotchas

- `share=True` opens a public tunnel. Corporate networks and antivirus often block it, and anyone with the link can use your API key's budget.
- `auth=("user", "pass")` is fine for a demo. Read the credentials from env vars, not source code.
- Forcing dark mode overrides a user accessibility preference. Gradio advises against it.
- Component arguments change between major versions (e.g. `height` → `max_height` in v5). Pin the version in real projects.

## Sources

- Course: [week2/day2.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day2.ipynb), [week2/day3.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day3.ipynb), [week2/day5.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week2/day5.ipynb), [week8/price_is_right.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/price_is_right.py)
