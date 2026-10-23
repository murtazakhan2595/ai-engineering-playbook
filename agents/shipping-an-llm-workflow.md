# Shipping an LLM workflow to production

> **TL;DR**
> Lessons from building Shortlist, my job-search capstone: a hosted TypeScript app that writes application emails, answers forms and ranks jobs every night with LLMs.
> Break the work into short queued steps, guard every side effect in code, and measure quality with your own evals before real use.
> The mistake to avoid: one long request that does everything. It times out, can't be retried, and ties you to one host.

## Mental model

```
trigger (user action or nightly cron)
  → save input, queue step 1
  → worker runs ONE step: at most one model call, save result, queue next step
  → ... until the queue is empty
  → result waits in the app for a human to review
```

Code decides every step; the model never chooses what happens next. It's a **workflow**, not an agent (see [Agents and workflows](agents-and-workflows.md)).

## When to use / when not to

| Use short queued steps when | Avoid them when |
|---|---|
| Work has several model calls in a row (write → check → revise) | A single call answers within a few seconds |
| It runs unattended (nightly jobs) and failures must retry | The user waits on the screen and needs a streamed answer |
| The host limits request time (serverless) | You own a long-running server and one process is simpler |

## How it's done

- **Queue in the database.** Postgres (Supabase Queues) holds the steps. Queuing a step fires a database trigger that calls the worker route at once; a cron job also calls it every minute, in case a trigger is missed.
- **One step per call, with a time budget.** Each call has a 60-second budget, below the serverless limit. Reading one full-page screenshot took 22 s, so a form is read one screenshot per step.
- **Every step is safe to run twice.** Retries happen, so a step checks before acting: create the email draft only if no draft ID is saved yet. Each step has a test that runs it twice.
- **Loops as a graph, run one stage per step.** The write → check → revise loop is a LangGraph graph. Its state is saved in Postgres, and each queued step calls `invoke` with `interruptAfter: "*"` to advance one stage. Large inputs (the profile) go in the run's context, not in the saved state.
- **One function for every model call.** It reads the provider, model and reasoning effort for that step from the database, checks the answer against a Zod schema, and records tokens, cost and time, also when the call fails. Changing a step's model is a setting, not a deploy.
- **Code checks before model checks.** Before the model reviews a draft, plain code confirms that every number and technology in it appears in the profile and that it fits the word limit. This is cheap and certain, and catches most invented facts.

## Gotchas

- **Prompts don't guard side effects.** "Never send" in a prompt is not a control. Instead: the app asks only for Gmail's draft permission, the code refuses a token with any other scope, and a test fails the build if `messages.send` appears anywhere.
- **User-supplied text is untrusted.** Job posts can hold hidden instructions. The model has no tools and no access to Gmail, and the recipient comes from the extracted data, never from the model's draft. A fake model that obeys hidden text proves in tests that nothing happens.
- **One failure can stall a nightly batch.** If the last listing to be scored failed every retry, ranking would never start. Fix: a `giveUp` hook that runs after the final attempt and moves the batch on.
- **Reasoning effort matters as much as the model.** High effort can take over a minute: fine overnight, too slow for a one-minute capture.
- **Free tiers pause.** A free Supabase project pauses after a week idle; the every-minute cron keeps it awake.
- **Cost estimates are guesses until real runs.** The plan estimated $35 to $50 a month; real nightly runs came in far lower (below).

## Numbers worth remembering

From the app's own call log, October 2026:

| Measure | Value |
|---|---|
| Scoring one job listing (`gpt-6-luna`, medium) | $0.0005, ~4 s |
| Extracting a post (`gpt-6.1-sol`, low) | $0.005, ~5 s |
| Writing + checking one email | ~$0.04 |
| One nightly run (fetch, score, write notes) | $0.17 to $0.40 |
| First 9 days, including all building and testing (414 calls) | $4.69 |
| Extraction eval on pasted posts / screenshots | 7/7 and 8/8 |
| Draft correctness eval (recipient, role, CV) | 5/5 |
| Hosting | $0 (Vercel and Supabase free plans) |

## Sources

- [Agents and workflows](agents-and-workflows.md): the workflow-vs-agent choice this builds on
- [Prompting and structured outputs](../foundations/prompting-and-structured-outputs.md): schemas with rules in the field descriptions
- [Choosing a model](../decision-guides/choosing-a-model.md): pick on your own evals, not on rankings
- LangGraph.js docs on checkpointers and interrupts; Supabase Queues and Cron docs

## Open questions

- Does a cheaper open model (e.g. `deepseek-flash`) pass the same evals for writing? The harness can run it once the provider is added.
- How well does the nightly ranking match my own picks? Measured on 200 Apply/Skip marks; early numbers are too few to judge.
- What changes on a host with a 30-second limit (AWS Amplify)? The step budget drops and slow steps split; nothing else should.
