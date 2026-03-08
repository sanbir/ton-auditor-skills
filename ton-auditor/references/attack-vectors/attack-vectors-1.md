# Attack Vectors Reference (1/4) — Message Handling, Authorization & Entry Points

## V1 — Missing Sender Validation on `transfer_notification`

**What:** `transfer_notification` (op `0x7362d09c`) handler accepts tokens from any sender without verifying it came from a legitimate Jetton wallet.

**Why it matters:** Attacker deploys a contract that sends fake `transfer_notification` messages, crediting themselves with tokens never actually transferred. This is the #1 TON exploit pattern.

**What to look for:**
- `recv_internal` handler for `op::transfer_notification` that does NOT compare `sender_address` against stored Jetton wallet address
- Trusting `forward_payload` data without sender validation
- Using the `from_user` field in the notification body as proof of deposit

**Secure pattern:** Store legitimate Jetton wallet addresses during initialization; `throw_unless(error::wrong_jetton_wallet, equal_slices(sender_address, jetton_wallet_address))` before processing.

---

## V2 — Missing Sender Validation on Other Internal Messages

**What:** Any `recv_internal` handler that processes operations without verifying the sender is an authorized contract or address.

**Why it matters:** TON's actor model means any contract can send any message to any other contract. Without sender checks, attackers can invoke privileged operations.

**What to look for:**
- `op` handlers that modify state without checking `sender_address`
- Missing `throw_unless` for sender validation on admin operations
- Relying on message content alone (not sender identity) for authorization

---

## V3 — `recv_external` Accepts Before Validation

**What:** `accept_message()` called before validating the external message signature or sequence number.

**Why it matters:** `accept_message()` tells the VM to charge gas from the contract's balance. If called before validation, attackers send invalid external messages and the contract pays gas for each one, draining its balance.

**What to look for:**
- `accept_message()` appearing before `check_signature()` or `throw_unless(seqno == stored_seqno)`
- External message handlers that accept unconditionally
- Missing replay protection (sequence numbers) in external messages

**Secure pattern:** Parse signature → validate signature → validate seqno → `accept_message()` → execute.

---

## V4 — Missing Sequence Number / Replay Protection

**What:** External messages (`recv_external`) lack sequence number or nonce validation, allowing message replay.

**Why it matters:** Without replay protection, a valid external message can be re-submitted indefinitely. For wallets, this means re-executing transfers.

**What to look for:**
- `recv_external` without `seqno` check
- Sequence number incremented AFTER execution (should be before or atomic)
- Missing `throw_unless(error::bad_seqno, seqno == stored_seqno)`

---

## V5 — Missing Bounce Handler

**What:** Contract sends messages but has no `bounced` message handler, or the handler doesn't properly revert state.

**Why it matters:** If a sent message bounces (recipient rejects, not found, or out of gas), the sender must handle the bounce to revert state changes. Without a handler: state updated but action failed = inconsistent state, potential fund loss.

**What to look for:**
- `send_raw_message()` calls without corresponding bounce handling
- `recv_internal` that doesn't check `msg_flags & 1` (bounced flag)
- State changes (credits, debits) made before sending with no rollback on bounce
- Jetton transfers without bounce recovery

---

## V6 — Improper Bounce Message Parsing

**What:** Bounce handler exists but parses the bounced message incorrectly — wrong op extraction, missing 32-bit prefix skip, or incorrect body parsing.

**Why it matters:** Bounced messages have a 32-bit `0xFFFFFFFF` prefix before the original body. If the handler doesn't skip this prefix, it misparses the message and fails to revert correctly.

**What to look for:**
- Bounce handler not skipping the first 32 bits of the body
- Parsing op from wrong position in bounced message
- Handler that silently ignores parse errors instead of reverting state

---

## V7 — Missing `end_parse()` After Deserialization

**What:** After reading data from a slice (storage or message), `end_parse()` is not called to verify no trailing bytes remain.

**Why it matters:** Without `end_parse()`, extra data in messages or storage goes undetected. This can mask injection attacks, storage corruption, or version incompatibilities.

**What to look for:**
- `begin_parse()` followed by `load_*` operations without a final `end_parse()`
- Storage loading (`get_data().begin_parse()`) without end verification
- Message body parsing that doesn't verify completeness

---

## V8 — Workchain Assumption

**What:** Contract assumes all addresses are in workchain 0 (basechain) without validating the workchain ID.

**Why it matters:** Valid TON addresses can be in workchain -1 (masterchain) or other workchains. Operations may behave differently across workchains, and address comparison can fail if workchain is not validated.

**What to look for:**
- `force_chain()` or workchain validation missing on incoming addresses
- Address construction that hardcodes workchain 0
- Missing `check_same_workchain()` on sender or destination addresses

---

## V9 — Integer-as-Boolean Logic Error

**What:** FunC uses -1 (all bits set) as true and 0 as false, but code uses positive integers (1, 2, etc.) as boolean values, causing bitwise NOT (`~`) to produce unexpected results.

**Why it matters:** `~1 = -2` (truthy, not false!), `~2 = -3` (truthy). Code like `if (~ is_active)` where `is_active = 1` will execute the "inactive" branch even though the intent was to check inactivity. This is a well-documented FunC footgun.

**What to look for:**
- Variables set to `1` that are later used with `~` (bitwise NOT)
- `load_uint(1)` returning 0 or 1 (not -1!) used directly in boolean logic with `~`
- Functions returning `1` for true instead of `-1`
- Boolean operations `~`, `&`, `|` on non-canonical boolean values

**Secure pattern:** Use `const int TRUE = -1; const int FALSE = 0;` or normalize: `int flag = -(cs~load_uint(1));`

---

## V10 — Forward TON Amount Not Validated

**What:** User-controlled `forward_ton_amount` in outgoing messages is not bounded or validated against `msg_value`.

**Why it matters:** Attacker specifies a large `forward_ton_amount` while sending minimal gas. The contract pays the difference from its own balance, draining it over repeated calls.

**What to look for:**
- `in_msg_body~load_coins()` used directly as `forward_ton_amount` in `send_raw_message`
- No check that `msg_value >= tx_fee + forward_ton_amount`
- `send_raw_message` with mode 1 (pay fees from contract) when forward amount is user-controlled
- Missing upper bound on forward amounts

**Secure pattern:** Use fixed forward amounts, or validate `msg_value` covers all costs, or use mode 64 (return remaining incoming value).

---

## V11 — Unsafe Send Mode Flags

**What:** Incorrect `send_raw_message` mode flags leading to unexpected gas payment or balance behavior.

**Why it matters:** Mode flags control who pays gas and how much value is sent:
- Mode 0: send specified value, pay fees from message
- Mode 1: pay fees separately from contract balance
- Mode 64: return remaining incoming message value
- Mode 128: carry ALL remaining contract balance
- +2: ignore errors
- +32: destroy contract if balance reaches 0

Using mode 128 with user input or mode 1 without validation can drain contracts.

**What to look for:**
- Mode 128 used when contract should retain balance
- Mode 1 with user-controlled amounts (contract pays gas)
- Mode +2 (ignore errors) masking critical failures
- Mode +32 accidentally destroying contracts
- Missing mode flag on `send_raw_message` (defaults to 0)

---

## V12 — Missing Access Control on Admin Operations

**What:** Operations like parameter updates, pausing, upgrading, or fund withdrawal lack owner/admin address checks.

**Why it matters:** Anyone can call admin functions, changing contract parameters, extracting funds, or upgrading code.

**What to look for:**
- `op` handlers for administrative functions without `equal_slices(sender_address, admin_address)` checks
- Missing `throw_unless(error::not_owner, ...)` on privileged operations
- Admin address stored but never checked in handlers

---

## V13 — Admin Address Not Updatable or Not Secured

**What:** Admin/owner address is hardcoded at deployment with no transfer mechanism, or the transfer isn't two-step.

**Why it matters:** If the admin key is compromised or lost, there's no recovery. Single-step transfer risks sending admin to wrong address.

**What to look for:**
- No `change_admin` or `transfer_ownership` operation
- Direct admin address change without pending/confirm pattern
- Admin address stored in code (not data) making it non-upgradeable

---

## V14 — Gas Draining via External Messages

**What:** Contract processes external messages in a way that allows attackers to force gas consumption from the contract's balance.

**Why it matters:** Each `accept_message()` charges gas from the contract. Attackers can spam invalid-but-accepted external messages to drain the contract to zero, causing it to freeze.

**What to look for:**
- `accept_message()` called before cheap validation (signature check should come first)
- External message handlers that do expensive computation before validation
- No rate limiting or minimum balance protection

---

## V15 — Storage Fee Exhaustion / Contract Freezing

**What:** Contract doesn't reserve minimum balance for storage fees, or allows operations that reduce balance below storage costs.

**Why it matters:** TON contracts must pay ongoing storage fees. If balance drops below the minimum, the contract is frozen (inaccessible). Attackers can deplete contract balance through small withdrawals or gas draining.

**What to look for:**
- Missing `raw_reserve()` calls to protect minimum balance
- Operations that send the entire contract balance (mode 128) without reserve
- No minimum balance check before sending funds
- Long-lived contracts without storage fee consideration

**Secure pattern:** `raw_reserve(MIN_TON_FOR_STORAGE, RESERVE_REGULAR)` at the start of critical operations.

---

## V16 — Unvalidated Message Opcode Handling

**What:** `recv_internal` doesn't handle unknown opcodes, or silently accepts messages with unrecognized operations.

**Why it matters:** Unknown opcodes should either bounce or be explicitly ignored. Silent acceptance can cause unexpected behavior or hide attack vectors.

**What to look for:**
- Missing `else` branch or default handler for unknown opcodes
- No `throw(error::unknown_op)` for unrecognized operations
- Opcode 0 (simple transfer) not handled separately from operational messages

---

## V17 — Missing Bounceable/Non-bounceable Address Distinction

**What:** Contract sends funds to bounceable addresses when non-bounceable is needed, or vice versa.

**Why it matters:** Sending to a bounceable address of a non-existent contract will bounce back (good for safety). Sending to non-bounceable means funds go to an uninitialized account (may be intended for user wallets). Confusion can cause fund loss.

**What to look for:**
- Address flags not set correctly in `store_uint(0x18, 6)` vs `store_uint(0x10, 6)`
- Sending to contracts with non-bounceable flag
- Sending to user wallets with bounceable flag (funds bounce if wallet not deployed)

---

## V18 — Insufficient Gas for Message Chains

**What:** Contract initiates a chain of internal messages but doesn't ensure enough gas/TON value propagates through the chain.

**Why it matters:** Multi-hop message chains (A → B → C) need sufficient value at each step. If gas runs out mid-chain, later messages aren't sent, leaving state inconsistent.

**What to look for:**
- Long message chains without gas estimation
- Fixed `forward_ton_amount` that doesn't account for downstream costs
- Missing value propagation in multi-contract workflows

---

## V19 — Timestamp / Clock Manipulation

**What:** Contract uses `now` (current Unix timestamp) for time-sensitive logic without accounting for validator timestamp flexibility.

**Why it matters:** Validators have some flexibility in setting block timestamps. For short time windows, this can be exploited.

**What to look for:**
- `now` used for auction deadlines with very tight windows
- Time-based access control with precision under ~30 seconds
- Missing deadline parameters where time matters

---

## V20 — Contract Initialization Race Condition

**What:** Contract can be initialized by anyone, or re-initialized after deployment.

**Why it matters:** If initialization isn't restricted to the deployer or isn't one-time-only, attackers can re-initialize with malicious parameters.

**What to look for:**
- Initialization function callable by anyone (no deployer check)
- Missing `is_initialized` flag to prevent re-initialization
- State that can be overwritten by sending specific opcodes
- `set_data()` callable without proper authorization

---

## V21 — Signed/Unsigned Integer Confusion

**What:** Code loads signed integers where unsigned is expected, or vice versa, leading to unexpected negative values or overflow.

**Why it matters:** `load_int(256)` can return negative values; `load_uint(256)` cannot. If financial amounts are loaded as signed, negative values bypass positive-amount checks.

**What to look for:**
- `load_int()` used for amounts, balances, or sizes that should never be negative
- Comparison operators that don't account for signedness
- Signed arithmetic on values that should be unsigned
- Missing range validation on signed inputs

---

## V22 — TON/Gram Amount Validation Missing

**What:** Operations that transfer TON don't validate that the amount is positive and within reasonable bounds.

**Why it matters:** Zero-amount transfers waste gas; extremely large amounts can exceed contract balance; negative amounts (if signed) bypass checks.

**What to look for:**
- Missing `throw_unless(error::invalid_amount, amount > 0)`
- No upper bound validation on transfer amounts
- Amount loaded as signed integer (could be negative)
- Missing check that contract balance covers the transfer

---

## V23 — Dictionary/Hashmap Size Unbounded

**What:** Contract stores data in a dictionary (hashmap) without limiting the number of entries.

**Why it matters:** Unbounded dictionary growth increases storage costs. Eventually, operations on large dictionaries exceed gas limits or storage fees drain the contract.

**What to look for:**
- `dict_set` / `udict_set` without a maximum size check
- No cleanup/pruning mechanism for old entries
- User-controlled dictionary keys allowing unlimited entries
- Missing `throw_unless(error::dict_full, dict_size < MAX_SIZE)`

---

## V24 — Cell Depth/Size Limits Not Checked

**What:** Contract builds cells exceeding TON's limits (1023 bits per cell, max 4 references per cell, max 256 cell depth).

**Why it matters:** Exceeding these limits causes runtime errors. Attackers can craft inputs that force the contract to build oversized cells, causing transactions to fail.

**What to look for:**
- Dynamic data packed into cells without size checks
- Recursive cell structures that could exceed depth limits
- User-provided data stored directly in cells without length validation

---

## V25 — Missing `impure` Specifier

**What:** Functions that modify state or send messages are missing the `impure` specifier in FunC.

**Why it matters:** Without `impure`, the FunC compiler may optimize away the function call if its return value is unused. This can silently skip critical state updates or message sends.

**What to look for:**
- Functions containing `send_raw_message()`, `set_data()`, `set_code()`, or `raw_reserve()` without `impure`
- State-modifying helper functions missing `impure`
- `impure` present on entry points but missing on called subroutines

---

## V26 — Exit Code Collision with TON Reserved Range

**What:** Custom error codes use values 0–127 (reserved by TON) or 128–255 (reserved by Tact), causing confusion during debugging or incorrect error handling.

**Why it matters:** TON reserves exit codes 0–127 for system errors. Using these for custom errors means the contract's error might be confused with a system error, making debugging difficult and error handling unreliable.

**What to look for:**
- `throw()` or `throw_unless()` with codes in the range 0–127
- Tact contracts using error codes 128–255
- No clear error code numbering scheme

---

## V27 — Recursive/Unbounded Message Loops

**What:** Contract A sends to contract B, which sends back to A, creating an infinite or very long message loop.

**Why it matters:** Message loops consume gas at each hop. While they eventually stop when gas runs out, they can drain contract balances and create inconsistent state.

**What to look for:**
- Circular message patterns between contracts
- Bounce handlers that send new messages which might bounce back
- Missing loop detection/breaking mechanisms

---

## V28 — Incorrect Bit/Ref Layout in Messages

**What:** Message serialization doesn't match expected deserialization format — wrong bit widths, missing references, or incorrect field ordering.

**Why it matters:** Mismatched serialization/deserialization causes silent data corruption. Fields are read from wrong positions, leading to incorrect amounts, addresses, or opcodes.

**What to look for:**
- `store_uint` bit width doesn't match corresponding `load_uint`
- References stored/loaded in wrong order
- Optional fields handled inconsistently between sender and receiver
- Standard message headers not following TL-B schema

---

## V29 — State Corruption on Partial Execution

**What:** Multi-step state update where some steps succeed but later steps fail (out of gas, throw), leaving state partially updated.

**Why it matters:** TON transactions are atomic within a single contract call, but state is committed if the transaction succeeds up to the point of the throw. If `set_data()` is called before a throw, state is already saved.

**What to look for:**
- `set_data()` called early in execution, before operations that might throw
- Multiple `set_data()` calls where later ones might fail
- State updates split across multiple operations without atomicity

**Note:** In FunC, `set_data()` sets c4 register; the actual commit happens at transaction end. A throw will revert c4. BUT if one transaction sends a message that triggers another transaction, the first transaction's state IS committed even if the second fails.

---

## V30 — TEP Standard Non-Compliance

**What:** Jetton (TEP-74), NFT (TEP-62), or other standard implementations deviate from the TEP specification.

**Why it matters:** Non-compliant implementations may be incompatible with wallets, DEXes, and other ecosystem contracts. Deviations can also introduce security vulnerabilities.

**What to look for:**
- Jetton op codes not matching TEP-74 specification
- Missing required getters (`get_wallet_address`, `get_jetton_data`)
- Non-standard message formats that break interoperability
- Missing required fields in transfer/notification messages
