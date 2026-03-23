# Finding Validation

Every finding passes four sequential gates. Fail any gate → **rejected** or **demoted** to lead. Later gates are not evaluated for failed findings.

## Gate 1 — Refutation

Construct the strongest argument that the finding is wrong. Find the `throw_unless`, `throw_if`, sender validation, or constraint that kills the attack — quote the exact line and trace how it blocks the claimed step.

- Concrete refutation (specific guard blocks exact claimed step) → **REJECTED** (or **DEMOTE** if code smell remains)
- Speculative refutation ("probably wouldn't happen") → **clears**, continue

## Gate 2 — Reachability

Prove the vulnerable state exists in a live deployment. Consider the asynchronous message model — state may be reachable through message sequences that are not obvious from a single handler.

- Structurally impossible (enforced invariant prevents it) → **REJECTED**
- Requires privileged actions outside normal operation → **DEMOTE**
- Achievable through normal usage, standard Jetton interactions, or common message sequences → **clears**, continue

## Gate 3 — Trigger

Prove an unprivileged actor executes the attack. Check sender validation, admin guards, and whether the attacker can craft the required messages.

- Only trusted roles can trigger → **DEMOTE**
- Costs exceed extraction (gas + message fees > profit) → **REJECTED**
- Unprivileged actor triggers profitably → **clears**, continue

## Gate 4 — Impact

Prove material harm to an identifiable victim.

- Self-harm only → **REJECTED**
- Dust-level, no compounding → **DEMOTE**
- Material loss to identifiable victim → **CONFIRMED**

## Confidence

Start at **100**, deduct: partial attack path **-20**, bounded non-compounding impact **-15**, requires specific (but achievable) state **-10**. Confidence ≥ 80 gets description + fix. Below 80 gets description only.

## Safe patterns (do not flag)

- Standard `throw_unless` sender validation checking against stored admin/owner address
- `end_parse()` enforcement after all fields loaded
- Proper bounce handlers that revert the corresponding state change
- `accept_message()` called AFTER signature/seqno validation in `recv_external`
- `raw_reserve` before `send_raw_message` to protect minimum balance
- Standard TEP-74 Jetton wallet address computation via StateInit hash
- Standard TEP-62 NFT ownership transfer with proper authorization
- Consistent protocol-favoring rounding unless compounding or zero-rounding

## Lead promotion

Before finalizing leads, promote where warranted:

- **Cross-contract echo.** Same root cause confirmed as FINDING in one contract → promote in every contract where the identical pattern appears.
- **Multi-agent convergence.** 2+ agents flagged same area, lead was demoted (not rejected) → promote to FINDING at confidence 75.
- **Partial-path completion.** Only weakness is incomplete trace but path is reachable and unguarded → promote to FINDING at confidence 75, description only.

## Leads

High-signal trails for manual investigation. No confidence score, no fix — title, code smells, and what remains unverified.

## Do Not Report

Linter/compiler issues, gas micro-opts, naming, documentation. Admin privileges by design. Missing logging (TON has no Solidity-style events). Centralization without exploit path. Implausible preconditions (but Jetton misbehavior, bounce failures, and gas exhaustion ARE plausible for contracts handling arbitrary tokens/messages).
