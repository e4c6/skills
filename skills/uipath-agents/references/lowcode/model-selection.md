# Low-Code Agent Model Selection

> **Agent type: Low-code agents using `uip agent` commands.**

Choose the right LLM model for your agent based on task complexity, speed requirements, and cost. Always verify available models with `uip agent config list-models` — availability depends on your tenant's governance policy.

---

## List Available Models

```bash
uip agent config list-models
uip agent config list-models --output json
```

Shows all models allowed by your tenant's governance policy, including provider, token limits, preview/deprecated status.

---

## Change Model

```bash
uip agent config set model "<model-name>" --path my-agent
```

After changing, push and re-run evals to measure impact:

```bash
uip agent push my-agent --overwrite <solutionId>
uip agent eval run start --set "My Eval Set" --wait --path my-agent
```

---

## Model Tiers

Models fall into three tiers. Pick the tier that matches your task, then choose within it.

### Tier 1 — Fast & Cheap (classification, routing, extraction)

Best for: structured tasks with clear rules, high-volume processing, simple Q&A.

| Model | Provider | Context | Strengths |
|-------|----------|---------|-----------|
| `gpt-5.4-mini-2026-03-17` | OpenAI | 128K out | Latest mini, fast, great structured output |
| `gpt-4.1-mini-2025-04-14` | OpenAI | 1M in / 32K out | Large context, good for document processing |
| `gemini-2.5-flash` | Vertex AI | 1M in / 65K out | Fastest, lowest cost, massive context |
| `anthropic.claude-haiku-4-5-20251001-v1:0` | AWS Bedrock | 64K out | Fast Anthropic option |

### Tier 2 — Balanced (tool-using agents, RAG, multi-step reasoning)

Best for: agents that call tools, search indexes, or need moderate reasoning.

| Model | Provider | Context | Strengths |
|-------|----------|---------|-----------|
| `gpt-4.1-2025-04-14` | OpenAI | 1M in / 32K out | Strong reasoning, 1M context, reliable tool use |
| `gpt-5.4` | OpenAI | 128K out | Latest full-size GPT, strong all-around |
| `anthropic.claude-sonnet-4-6` | AWS Bedrock | 64K out | Great reasoning, good tool use |
| `gemini-2.5-pro` | Vertex AI | 1M in / 65K out | Strong reasoning with massive context |

### Tier 3 — Maximum Quality (complex reasoning, ambiguous inputs, critical decisions)

Best for: agents handling ambiguous inputs where accuracy matters more than speed.

| Model | Provider | Context | Strengths |
|-------|----------|---------|-----------|
| `anthropic.claude-opus-4-6-v1` | AWS Bedrock | 128K out | Highest accuracy, best at nuance |
| `gpt-5.4` | OpenAI | 128K out | Latest GPT, competitive with Opus |

---

## Choosing by Task Type

| Task | Recommended Tier | Example Models |
|------|-----------------|----------------|
| Document classification | Tier 1 | gpt-5.4-mini, gpt-4.1-mini |
| Structured data extraction | Tier 1 | gpt-4.1-mini, gemini-2.5-flash |
| Customer support / Q&A | Tier 2 | gpt-4.1, claude-sonnet-4-6 |
| Tool-using agent (RAG, search, APIs) | Tier 2-3 | gpt-4.1, claude-opus |
| Multi-step reasoning with ambiguity | Tier 3 | claude-opus, gpt-5.4 |
| High-volume batch processing | Tier 1 | gemini-2.5-flash, gpt-5.4-mini |

---

## Evaluator Model Selection

Evaluators run concurrently with the agent. If both use the same model or provider, you'll hit 429 rate limits. Always use a **different, faster model** for evaluators:

| Agent Model | Evaluator Model |
|-------------|-----------------|
| claude-opus | gpt-5.4-mini or gpt-4.1 |
| gpt-5.4 | gpt-5.4-mini (different rate limit pool) |
| gpt-4.1 | gpt-5.4-mini |
| gemini-2.5-pro | gpt-5.4-mini |

Never use `same-as-agent` for evaluator models — this causes rate limit conflicts.

---

## Governance Policy

Models available depend on your tenant's governance policy. If `uip agent config list-models` doesn't show a model you expect, contact your tenant admin.

Common gotcha: bare model names like `gpt-4o` may be blocked by governance while date-suffixed versions like `gpt-4o-2024-11-20` are allowed. Always use the exact name from `list-models` output.
