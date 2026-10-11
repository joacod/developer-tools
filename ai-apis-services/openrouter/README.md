# OpenRouter

[OpenRouter](https://openrouter.ai/) is a unified API and model catalog that makes it easy to switch between LLM providers and models without changing tools every time.

## Why I use it

For my setup, OpenRouter is useful because it gives me one place to:

- test different model families without rebuilding my workflow
- use strong daily models for agents like Hermes
- keep a cheaper or more available fallback ready
- use provider routing when I want to bias for cost, speed, or lower data retention

This page is a living shortlist of models I am using or actively tracking.

## Recommended models

This is a practical shortlist, not a fixed ranking.

Model quality, pricing, tool use behavior, and provider availability can change quickly, so I'll keep this page updated as I try new models.

### Coding models

| Model | Link | Notes |
| --- | --- | --- |
| `z-ai/glm-5.2` | [GLM 5.2](https://openrouter.ai/z-ai/glm-5.2) | Current Z.ai coding model on the catalog, with a 1,048,576-token context window. Track it beside 5.1 before I replace the model I already use for harder implementation work. |
| `z-ai/glm-5.1` | [GLM 5.1](https://openrouter.ai/z-ai/glm-5.1) | Previous Z.ai coding model in my regular rotation. Still listed while I compare it with 5.2. |
| `moonshotai/kimi-k2.6` | [Kimi K2.6](https://openrouter.ai/moonshotai/kimi-k2.6) | Coding-first model with a 262,144-token context window. Worth testing for long-horizon implementation work, coding-driven UI generation, and multi-step agent tasks. |
| `moonshotai/kimi-k2.5` | [Kimi K2.5](https://openrouter.ai/moonshotai/kimi-k2.5) | Earlier Kimi model. Keep it as the lighter comparison point against K2.6 for everyday coding. |

### Models for agents

These are the models I would look at first for daily agent use, where tool use, reliability, and price matter more than pure benchmark appeal.

| Model | Link | Notes |
| --- | --- | --- |
| `deepseek/deepseek-v4-pro` | [DeepSeek V4 Pro](https://openrouter.ai/deepseek/deepseek-v4-pro) | Good daily "brain" candidate for agents. Fast, practical, and a strong default when I want solid tool-oriented behavior at a reasonable price. |
| `deepseek/deepseek-v4-flash-0731` | [DeepSeek V4 Flash 0731](https://openrouter.ai/deepseek/deepseek-v4-flash-0731) | Preferred OpenRouter fallback for Hermes when the default model reaches its limit. |

`deepseek/deepseek-v4-flash-0731` is a dated snapshot. The page still resolves, and OpenRouter keeps dated snapshots addressable after a newer Flash release appears. Confirm the slug in the catalog before changing the Hermes fallback.
