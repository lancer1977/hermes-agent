---
title: Layered Model Routing
description: Route Hermes through LiteLLM and local models in a concrete multi-lane stack.
sidebar_label: Layered Routing
sidebar_position: 8.5
---

# Layered Model Routing

Use this when you want Hermes to keep a **strong primary model**, add a **LiteLLM proxy layer**, and keep **local models** in the mix for cheap or private work.

This is the concrete stack we recommend when one provider is not enough:

1. **Hermes** decides which lane to use for a task.
2. **LiteLLM** can act as the OpenAI-compatible front door or policy/router layer.
3. **Local models** handle cheap, private, read-heavy, or offline-friendly tasks.
4. **Fallback providers** catch outages and auth failures when the primary lane breaks.

## Recommended lane split

| Work type | Best lane | Why |
|---|---|---|
| Main reasoning / tool use | Strong remote model, optionally behind LiteLLM | Best quality and tool reliability |
| Compression / title generation / triage | Local model or cheap LiteLLM route | Low-risk, high-volume, cost-sensitive |
| Session scouting / summaries | Local model first | Fast and private |
| Browser screenshot / vision | Local if capable, otherwise strong remote | Use the cheapest model that can reliably see the page |
| Emergency fallback | Local endpoint or second remote lane | Keeps the session moving when the primary fails |

## Two valid topologies

### Topology A: LiteLLM as the front door

Hermes points its main model at LiteLLM, and LiteLLM decides whether to route to a remote model or a local server.

```yaml
model:
  provider: custom
  base_url: http://litellm:4000/v1
  default: claude-sonnet-4.5
  api_mode: chat_completions
```

Use this when you want one place to manage:

- provider selection
- local vs remote switching
- logging / policy / cost controls
- model aliases that are shared across tools

### Topology B: Hermes routes directly, LiteLLM is optional

Hermes sends the main model directly to a strong provider, while auxiliary slots point at local models or a custom LiteLLM endpoint.

```yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4.5

auxiliary:
  compression:
    provider: custom
    base_url: http://localhost:8001/v1
    model: llama-3.1-8b-instruct
  title_generation:
    provider: custom
    base_url: http://localhost:8001/v1
    model: qwen2.5-3b-instruct
```

Use this when you want the simplest main path and only offload cheap work.

## Practical routing plan

A good default for agentic work is:

- **main model**: strong remote reasoning model
- **compression**: local model
- **title generation**: local model
- **triage / scouting**: local model
- **fallback model**: local or secondary remote lane
- **specialized work**: direct provider only when the task needs native support

That gives you three useful behaviors:

1. **Cheap background work stays cheap.**
2. **Private or local-only tasks never leave the box.**
3. **High-stakes reasoning still goes to the best model.**

## Where LiteLLM fits

LiteLLM is most useful when you want a single OpenAI-compatible endpoint that can:

- normalize access to many providers
- expose local servers and remote APIs through one URL
- enforce routing policy outside Hermes
- make provider swaps invisible to the agent

If you use LiteLLM, treat it as a **routing layer**, not as the source of truth for task ownership. Hermes still decides which task slot is main, auxiliary, or fallback.

## Adding all your llama.cpp servers

If you have several llama.cpp servers, the cleanest approach is to name each one as a separate custom provider and then assign them to different lanes instead of forcing every request through a single box.

That gives you three useful options:

1. **Direct lane selection** — pick a specific llama.cpp server by name for a task slot.
2. **Fallback chain** — try one local server, then another, then a remote backup.
3. **LiteLLM aggregation** — point LiteLLM at all of the llama.cpp servers and let it pick among them.

Example `custom_providers` layout:

```yaml
custom_providers:
  - name: llama-home
    base_url: http://192.168.1.50:8080/v1
  - name: llama-gpu
    base_url: http://192.168.1.51:8080/v1
  - name: llama-workstation
    base_url: http://192.168.1.52:8080/v1
```

Then bind the lanes intentionally:

```yaml
model:
  provider: custom
  base_url: http://192.168.1.51:8080/v1
  default: llama-3.1-70b-instruct
  api_mode: chat_completions

fallback_providers:
  - provider: custom
    model: llama-3.1-70b-instruct
    base_url: http://192.168.1.50:8080/v1
  - provider: custom
    model: llama-3.1-70b-instruct
    base_url: http://192.168.1.52:8080/v1
```

Use this pattern when you want every llama.cpp box represented, but still want Hermes to keep a clear lane hierarchy.

If you want all the servers behind one alias, make LiteLLM the front door and point its router at the full local pool. Hermes then only needs one `base_url`, while LiteLLM handles which llama.cpp instance actually serves the request.

## Configuration checklist

1. Pick the **main model lane**.
2. Decide whether LiteLLM is the **front door** or just a **shared proxy**.
3. Map cheap auxiliary work to **local models** first.
4. Set a **fallback model** that can still answer if the primary lane fails.
5. Verify the model names are valid for the endpoint you exposed.
6. Run `hermes config check` and a short smoke session.

## Verification

Recommended smoke sequence:

```bash
hermes config check
hermes status
hermes model
```

Then confirm:

- the main model resolves to the intended lane
- auxiliary tasks show the expected local or proxy-backed model
- fallback behavior is documented and intentionally chosen

## Notes and pitfalls

- If your LiteLLM route is OpenAI-compatible, `api_mode: chat_completions` is usually the right default.
- If LiteLLM fronts a non-OpenAI-shaped backend, set `api_mode` explicitly.
- Do not assume a local model is good just because it is local. Use it where the task is read-heavy, short-horizon, or cheap to rerun.
- Do not route every task through LiteLLM if it hides a useful native provider feature you rely on.

## DreadBreadcrumb

- **Now:** Concrete Hermes + LiteLLM + local-model routing stack documented.
- **Proof:** This doc defines the layered routing roles, configuration examples, and verification checklist.
- **Next:** Wire the specific Hermes profile/config values to match the chosen topology.
