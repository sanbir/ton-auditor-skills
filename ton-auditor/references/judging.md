# Finding Validation

Each finding passes a false-positive gate, then gets a confidence score (how certain you are it is real).

## FP Gate

Every finding must pass all three checks. If any check fails, drop the finding — do not score or report it.

1. You can trace a concrete attack path: sender → message handler → state change → loss/impact. Evaluate what the code _allows_, not what the deployer _might choose_.
2. The entry point is reachable by the attacker (check sender validation, `throw_unless`/`throw_if` guards, admin-only restrictions, bounced message checks).
3. No existing guard already prevents the attack (`throw_unless`, `throw_if`, sender address comparison, balance checks, sequence number validation, `end_parse()`, etc.).

## Confidence Score

Confidence measures certainty that the finding is real and exploitable — not how severe it is. Every finding that passes the FP gate starts at **100**.

**Deductions (apply all that fit):**

- Privileged caller required (admin, owner, multi-sig) → **-25**.
- Attack path is partial (general idea is sound but cannot write exact sender → handler → state change → outcome) → **-20**.
- Impact is self-contained (only affects the attacker's own funds, no spillover to other users) → **-15**.
- Requires specific token behavior (non-standard Jettons, custom wallets) that may not apply to whitelisted tokens → **-10**.
- Requires external precondition (oracle failure, cross-chain message delay, specific validator behavior) → **-10**.

Confidence indicator: `[score]` (e.g., `[95]`, `[75]`, `[60]`).

Findings below the confidence threshold (default 75) are still included in the report table but do not get a **Fix** section — description only.

## Do Not Report

- Anything a linter, compiler, or seasoned FunC/Tact developer would dismiss — INFO-level notes, gas micro-optimizations, naming, documentation, redundant comments.
- Admin/owner can set fees, parameters, or pause — these are by-design privileges, not vulnerabilities.
- Missing event emissions or insufficient logging (TON doesn't have Solidity-style events).
- Centralization observations without a concrete exploit path (e.g., "admin could rug" with no specific mechanism beyond trust assumptions).
- Theoretical issues requiring implausible preconditions (e.g., compromised validator, >50% token supply held by attacker).
- Gas optimization suggestions without security impact.
