# TON Auditor Skills

> The ultimate Claude Code skill for TON smart contract security auditing — 120 attack vectors, 6 parallel agents, DeFi protocol checklists, and adversarial reasoning.

Built in the style of [pashov/skills](https://github.com/pashov/skills) (Solidity) but rebuilt from scratch for **TON/FunC/Tact**. Aggregates knowledge from 5+ open-source audit, development, and security repositories.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## Architecture

The same proven parallelized architecture as pashov/skills — adapted for TON's actor model, asynchronous messaging, Jetton/NFT standards (TEP-74/TEP-62), FunC's boolean semantics, cell serialization, and the TVM execution environment.

### 4-Turn Orchestration

1. **Discover** — find all in-scope `.fc`, `.func`, `.tact` files, resolve reference paths
2. **Prepare** — bundle codebase + attack vectors + judging rules into per-agent files
3. **Spawn** — launch 4–6 agents in parallel (vector scan, adversarial reasoning, protocol analysis)
4. **Report** — merge, deduplicate by root cause, sort by confidence, format

### 120 Attack Vectors (4 reference files)

| File | Vectors | Focus Areas |
| --- | --- | --- |
| [attack-vectors-1](ton-auditor/references/attack-vectors/attack-vectors-1.md) | 1–30 | Sender validation (fake Jetton, admin ops), `recv_external` safety (`accept_message` before validation), bounce handling, `end_parse`, integer-as-boolean, forward TON amount, send modes, workchain, timestamps, initialization, TEP compliance |
| [attack-vectors-2](ton-auditor/references/attack-vectors/attack-vectors-2.md) | 31–60 | Async reentrancy, race conditions, message ordering, partial execution, stale callbacks, contract upgrades (`set_code`/`set_data`), state migration, contract deletion (mode +32), gas exhaustion, storage phase failure, Tact-specific patterns |
| [attack-vectors-3](ton-auditor/references/attack-vectors/attack-vectors-3.md) | 61–90 | Integer overflow/truncation, precision loss, division-before-multiplication, rounding direction, first-depositor inflation, Jetton accounting (burn notification, supply invariant), NFT index manipulation, fee bypass, dust locking, coupled state |
| [attack-vectors-4](ton-auditor/references/attack-vectors/attack-vectors-4.md) | 91–120 | Oracle manipulation (staleness, confidence, fake oracle), flash loan price manipulation, vault share inflation, staking reward gaming, liquidation economics, bridge replay, governance flash-vote, NFT transfer policy bypass, multi-sig bugs, TVM assembly risks |

### 3 Specialized Agent Types

| Agent | Mode | Model | Approach |
| --- | --- | --- | --- |
| Vector Scan (x4) | Default + Deep | Sonnet | Systematic triage of ~30 vectors each against full codebase |
| Adversarial Reasoning | Deep only | Opus | Free-form exploit hunting with Feynman questioning, state inconsistency analysis, invariant hunting |
| TON Protocol | Deep only | Opus | Domain-specific checklists for Jetton, lending, AMM, vaults, staking, bridges, NFT/marketplace, governance |

### Quality Controls

- **FP Gate:** 3-check filter (concrete path, reachable entry, no existing guard)
- **Confidence Scoring:** Base 100 with deductions for privileged callers (-25), partial paths (-20), self-contained impact (-15), token assumptions (-10), external preconditions (-10)
- **Threshold:** Findings below 75 confidence reported without fix suggestions
- **Language-aware:** Works with FunC and Tact

---

## Install & Run

Works with **Claude Code CLI**, the **VS Code Claude extension**, and **Cursor**.

**Claude Code CLI:**

```bash
git clone https://github.com/sanbir/ton-auditor-skills.git && mkdir -p ~/.claude/commands && cp -r ton-auditor-skills/ton-auditor ~/.claude/commands/ton-auditor
```

**Cursor:**

```bash
git clone https://github.com/sanbir/ton-auditor-skills.git && mkdir -p ~/.cursor/skills && cp -r ton-auditor-skills/ton-auditor ~/.cursor/skills/ton-auditor
```

The skill is then invocable as `/ton-auditor`. See the [skill README](ton-auditor/README.md) for usage.

**Update to latest:** `cd` into the cloned repo and run:

```bash
git pull
# Claude Code CLI:
cp -r ton-auditor/ ~/.claude/commands/ton-auditor
# Cursor:
cp -r ton-auditor/ ~/.cursor/skills/ton-auditor
```

---

## Skills

| Skill | Description |
| --- | --- |
| [ton-auditor](ton-auditor/) | 120-vector security audit with 4–6 parallel agents, DeFi protocol checklists, and adversarial reasoning |

---

## What's Included

### 120 Attack Vectors (4 reference files)

Organized by attack surface:

**Message Handling, Authorization & Entry Points (V1–V30):** Fake Jetton wallet (missing sender validation on `transfer_notification`), missing sender validation on other handlers, `recv_external` accepting before validation (gas draining), replay attacks (missing seqno), bounce handler missing or broken, `end_parse()` omitted, workchain assumptions, integer-as-boolean logic errors (`~1 = -2`), forward TON amount not validated, unsafe send mode flags, admin access control, contract initialization race, signed/unsigned confusion, dictionary size unbounded, cell depth limits, `impure` missing, exit code collisions, message bit layout errors, TEP non-compliance.

**Asynchronous Execution, Concurrency & Contract Lifecycle (V31–V60):** Async reentrancy via message chains, race conditions between independent messages, message ordering assumptions, partial execution in multi-message ops, stale state after callback, contract upgrade without state migration, unsafe upgrade mechanism, `set_code` timing, mode +32 contract deletion, balance-based logic, gas exhaustion in loops, stack depth overflow, storage phase failure, account state limits, StateInit missing in deploy, address computation mismatch, replay on wallet ops, Tact ownership enforcement, Tact fallback misuse, Tact struct encoding, `throw_unless`/`throw_if` polarity, gas reservation before send.

**Arithmetic, Token Operations & State Management (V61–V90):** Integer overflow/truncation, division-before-multiplication, division by zero, rounding direction, first-depositor vault inflation, coin amount truncation, fee bypass, fee-on-transfer tokens, dust locking, supply invariant violation, double-spend race, Jetton wallet balance manipulation, NFT index manipulation, `store_coins`/`load_coins` misuse, cross-contract state dependency, dictionary key mismatch, coupled state inconsistency, accumulator precision loss, raw vs tracked balance confusion, burn without notification.

**Oracle, DeFi & Platform-Level (V91–V120):** Stale oracle, confidence interval ignored, fake oracle contract, single oracle dependency, flash loan price manipulation, vault share inflation, staking reward gaming, flash stake capture, reward dilution, liquidation incentive gaps, self-liquidation profit, interest during pause, bad debt not socialized, DEX slippage from on-chain, missing swap deadline, AMM invariant violation, bridge replay, bridge supply invariant, governance flash-vote, timelock missing, NFT transfer policy bypass, NFT ownership spoofing, multi-sig counting errors, proxy forwarding, Tact init re-entrancy, TVM assembly risks.

### Protocol Checklists (81 items across 8 domains)

| Domain | Items | Key Checks |
| --- | --- | --- |
| Jetton (Token) | 12 | `transfer_notification` sender validated, `burn_notification` decrements supply, bounce handlers, wallet address computation, `raw_reserve` |
| Lending/Borrowing | 14 | Health factor includes accrued interest, liquidation incentive covers gas, bad debt socialization |
| AMM/DEX | 10 | Slippage from user message, deadline enforced, constant product verified, flash swap authorized |
| Vault/Token Accounting | 10 | First-depositor mitigated, rounding correct, share price not manipulable via direct transfer |
| Staking/Rewards | 10 | Accumulator updated before balance change, no flash stake, precision for small stakers |
| Bridge/Cross-Chain | 9 | Replay protection, rate limits, supply invariant, decimal conversion |
| NFT/Marketplace | 10 | Ownership transfer restricted, royalties enforced, auction deadlines, TEP-62 compliance |
| Governance | 6 | Vote weight from past snapshot, timelock, quorum, no double-voting |

---

## Attributions

This skill aggregates knowledge from the following open-source repositories. We are grateful to all contributors.

### Architecture Inspiration

| Repository | Author | Contribution |
| --- | --- | --- |
| [pashov/skills](https://github.com/pashov/skills) | Pashov Audit Group | Parallelized agent orchestration pattern, FP gate, confidence scoring, vector-scan and adversarial-reasoning agent design, report formatting — adapted from Solidity/EVM to TON/FunC/Tact |

### TON Security Knowledge

| Repository | Author | Contribution |
| --- | --- | --- |
| [trailofbits/building-secure-contracts](https://github.com/trailofbits/building-secure-contracts) | Trail of Bits | 3 critical TON vulnerability patterns with detailed code examples (fake Jetton contract/sender validation, integer-as-boolean/bitwise NOT logic errors, forward TON without gas check/balance drainage), FunC-specific detection patterns, multiple Jetton support considerations, send mode flag reference |
| [nicholasgasior/ton-audit-guide](https://github.com/nicholasgasior/ton-audit-guide) | TON Audit Guide | Comprehensive TON audit checklist covering message flow analysis, entry point identification, `recv_external` validation, bounce handling, logical errors (division before multiplication, unrestricted data recording), storage compatibility during upgrades, signed/unsigned number handling, loop safety, `end_parse` verification, Tact-specific considerations (exit codes, ownership patterns) |
| [nicholasgasior/ton-audit-ai](https://github.com/nicholasgasior/ton-audit-ai) | TonAudit AI | 15+ vulnerability categories for TON contracts (async reentrancy, bounce handling, sender validation, storage exhaustion, TEP compliance, send mode errors, access control, integer issues, state management, gas griefing, replay attacks), sample vulnerable contracts (escrow) and secure patterns (Jetton minter), audit system prompt architecture |
| [nicholasgasior/finite-monkey-engine](https://github.com/nicholasgasior/finite-monkey-engine) | Finite Monkey Engine | AI-driven code security analysis platform with planning-reasoning-validation pipeline, Tree-sitter based code parsing for vulnerability scanning, multi-model collaboration architecture (Claude, GPT-4, DeepSeek), structured audit finding format |

### Methodology & Agents

| Repository | Author | Contribution |
| --- | --- | --- |
| [sainikethan/nemesis-auditor](https://github.com/sainikethan/nemesis-auditor) | Nemesis | Feynman questioning strategy, state inconsistency analysis methodology — adapted for adversarial reasoning agent |
| [carni-ships/SolidSecs](https://github.com/carni-ships/SolidSecs) | SolidSecs | Protocol-specific checklist methodology (lending, AMM, vault, staking, bridge, governance) — adapted for TON protocol agent |
| [auditmos/skills](https://github.com/auditmos/skills) | Auditmos | Lending protocol vulnerability patterns, liquidation mechanics, staking reward edge cases — adapted for DeFi attack vectors |

---

## Real-World TON Security Considerations

| Pattern | Severity | Description |
| --- | --- | --- |
| Fake Jetton Wallet | Critical | `transfer_notification` accepted from any sender without validating against stored Jetton wallet address — attacker credits themselves with tokens never transferred |
| Integer-as-Boolean | High | FunC `true = -1`, using `1` as true causes `~1 = -2` (truthy, not false) — security checks inverted |
| Forward TON Gas Drain | High | User-controlled `forward_ton_amount` paid from contract balance, draining it over repeated calls |
| `accept_message` Before Validation | High | External message handler accepts (pays gas) before validating signature — attackers drain contract by spamming invalid messages |
| Missing Bounce Handler | High | State updated before sending message, message bounces, no rollback — funds lost or state corrupted |

---

## License

[MIT](LICENSE) — see individual attribution repos for their respective licenses.
