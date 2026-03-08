# ton-auditor

Security audit skill for TON smart contracts (FunC/Tact). Uses 120 attack vectors, up to 6 parallel agents, and DeFi protocol checklists.

## Usage

```
/ton-auditor              # scan all .fc/.func/.tact files (4 agents)
/ton-auditor deep         # + adversarial reasoning + protocol analysis (6 agents)
/ton-auditor contract.fc  # scan specific file(s)
```

## Flags

- `--file-output` — write report to a markdown file in `assets/findings/`

## Modes

| Mode | Agents | Cost |
| --- | --- | --- |
| Default | 4 vector-scan (Sonnet) | Low |
| Deep | 4 vector-scan + adversarial (Opus) + protocol (Opus) | High |
| File | 4 vector-scan on specified files | Low |

## Output

Findings sorted by confidence (highest first), deduplicated by root cause, with fix suggestions for findings above the confidence threshold.
