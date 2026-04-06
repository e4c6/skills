# Low-Code Agent Model Selection

> **Agent type: Low-code agents using `uip agent` commands.**

Choose the right LLM model for your agent based on task complexity, speed requirements, and cost.

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
uip agent config set model "gpt-4.1-2025-04-14" --path my-agent
```

After changing, push and re-run evals to measure impact:

```bash
uip agent push my-agent --overwrite <solutionId>
uip agent eval run start --set "My Eval Set" --wait --path my-agent
```

---

## Model Recommendations by Use Case

| Use Case | Recommended Model | Why |
|----------|-------------------|-----|
| Simple classification | `gpt-4.1-mini-2025-04-14` | Fast, cheap, accurate for structured tasks |
| Tool-using agents (RAG, search) | `anthropic.claude-opus-4-6-v1` or `gpt-4.1-2025-04-14` | Better multi-step reasoning |
| High-volume processing | `gemini-2.5-flash` | Fastest, 1M context, lowest cost |
| Complex reasoning + accuracy | `anthropic.claude-opus-4-6-v1` | Best quality, slowest |
| Balanced quality/speed | `gpt-4.1-2025-04-14` | Good reasoning, 1M context, fast |

## Evaluator Model Selection

Use a **different model** for evaluators than the agent to avoid rate limit conflicts:

| Agent Model | Evaluator Model |
|-------------|-----------------|
| claude-opus | gpt-4.1 or gpt-5.4-mini |
| gpt-4.1 | gpt-5.4-mini |
| gemini-2.5-flash | gpt-4.1-mini |

Never use `same-as-agent` for evaluator models when the agent model has low rate limits — the concurrent LLM calls will hit 429 errors.

---

## Governance Policy

Models available depend on your tenant's governance policy. If `uip agent config list-models` doesn't show a model you expect, contact your tenant admin. Common issue: `gpt-4o` (without date suffix) may be blocked while `gpt-4o-2024-11-20` is allowed.
