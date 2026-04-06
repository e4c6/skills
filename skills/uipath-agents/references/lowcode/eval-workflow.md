# Low-Code Agent Evaluation Workflow

> **Agent type: Low-code agents using `uip agent` commands.** For coded agents using `uip codedagents eval`, see [running-evaluations.md](../lifecycle/evaluations/running-evaluations.md).

This guide covers the complete evaluation workflow for low-code agents using the native `uip agent eval` commands — creating eval sets, adding test cases, designing custom evaluators, running evaluations, analyzing results, and iterating on prompts.

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

# List test cases
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

Every `uip agent init` project comes with a Default Evaluation Set and two default evaluators (response quality + trajectory). For production agents, create focused eval sets:

```bash
uip agent eval set add "Happy Path" --path my-agent
uip agent eval set add "Edge Cases" --path my-agent
uip agent eval set add "Critical Error Traps" --path my-agent
```

### Test Case Design Principles

1. **Cover every category** — At least one test case per expected output category
2. **Include ambiguous cases** — Documents that could plausibly belong to multiple categories. Set lower expected confidence.
3. **Include adversarial cases** — Inputs specifically designed to trick the agent into critical errors
4. **Include low-information cases** — Minimal input where the agent should express uncertainty
5. **Include degraded input** — OCR errors, truncated documents, garbled text

### Adding Test Cases

```bash
uip agent eval add "clear authorization" \
  --set "Happy Path" \
  --inputs '{"fileContent":"APPROVED. Auth #12345. Service authorized for 04/01-04/30."}' \
  --expected '{"classification":"Auths","confidenceScore":95}' \
  --expected-agent-behavior "Clear approval language. Should classify as Auths with high confidence." \
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

Default evaluators (LLM judge + trajectory) work for general cases. For domain-specific scoring, create custom evaluators.

### Evaluator Types

| Type | Value | Use Case |
|------|-------|----------|
| LLM Judge | 5 | Semantic comparison using an LLM |
| Exact Match | 6 | Deterministic field comparison |
| Trajectory | 7 | Evaluate agent reasoning and tool usage |

### Creating a Custom Evaluator

Evaluator files live in `Agent/evals/evaluators/`. Create a JSON file:

```json
{
  "fileName": "evaluator-custom.json",
  "id": "unique-uuid",
  "name": "My Custom Evaluator",
  "description": "Scores based on domain-specific rules",
  "type": 5,
  "category": 1,
  "targetOutputKey": "*",
  "targetSubOutputKey": "",
  "prompt": "Your scoring prompt here...\n----\nExpectedOutput:\n{{ExpectedOutput}}\n----\nActualOutput:\n{{ActualOutput}}",
  "model": "gpt-4.1-2025-04-14",
  "createdAt": "2026-01-01T00:00:00.000Z",
  "updatedAt": "2026-01-01T00:00:00.000Z"
}
```

### Example: Criticality Matrix Evaluator

For classification agents with critical error pairs:

```json
{
  "prompt": "Compare expected vs actual classification.\n\nScore 100: Correct match, acceptable error, or medium-risk error.\nScore 0: CRITICAL misclassification (list your critical pairs here).\n\n----\nExpectedOutput:\n{{ExpectedOutput}}\n----\nActualOutput:\n{{ActualOutput}}\n\nReturn ONLY a numeric score (0 or 100) and a one-line justification.",
  "model": "gpt-5.4-mini-2026-03-17"
}
```

### Example: Confidence Passthrough Evaluator

Surface the agent's own confidence score as an evaluator metric:

```json
{
  "prompt": "Extract the \"confidenceScore\" field from the ActualOutput and return it as the score.\n\nNothing else. Just return the numeric confidenceScore value from the agent's output.\n\n----\nActualOutput:\n{{ActualOutput}}",
  "model": "gpt-5.4-mini-2026-03-17"
}
```

### Wiring Evaluators to Eval Sets

Each eval set has an `evaluatorRefs` array of evaluator IDs. After creating an evaluator, add its ID to the eval set:

```python
# In the eval set JSON file:
"evaluatorRefs": [
  "evaluator-id-1",
  "evaluator-id-2"
]
```

### Evaluator Model Selection

Use a **different, faster model** for evaluators than the agent model to avoid rate limit conflicts:
- Agent on Opus → Evaluators on gpt-4.1 or gpt-5.4-mini
- Agent on gpt-4.1 → Evaluators on gpt-5.4-mini or gemini-2.5-flash
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
- **EvaluatorScores**: Per-evaluator breakdown
- **Duration**: How long the agent took (longer = more tool calls = better for tool-using agents)
- **Justifications** (with `--verbose`): Why each evaluator gave its score

---

## Step 4 — Analyze and Iterate

### Identify Patterns

After a run, look for:

1. **Consistent failures on a category** → Prompt needs better rules for that category
2. **High confidence + wrong answer** → Agent is overconfident — add calibration rules
3. **Low confidence + right answer** → Agent is underconfident — strengthen recognition patterns
4. **Tool not being used** → Check that resources are included in the eval payload and the agent has access to the tool
5. **Rate limit errors on evaluators** → Switch evaluator models to a different provider

### Compare Runs After Changes

```bash
# Run A: before prompt change
uip agent eval run start --set "My Eval Set" --path my-agent --wait
# Note runId A

# Make prompt changes...
uip agent config set systemPrompt "..." --path my-agent
uip agent push my-agent --overwrite <solutionId>

# Run B: after prompt change
uip agent eval run start --set "My Eval Set" --path my-agent --wait
# Note runId B

# Compare
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Eval Set" --path my-agent
```

---

## Step 5 — Prompt Optimization

### Confidence Calibration

If the agent scores correctly but with uniformly high confidence (95+ on everything), add calibration rules to the system prompt:

```
## Confidence Calibration — STRICT

90-100: Unambiguous. Clear category-specific language. No conflicting signals.
70-89: Clear category but minor ambiguity. One confusing phrase in otherwise clear document.
50-69: Genuinely ambiguous. Multiple categories plausible.
30-49: Very low information or heavily degraded input.
Below 30: Almost no usable content.

Hard caps:
- Document under 50 words → cap at 60
- Key determination word is ambiguous → cap at 75
- Contains language from 2+ categories → cap at 70
- OCR artifacts corrupt >20% of text → cap at 50
```

### Critical Error Prevention

For classification agents with error criticality matrices, add explicit rules:

```
## Critical Misclassification Rules
Some misclassifications have severe business impact:
- Classifying X as Y is CRITICAL (reason)
- Classifying Y as X is CRITICAL (reason)
When in doubt between a critical pair, lower your confidence score.
```

---

## Step 6 — Model Comparison

### List Available Models

```bash
uip agent config list-models
```

### A/B Test Models

```bash
# Baseline: gpt-4.1
uip agent config set model "gpt-4.1-2025-04-14" --path my-agent
uip agent push my-agent --overwrite <solutionId>
uip agent eval run start --set "My Eval Set" --wait --path my-agent
# Note runId A

# Challenger: claude-sonnet
uip agent config set model "anthropic.claude-sonnet-4-6" --path my-agent
uip agent push my-agent --overwrite <solutionId>
uip agent eval run start --set "My Eval Set" --wait --path my-agent
# Note runId B

# Compare scores, duration, and cost
uip agent eval run compare <runIdA> --compare-to <runIdB> --set "My Eval Set"
```

### Model Selection Criteria

| Factor | Faster Models (mini/flash) | Larger Models (opus/gpt-4.1) |
|--------|---------------------------|------------------------------|
| Simple classification | Sufficient | Overkill |
| Tool-using agents | May miss complex search strategies | Better multi-step reasoning |
| Ambiguous inputs | Lower accuracy on edge cases | Better judgment calls |
| Cost per eval run | Lower | Higher |
| Latency per case | 2-5s | 10-60s |
