# Serving models with Modal (serverless GPU)

> **TL;DR**
> Modal turns a decorated Python function or class into a GPU-backed API. No servers, pay per second.
> Load the model once per container (`@modal.enter()`), cache weights on a volume, and decide consciously how warm to keep it.
> This is how a fine-tuned open model becomes a tool your agents can call.

## Mental model

- You declare an **image** (OS + pip packages), **secrets**, a **GPU type** and a **timeout** in Python.
- `.local()` runs on your machine; `.remote()` runs in Modal's cloud.
- **Ephemeral app:** exists only while your script runs (`with app.run():`). Good for experiments.
- **Deployed app:** `modal deploy` gives a persistent service you look up by name from anywhere. Good for production.

## Patterns

**Function: simplest, but reloads the model every call**
```python
app = modal.App("pricer-service")
image = Image.debian_slim().pip_install("torch", "transformers", "bitsandbytes", "accelerate", "peft")
secrets = [modal.Secret.from_name("huggingface-secret")]

@app.function(image=image, secrets=secrets, gpu="T4", timeout=1800)
def price(description: str) -> float: ...
```

**Class: load once per container, cache weights on a volume**
```python
hf_cache = Volume.from_name("hf-hub-cache", create_if_missing=True)

@app.cls(image=image.env({"HF_HUB_CACHE": CACHE_DIR}), secrets=secrets, gpu="T4",
         timeout=1800, min_containers=MIN_CONTAINERS, volumes={CACHE_DIR: hf_cache})
class Pricer:
    @modal.enter()
    def setup(self): ...            # tokenizer + 4-bit base + PEFT adapter, once
    @modal.method()
    def price(self, description: str) -> float: ...
```

**Calling the deployed service from an agent**
```python
Pricer = modal.Cls.from_name("pricer-service", "Pricer")
estimate = Pricer().price.remote("iPhone 10")
```

## Cold starts and cost (course numbers)

| State | Latency |
|---|---|
| First build of the image | up to ~10 minutes |
| Cold container waking up | ~30 seconds |
| Warm container | ~2 seconds |

- Default: the container goes cold after ~2 minutes idle.
- `min_containers=1` keeps one always warm, and "will eat up credits".
- Middle ground: `update_autoscaler(scaledown_window=1200)` keeps it warm for 20 minutes after use.
- Pinning a `region` costs slightly more.

## Gotchas

- **Secret naming confuses everyone:** the secret's *name* is `huggingface-secret`, the *key* inside it is `HF_TOKEN`, the *value* is `hf_...`.
- **Two apps with the same name overwrite each other.** The course's `pricer_service.py` and `pricer_service2.py` both use `modal.App("pricer-service")`, so deploying the second replaces the first.
- On Windows, run `modal deploy` from a terminal rather than Jupyter (unicode errors), and check where `.modal.toml` was written.
- Pin model and adapter revisions so a redeploy doesn't silently change behaviour.

## Sources

- Course: [week8/day1.ipynb](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/day1.ipynb), [week8/hello.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/hello.py), [week8/pricer_service.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/pricer_service.py), [week8/pricer_service2.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/pricer_service2.py), [week8/agents/specialist_agent.py](https://github.com/murtazakhan2595/llm_engineering/blob/main/week8/agents/specialist_agent.py)

## Open questions

- Compare the same model on Modal vs AWS SageMaker serverless inference: cold start, cost per 1k calls, ops effort.
