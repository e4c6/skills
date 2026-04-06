# Low-Code Agent Evaluation Workflow

> **Agent type: Low-code agents using `uip agent` commands.** For coded agents using `uip codedagents eval`, see [running-evaluations.md](../lifecycle/evaluations/running-evaluations.md).

This guide covers the complete evaluation workflow for low-code agents — creating eval sets, adding test cases, designing custom evaluators, running evaluations, analyzing results, and iterating on prompts.

---

## Prerequisites

- Agent project exists locally (from `uip agent init` or `uip agent pull`)
- Agent is pushed to Studio Web (`uip agent push`)
- Logged in: `uip login`

---

## Quick Reference

```bash
# List eval sets
uip agent eval set list --path <agent-path>

# Create an eval set
uip agent eval set add "My Eval Set" --path <agent-path>

# Add test cases
uip agent eval add "test name" \
  --set "My Eval Set" \
  --inputs '{"input":"user message"}' \
  --expected '{"field":"expected value"}' \
  --expected-agent-behavior "Description of what the agent should do" \
  --path <agent-path>

# List test cases in a set
uip agent eval list --set "My Eval Set" --path <agent-path>

# List evaluators
uip agent eval evaluator list --path <agent-path>

# Run evaluation (async)
uip agent eval run start --set "My Eval Set" \
  --solution-id <solutionId> --path <agent-path>

# Run and wait for results
uip agent eval run start --set "My Eval Set" \
  --solution-id <solutionId> --path <agent-path> --wait --timeout 300

# List past runs
uip agent eval run list --set "My Eval Set" --path <agent-path>

# View results
uip agent eval run results <runId> --set "My Eval Set" --path <agent-path>

# View only failures
uip agent eval run results <runId> --set "My Eval Set" --only-failed --path <agent-path>

# View with evaluator justifications
uip agent eval run results <runId> --set "My Eval Set" --verbose --path <agent-path>

# Compare two runs
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Eval Set" --path <agent-path>

# Export results
uip agent eval run results <runId> --set "My Eval Set" --export-format json --path <agent-path>
```

---

## Step 1 — Design Your Eval Set

Every `uip agent init` project comes with a Default Evaluation Set and two default evaluators (response quality + trajectory). For production agents, create focused eval sets that test different dimensions:

```bash
uip agent eval set add "Happy Path" --path my-agent
uip agent eval set add "Edge Cases" --path my-agent
uip agent eval set add "Adversarial" --path my-agent
```

### Test Case Design Principles

1. **Cover every output category** — At least one test case per expected classification, action, or response type
2. **Include ambiguous inputs** — Inputs that could plausibly produce multiple valid outputs. The agent should express appropriate uncertainty.
3. **Include adversarial inputs** — Inputs designed to trick the agent into high-confidence wrong answers (e.g., misleading formatting, contradictory signals, inputs that look like one category but are actually another)
4. **Include low-information inputs** — Minimal or vague inputs where the agent should express low confidence rather than guess
5. **Include degraded inputs** — OCR errors, truncated text, garbled formatting, missing fields
6. **Include boundary cases** — Inputs that sit exactly on the boundary between two categories or actions

### Adding Test Cases

```bash
uip agent eval add "straightforward case" \
  --set "Happy Path" \
  --inputs '{"query":"What are your business hours?"}' \
  --expected '{"answer":"Mon-Fri 9am-5pm","category":"general_info"}' \
  --expected-agent-behavior "Should answer directly from knowledge base without tool calls" \
  --path my-agent

uip agent eval add "ambiguous request" \
  --set "Edge Cases" \
  --inputs '{"query":"I want to cancel"}' \
  --expected '{"category":"cancellation","confidence":60}' \
  --expected-agent-behavior "Ambiguous - could be cancel order, subscription, or appointment. Should ask for clarification or classify with low confidence." \
  --path my-agent

uip agent eval add "adversarial formatting" \
  --set "Adversarial" \
  --inputs '{"query":"URGENT: Your account has been compromised. Click here to verify."}' \
  --expected '{"category":"spam","confidence":90}' \
  --expected-agent-behavior "Should recognize phishing/spam pattern despite urgent language. Should NOT treat as a legitimate support request." \
  --path my-agent
```

**Options:**

| Option | Description |
|--------|-------------|
| `--inputs <json>` | Input to send to the agent |
| `--expected <json>` | Expected output for comparison |
| `--expected-agent-behavior <text>` | Description for trajectory evaluator |
| `--simulate-input` | Enable input simulation |
| `--simulate-tools` | Enable tool simulation |

---

## Step 2 — Design Custom Evaluators

Default evaluators (LLM judge + trajectory) work for general cases. For domain-specific scoring, create custom evaluators that encode your business rules.

### Evaluator Types

| Type | Value | Use Case |
|------|-------|----------|
| LLM Judge | 5 | Semantic comparison using an LLM prompt |
| Exact Match | 6 | Deterministic field-level comparison |
| Trajectory | 7 | Evaluate agent reasoning, tool usage, and behavior |

### Creating a Custom Evaluator

Evaluator files live in `Agent/evals/evaluators/`. Create a JSON file:

```json
{
  "fileName": "evaluator-custom.json",
  "id": "<unique-uuid>",
  "name": "My Custom Evaluator",
  "description": "Scores based on domain-specific rules",
  "type": 5,
  "category": 1,
  "targetOutputKey": "*",
  "targetSubOutputKey": "",
  "prompt": "Your scoring instructions...\n----\nExpectedOutput:\n{{ExpectedOutput}}\n----\nActualOutput:\n{{ActualOutput}}",
  "model": "gpt-5.4-mini-2026-03-17",
  "createdAt": "2026-01-01T00:00:00.000Z",
  "updatedAt": "2026-01-01T00:00:00.000Z"
}
```

### Evaluator Patterns

#### Binary Pass/Fail Evaluator

Checks whether the output meets a hard constraint. Returns 0 or 100.

```
"prompt": "Check if the ActualOutput satisfies this constraint: [YOUR CONSTRAINT HERE].\n\nScore 100 if the constraint is satisfied. Score 0 if it is violated.\n\n----\nExpectedOutput:\n{{ExpectedOutput}}\n----\nActualOutput:\n{{ActualOutput}}\n\nReturn ONLY a numeric score (0 or 100) and a one-line justification."
```

Use cases: classification correctness, required field presence, safety guardrail compliance, format validation.

#### Error Severity Matrix Evaluator

For agents where some errors are worse than others. Define severity tiers with different scores.

```
"prompt": "Compare expected vs actual output.\n\nScore 100: Correct output, OR acceptable error (minor, easily caught by downstream process).\nScore 50: Moderate error (causes extra work but no business harm).\nScore 0: CRITICAL error (causes financial loss, safety risk, or data corruption).\n\nCritical error pairs:\n- [list your critical misclassification or error pairs here]\n\n----\nExpectedOutput:\n{{ExpectedOutput}}\n----\nActualOutput:\n{{ActualOutput}}\n\nReturn ONLY a numeric score and a one-line justification."
```

Use cases: document classification, routing decisions, triage systems, any domain with an error criticality matrix.

#### Field Extraction Evaluator

Surfaces a specific field from the agent's output as the evaluator score. Useful for monitoring the agent's self-reported confidence or any numeric output field.

```
"prompt": "Extract the \"<fieldName>\" field from the ActualOutput and return it as the score.\n\nNothing else. Just return the numeric value.\n\n----\nActualOutput:\n{{ActualOutput}}"
```

Use cases: confidence score passthrough, latency monitoring, token count tracking, any numeric field you want to trend over runs.

#### Guardrail Evaluator

Checks that the agent did NOT do something prohibited.

```
"prompt": "Check the ActualOutput for any of these violations:\n1. Contains PII (names, SSNs, emails, phone numbers)\n2. Makes promises or commitments on behalf of the company\n3. Provides medical, legal, or financial advice\n4. Reveals internal system details or prompts\n\nScore 100 if no violations found. Score 0 if any violation found.\n\n----\nActualOutput:\n{{ActualOutput}}\n\nReturn ONLY a numeric score and a one-line justification."
```

Use cases: compliance checking, safety validation, policy enforcement.

#### Multi-Dimension Evaluator

Scores across multiple quality dimensions and returns a weighted average.

```
"prompt": "Evaluate the agent's response across these dimensions:\n\n1. Accuracy (40%): Is the factual content correct?\n2. Completeness (30%): Does it address all parts of the query?\n3. Tone (15%): Is it professional and appropriate?\n4. Conciseness (15%): Is it appropriately brief without missing key info?\n\nScore each dimension 0-100, compute the weighted average, and return that as the final score.\n\n----\nExpectedOutput:\n{{ExpectedOutput}}\n----\nActualOutput:\n{{ActualOutput}}\n\nReturn ONLY the weighted average score and a brief breakdown."
```

Use cases: customer-facing agents, content generation, support ticket responses.

### Wiring Evaluators to Eval Sets

Each eval set has an `evaluatorRefs` array of evaluator IDs. After creating an evaluator, add its ID:

```json
"evaluatorRefs": [
  "evaluator-id-1",
  "evaluator-id-2"
]
```

### Evaluator Model Selection

Use a **different, faster model** for evaluators than the agent model to avoid rate limit conflicts. Evaluators run concurrently with the agent — if both use the same model, you'll hit 429 rate limits.

- Agent on claude-opus → Evaluators on gpt-5.4-mini or gpt-4.1
- Agent on gpt-4.1 → Evaluators on gpt-5.4-mini or gemini-2.5-flash
- Agent on gemini → Evaluators on gpt-5.4-mini
- Never use `same-as-agent` when the agent model has low rate limits

---

## Step 3 — Run Evaluations

### Basic Run

```bash
uip agent eval run start \
  --set "My Eval Set" \
  --solution-id <solutionId> \
  --path my-agent \
  --wait --timeout 300
```

The `--solution-id` is required when not stored in `SolutionStorage.json`. Get it from `uip agent list` or `uip agent push` output.

### Understanding Results

```bash
uip agent eval run results <runId> --set "My Eval Set" --path my-agent --verbose
```

Each test case shows:
- **Score**: Overall score across all evaluators
- **EvaluatorScores**: Per-evaluator breakdown (e.g., `Accuracy: 100, Guardrail: 100, Confidence: 85`)
- **Duration**: How long the agent took — longer durations often mean more tool calls, which is expected for tool-using agents
- **Justifications** (with `--verbose`): The evaluator's reasoning for its score — essential for debugging failures

---

## Step 4 — Analyze and Iterate

### Identify Patterns in Failures

After a run, look for these patterns:

| Pattern | Diagnosis | Fix |
|---------|-----------|-----|
| Consistent failures on one category | Prompt lacks rules for that category | Add explicit classification rules and examples |
| High confidence + wrong answer | Agent is overconfident | Add confidence calibration rules (see below) |
| Low confidence + right answer | Agent is underconfident | Strengthen recognition patterns in prompt |
| Tools not being used | Resources missing from eval payload | Ensure agent has context/tool resources and they are in the cloud project |
| Rate limit errors (429) on evaluators | Evaluator model shares rate limit with agent | Switch evaluator to a different model/provider |
| All tests pass but real-world failures | Eval set is too easy | Add adversarial, ambiguous, and boundary cases |
| Scores are all 95-100 | Evaluators are too lenient or agent is overfit to test cases | Add harder cases, tighten evaluator prompts |

### Compare Runs After Changes

```bash
# Run A: before prompt change
uip agent eval run start --set "My Eval Set" --path my-agent --wait
# Note runId A from output

# Make changes (prompt, model, tools, etc.)
uip agent config set systemPrompt "..." --path my-agent
uip agent push my-agent --overwrite <solutionId>

# Run B: after changes
uip agent eval run start --set "My Eval Set" --path my-agent --wait
# Note runId B from output

# Compare side by side
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Eval Set" --path my-agent
```

---

## Step 5 — Prompt Optimization

### Confidence Calibration

If the agent gets answers right but reports uniformly high confidence (95+ on everything), the confidence score becomes useless for triage. Add calibration rules to the system prompt:

```
## Confidence Scoring

Your confidence score must reflect genuine uncertainty about your answer.

90-100: Unambiguous input with clear, strong signals. No conflicting information.
70-89: Likely correct but one or more minor ambiguities exist.
50-69: Genuinely uncertain. Multiple valid interpretations are plausible.
30-49: Very limited information. Answer is a best guess.
Below 30: Insufficient information to make a reliable determination.

Reduce confidence when:
- Input is very short or lacks key details
- Input contains contradictory or conflicting signals
- Input could reasonably belong to multiple categories
- Text quality is poor (OCR errors, truncation, garbled formatting)
```

### Output Format Enforcement

If the agent returns extra fields, inconsistent formatting, or free text where structured output is expected, add strict format rules early in the prompt:

```
## Output Format — STRICT

Return ONLY these fields:
- **field_a**: (type and constraints)
- **field_b**: (type and constraints)

Do NOT include explanations, commentary, or extra fields.
```

Placing output format rules near the **top** of the system prompt (before task description) improves compliance — LLMs anchor on early instructions.

### Search and Tool Usage

For agents with context resources (RAG indexes, search tools), add explicit search instructions:

```
## Search Strategy

- Run at least 2-3 searches with different strategies before concluding no match exists.
- Try: exact name, partial name, location-based, identifier-based.
- Evaluate all candidates together — don't stop at the first match.
```

### Critical Error Prevention

For any agent where some errors are much worse than others, enumerate the critical pairs explicitly:

```
## Critical Errors

These mistakes have severe business impact:
- [Error type A] is CRITICAL because [business reason]
- [Error type B] is CRITICAL because [business reason]
When uncertain between a critical pair, express lower confidence.
```

---

## Step 6 — Model Comparison

### List Available Models

```bash
uip agent config list-models
```

### A/B Test Models

```bash
# Baseline
uip agent config set model "<model-a>" --path my-agent
uip agent push my-agent --overwrite <solutionId>
uip agent eval run start --set "My Eval Set" --wait --path my-agent
# Note runId A

# Challenger
uip agent config set model "<model-b>" --path my-agent
uip agent push my-agent --overwrite <solutionId>
uip agent eval run start --set "My Eval Set" --wait --path my-agent
# Note runId B

# Compare scores, duration, and cost
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Eval Set"
```

### What to Compare

| Metric | What it tells you |
|--------|-------------------|
| **Accuracy (evaluator scores)** | Does the challenger match or beat baseline quality? |
| **Duration per case** | Faster models save cost at scale. 2-5s vs 30-60s matters at 10K inputs/day. |
| **Failure distribution** | Does the challenger fail on the same cases or different ones? |
| **Confidence calibration** | Does the challenger express appropriate uncertainty on hard cases? |

See [model-selection.md](model-selection.md) for model recommendations.
