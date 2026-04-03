# UI Automation Playbooks

**Investigation guide:** [investigation_guide.md](./investigation_guide.md) — data correlation rules and testing prerequisites for UI Automation investigations

| Issue | Confidence | Description | Playbook |
|-------|:---:|-------------|----------|
| Selector Failure | High | SelectorNotFoundException, UiElementNotFoundException, ElementNotInteractableException, NodeNotFoundException — selector mismatch, app change, or popup blocking | [selector-failure.md](./playbooks/selector-failure.md) |
| Get Asset / Get Credential Failed | Medium | Get Asset or Get Credential activity fails with authorization, asset-not-found, folder scope, vault, network, or package version errors | [get-asset-failed.md](./playbooks/get-asset-failed.md) |
| Timeout Issue | Low | UI automation activity exceeded its timeout waiting for an element or application state | [timeout-issue.md](./playbooks/timeout-issue.md) |
| Healing Agent — No Recovery Data | Low | Healing Agent enabled but no recovery data generated after failure | [no-recovery-data.md](./playbooks/no-recovery-data.md) |
