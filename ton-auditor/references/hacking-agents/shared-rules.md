# Shared Scan Rules

## Reading

Your bundle has two sections:

1. **Core source** (inline) — read in parallel chunks (offset + limit), compute offsets from the line count in your prompt.
2. **Peripheral file manifest** — file paths under `# Peripheral Files (read on demand)`. Read only those relevant to your specialty.

When matching function names in FunC, check both `function_name` and `_function_name` (underscore-prefixed internal helpers). In Tact, check both `receive()` handlers and named functions.

## Cross-contract patterns

When you find a bug in one contract, **weaponize that pattern across every other contract in the bundle.** Search by handler name AND by code pattern. Finding missing sender validation in `ContractA::recv_internal` means you check every other contract's `recv_internal` — missing a repeat instance is an audit failure.

In TON, cross-contract interaction happens via asynchronous message chains, not function calls. Trace message flows: sender → `send_raw_message` → recipient's `recv_internal`. Follow the full A→B→C chain, checking each leg for validation gaps.

After scanning: escalate every finding to its worst exploitable variant (DoS may hide fund theft). Then revisit every handler where you found something and attack the other branches.

## Do not report

Admin-only handlers doing admin things. Standard FunC/Tact patterns (standard Jetton/NFT implementations following TEP-74/TEP-62). Self-harm-only bugs. "Admin can rug" without a concrete mechanism. Gas micro-optimizations. Missing logging (TON has no Solidity-style events).

## Output

Return structured blocks only — no preamble, no narration. Exception: vector scan agent outputs its classification block first.

FINDINGs have concrete, unguarded, exploitable attack paths. LEADs have real code smells with partial paths — default to LEAD over dropping.

**Every FINDING must have a `proof:` field** — concrete values, traces, or message sequences from the actual code. No proof = LEAD, no exceptions.

**One vulnerability per item.** Same root cause = one item. Different fixes needed = separate items.

```
FINDING | contract: Name | handler: func | bug_class: kebab-tag | group_key: Contract | handler | bug-class
path: sender → handler → state change → impact
proof: concrete values/trace demonstrating the bug
description: one sentence
fix: one-sentence suggestion

LEAD | contract: Name | handler: func | bug_class: kebab-tag | group_key: Contract | handler | bug-class
code_smells: what you found
description: one sentence explaining trail and what remains unverified
```

The `group_key` enables deduplication: `ContractName | handlerName | bug_class`. Agents may add custom fields.
