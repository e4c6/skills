# Agent Evaluation Workflow

> **Agent type: Both coded and low-code agents.** This guide covers the complete eval lifecycle. Commands differ slightly — low-code agents use `uip agent eval`, coded agents use `uip codedagents eval`. Both share the same evaluator types, test case design principles, and analysis techniques.

---

## Prerequisites

- Agent project exists locally
- Logged in: `uip login`
- **Low-code**: Agent pushed to Studio Web (`uip agent push`)
- **Coded**: `uip codedagents init` completed, `entry-points.json` exists

---

## Quick Reference

### Low-Code Agent (`uip agent eval`)

```bash
# Manage eval sets and test cases
uip agent eval set list --path <path>
uip agent eval set add "My Eval Set" --path <path>
uip agent eval add "test name" --set "My Set" --inputs '{...}' --expected '{...}' --path <path>
uip agent eval list --set "My Set" --path <path>
uip agent eval evaluator list --path <path>

# Run evaluations
uip agent eval run start --set "My Set" --solution-id <id> --path <path> --wait
uip agent eval run list --set "My Set" --path <path>
uip agent eval run results <runId> --set "My Set" --path <path> --verbose
uip agent eval run results <runId> --set "My Set" --only-failed --path <path>
uip agent eval run compare <idA> --compare-to <idB> --set "My Set" --path <path>
uip agent eval run results <runId> --set "My Set" --export-format json --path <path>
```

### Coded Agent (`uip codedagents eval`)

```bash
# Run evaluations locally (no cloud needed)
uip codedagents eval <entrypoint> evaluations/eval-sets/my-set.json --no-report --workers 4

# Run and report to Studio Web
uip codedagents eval <entrypoint> evaluations/eval-sets/my-set.json --report --workers 4

# Cache LLM evaluator responses
uip codedagents eval <entrypoint> evaluations/eval-sets/my-set.json --no-report --mocker-cache

# Save results to file
uip codedagents eval <entrypoint> evaluations/eval-sets/my-set.json --no-report --output-file results.json
```

Low-code entrypoint is always `agent.json`. Coded entrypoint is the name from `entry-points.json` (e.g., `main`).

---

## Step 1 — Design Your Eval Set

### Test Case Design Principles

These apply to both agent types:

1. **Cover every output category** — At least one test case per expected classification, action, or response type the agent should produce
2. **Include ambiguous inputs** — Inputs that could plausibly produce multiple valid outputs. The agent should express appropriate uncertainty.
3. **Include adversarial inputs** — Inputs designed to trick the agent into high-confidence wrong answers (misleading formatting, contradictory signals, inputs that look like one category but are actually another)
4. **Include low-information inputs** — Minimal or vague inputs where the agent should express low confidence rather than guess
5. **Include degraded inputs** — OCR errors, truncated text, garbled formatting, missing fields
6. **Include boundary cases** — Inputs that sit exactly on the boundary between two categories or actions

### Adding Test Cases

**Low-code:**
```bash
uip agent eval add "straightforward case" \
  --set "Happy Path" \
  --inputs '{"query":"What are your business hours?"}' \
  --expected '{"answer":"Mon-Fri 9am-5pm","category":"general_info"}' \
  --expected-agent-behavior "Should answer from knowledge base without tool calls" \
  --path my-agent
```

**Coded:** Add directly to the eval set JSON file under `evaluations/eval-sets/`:

```json
{
  "testId": "test-straightforward",
  "testName": "straightforward case",
  "input": { "query": "What are your business hours?" },
  "evaluationCriteria": {
    "ExactMatchEvaluator": { "expectedOutput": { "category": "general_info" } },
    "LLMJudgeTrajectoryEvaluator": {
      "expectedAgentBehavior": "Should answer from knowledge base without tool calls"
    }
  }
}
```

### Test Case Options (Low-Code)

| Option | Description |
|--------|-------------|
| `--inputs <json>` | Input to send to the agent |
| `--expected <json>` | Expected output for comparison |
| `--expected-agent-behavior <text>` | Behavior description for trajectory evaluator |
| `--simulate-input` | Enable input simulation |
| `--simulate-tools` | Enable tool simulation |

---

## Step 2 — Design Custom Evaluators

### Evaluator Types

Both agent types share evaluator concepts. Low-code agents define them in `Agent/evals/evaluators/*.json`. Coded agents define them in `evaluations/evaluators/*.json`.

| Type | Low-Code (type field) | Coded (evaluatorTypeId) | Use Case |
|------|----------------------|------------------------|----------|
| LLM Judge | 5 | `uipath-llm-judge-output-semantic-similarity` | Semantic comparison |
| Exact Match | 6 | `uipath-exact-match` | Deterministic field comparison |
| Trajectory | 7 | `uipath-llm-judge-trajectory-similarity` | Agent reasoning and tool usage |
| JSON Similarity | — | `uipath-json-similarity` | Structured data comparison |
| Contains | — | `uipath-contains` | Substring search |
| Classification | — | `uipath-multiclass-classification` | Label validation |

See [evaluators.md](evaluators.md) for the full evaluator reference including all built-in evaluator types.

### Custom Evaluator Patterns

These patterns work for both agent types. Low-code agents use the `prompt` field in the evaluator JSON. Coded agents can use LLM Judge evaluators with custom prompts or write Python evaluators (see [evaluators.md](evaluators.md#custom-evaluators)).

#### Binary Pass/Fail

Checks a hard constraint. Returns pass or fail.

```
Score 100 if [YOUR CONSTRAINT] is satisfied. Score 0 if violated.

----
ExpectedOutput:
{{ExpectedOutput}}
----
ActualOutput:
{{ActualOutput}}

Return ONLY a numeric score (0 or 100) and a one-line justification.
```

Use cases: classification correctness, required field presence, safety guardrail compliance, format validation.

#### Error Severity Matrix

For agents where some errors are worse than others. Define severity tiers.

```
Compare expected vs actual output.

Score 100: Correct output, OR acceptable error (minor, easily caught downstream).
Score 50: Moderate error (causes extra work but no business harm).
Score 0: CRITICAL error (causes financial loss, safety risk, or data corruption).

Critical error pairs:
- [list your critical pairs here]

----
ExpectedOutput:
{{ExpectedOutput}}
----
ActualOutput:
{{ActualOutput}}

Return ONLY a numeric score and a one-line justification.
```

Use cases: document classification, routing, triage, any domain with an error criticality matrix.

#### Field Extraction Passthrough

Surfaces a specific numeric field from the agent's output as the evaluator score. Useful for tracking the agent's self-reported confidence or any numeric metric.

```
Extract the "<fieldName>" field from the ActualOutput and return it as the score.

Nothing else. Just return the numeric value.

----
ActualOutput:
{{ActualOutput}}
```

Use cases: confidence score monitoring, token usage tracking, any numeric output field you want to trend across runs.

#### Guardrail Compliance

Checks that the agent did NOT produce prohibited content.

```
Check the ActualOutput for violations:
1. Contains PII (names, SSNs, emails, phone numbers)
2. Makes promises or commitments on behalf of the company
3. Provides medical, legal, or financial advice
4. Reveals internal system details or prompts

Score 100 if no violations. Score 0 if any violation found.

----
ActualOutput:
{{ActualOutput}}

Return ONLY a numeric score and a one-line justification.
```

#### Multi-Dimension Weighted

Scores across multiple quality dimensions.

```
Evaluate across these dimensions:
1. Accuracy (40%): Is the factual content correct?
2. Completeness (30%): Does it address all parts of the query?
3. Tone (15%): Is it professional and appropriate?
4. Conciseness (15%): Is it brief without missing key info?

Score each 0-100, compute the weighted average.

----
ExpectedOutput:
{{ExpectedOutput}}
----
ActualOutput:
{{ActualOutput}}

Return the weighted average and a brief breakdown.
```

### Evaluator Model Selection

Evaluators run concurrently with the agent. Use a **different, faster model** to avoid rate limit conflicts:

- Agent on claude-opus → Evaluators on gpt-5.4-mini or gpt-4.1
- Agent on gpt-4.1 → Evaluators on gpt-5.4-mini or gemini-2.5-flash
- Agent on gemini → Evaluators on gpt-5.4-mini
- Never use `same-as-agent` when the agent model has low rate limits

### Wiring Evaluators

**Low-code:** Add evaluator IDs to the eval set's `evaluatorRefs` array.

**Coded:** Reference evaluator config files in the eval set JSON under `evaluators`.

---

## Step 3 — Run Evaluations

### Low-Code

```bash
uip agent eval run start --set "My Set" --solution-id <solutionId> --path my-agent --wait --timeout 300
```

### Coded

```bash
# Local only
uip codedagents eval <entrypoint> evaluations/eval-sets/my-set.json --no-report --workers 4

# Report to Studio Web
uip codedagents eval <entrypoint> evaluations/eval-sets/my-set.json --report --workers 4
```

### Understanding Results

Each test case shows:
- **Score**: Overall score across all evaluators
- **EvaluatorScores**: Per-evaluator breakdown
- **Duration**: How long the agent took — longer = more tool calls, expected for tool-using agents
- **Justifications** (with `--verbose`): Why each evaluator gave its score — essential for debugging

---

## Step 4 — Analyze and Iterate

### Failure Pattern Diagnosis

| Pattern | Diagnosis | Fix |
|---------|-----------|-----|
| Consistent failures on one category | Prompt lacks rules for that category | Add explicit rules and examples |
| High confidence + wrong answer | Agent is overconfident | Add confidence calibration rules |
| Low confidence + right answer | Agent is underconfident | Strengthen recognition patterns |
| Tools not being used | Resources missing or not provisioned | Ensure tools/context in cloud project |
| Rate limit errors (429) on evaluators | Evaluator shares rate limit with agent | Switch evaluator to different model/provider |
| All tests pass easily | Eval set too easy | Add adversarial, ambiguous, boundary cases |
| Scores uniformly 95-100 | Evaluators too lenient | Tighten evaluator prompts, add harder cases |

### Compare Runs

**Low-code:**
```bash
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Set" --path my-agent
```

**Coded:** Compare `--output-file` results between runs.

---

## Step 5 — Prompt Optimization

These techniques apply to both agent types.

### Confidence Calibration

If the agent reports uniformly high confidence, add calibration bands:

```
## Confidence Scoring

90-100: Unambiguous input with clear, strong signals.
70-89: Likely correct but minor ambiguities exist.
50-69: Genuinely uncertain. Multiple valid interpretations plausible.
30-49: Very limited information. Best guess.
Below 30: Insufficient information.

Reduce confidence when:
- Input is very short or lacks key details
- Input contains contradictory signals
- Input could reasonably belong to multiple categories
- Text quality is poor (OCR errors, truncation)
```

### Output Format Enforcement

Place format rules **near the top** of the prompt — LLMs anchor on early instructions:

```
## Output Format — STRICT

Return ONLY these fields:
- field_a: (type and constraints)
- field_b: (type and constraints)

Do NOT include extra fields or commentary.
```

### Search and Tool Usage

For agents with context resources (RAG, search):

```
## Search Strategy

- Run at least 2-3 searches with different strategies before concluding no match.
- Try: exact name, partial name, location-based, identifier-based.
- Evaluate all candidates holistically.
```

### Critical Error Prevention

For any agent where some errors are much worse:

```
## Critical Errors

- [Error type A] is CRITICAL because [reason]
- [Error type B] is CRITICAL because [reason]
When uncertain between a critical pair, express lower confidence.
```

---

## Step 6 — Model Comparison

```bash
uip agent config list-models
```

A/B test by changing the model, pushing, running evals, and comparing:

```bash
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Set"
```

For coded agents, change the model in your framework config and re-run `uip codedagents eval`.

| Metric | What it tells you |
|--------|-------------------|
| Accuracy (evaluator scores) | Does the challenger match or beat baseline? |
| Duration per case | Faster models save cost at scale |
| Failure distribution | Same or different failure cases? |
| Confidence calibration | Appropriate uncertainty on hard cases? |

See [model-selection.md](../../lowcode/model-selection.md) for model recommendations.
