# Attack Vectors Reference (2/4) — Asynchronous Execution, Concurrency & Contract Lifecycle

## V31 — Asynchronous Reentrancy via Message Chains

**What:** Contract updates state, sends a message to another contract, and assumes the callback will see the original state — but another message arrives and modifies state before the callback.

**Why it matters:** TON's asynchronous model means reentrancy works through message ordering. Unlike Solidity's same-transaction reentrancy, TON reentrancy occurs across multiple transactions with state committed between them.

**What to look for:**
- State update → send message → await callback pattern without state locking
- No "processing" flag to prevent concurrent operations
- State that can be modified by other messages while awaiting callback
- Lack of logical time or sequence number to detect stale callbacks

---

## V32 — Race Condition Between Internal Messages

**What:** Two independent message chains can modify the same state concurrently, and the outcome depends on which message is processed first.

**Why it matters:** Message ordering between different sender contracts is NOT guaranteed on TON. If two users interact with the same contract simultaneously, the processing order depends on validators.

**What to look for:**
- Shared mutable state accessed by multiple independent message flows
- First-come-first-served patterns without explicit ordering
- Auction/bidding logic where simultaneous bids can conflict
- Balance updates from multiple sources without atomic guarantees

---

## V33 — Message Ordering Assumptions

**What:** Code assumes messages from different contracts arrive in a specific order, but only messages from the SAME contract to the SAME contract maintain order (via logical time).

**Why it matters:** Incorrect ordering assumptions lead to processing operations out of sequence, potentially bypassing preconditions.

**What to look for:**
- Multi-contract workflows assuming A→C arrives before B→C
- Setup/initialization messages assumed to arrive before operational messages
- Cross-contract state dependencies without explicit sequencing

---

## V34 — Partial Execution in Multi-Message Operations

**What:** A multi-step operation sends several messages, but some succeed while others fail (bounce, out of gas), leaving the system in an inconsistent state.

**Why it matters:** Each internal message creates a separate transaction. If message 1 succeeds (debits sender) but message 2 fails (credit to receiver bounces), funds are lost unless bounce handling restores state.

**What to look for:**
- Operations that send 2+ messages where all must succeed for correctness
- Missing bounce handlers for any sent message in a chain
- State committed after sending first message but before all messages complete
- No rollback mechanism for partial failures

---

## V35 — Stale State After Callback

**What:** Contract reads state, sends a message, and in the callback handler uses the same state without re-reading — but the state may have changed between the send and the callback.

**Why it matters:** Between sending a message and receiving its callback, other messages may modify the contract's state. Using cached/stale state in the callback leads to incorrect calculations.

**What to look for:**
- Callback handlers using values from the original call context
- State not re-read from c4 in bounce/callback handlers
- Assumptions that "nothing changed" between send and callback

---

## V36 — Contract Upgrade Without State Migration

**What:** `set_code()` changes contract code but doesn't migrate storage format, causing the new code to misinterpret existing data.

**Why it matters:** New code with different storage layout reads old data incorrectly. Field positions shift, causing corrupted state, wrong balances, or broken access control.

**What to look for:**
- `set_code()` without corresponding `set_data()` for migration
- No version field in storage to detect format mismatches
- Missing migration function in the new code
- Storage layout changes between versions without a migration path

---

## V37 — Unsafe Code Upgrade Mechanism

**What:** Contract upgrade (`set_code` + `set_data`) lacks proper authorization or validation of the new code.

**Why it matters:** Unauthorized code upgrades allow attackers to replace contract logic entirely, stealing all funds.

**What to look for:**
- `set_code()` reachable without admin/owner authorization
- No governance or multi-sig requirement for upgrades
- Missing timelock on upgrades (allowing instant malicious upgrades)
- New code cell not validated before installation

---

## V38 — `set_code` Takes Effect Timing

**What:** Developer assumes `set_code()` takes effect immediately within the current transaction, but it only takes effect for SUBSEQUENT transactions.

**Why it matters:** The current transaction continues executing the OLD code after `set_code()`. If the developer expects new logic to run immediately, the contract behaves unexpectedly.

**What to look for:**
- Logic after `set_code()` that assumes new code is active
- State changes after `set_code()` that conflict with new code's expectations
- Initialization logic placed in the same transaction as `set_code()`

---

## V39 — Contract Deletion / Self-Destruct via Mode +32

**What:** `send_raw_message` with mode flag `+32` causes the contract to be destroyed if its balance reaches zero after the send.

**Why it matters:** Accidental or malicious use of mode +32 can permanently destroy a contract, losing all state and making it permanently inaccessible.

**What to look for:**
- `send_raw_message(msg, 160)` or any mode including +32
- Mode +32 used in error paths or refund logic
- User-influenced control flow that could reach a +32 send

---

## V40 — Contract Balance Used for Logic Decisions

**What:** Contract uses `my_balance` (balance before message processing) or `get_balance()` for business logic decisions.

**Why it matters:** Anyone can send TON to any contract, manipulating its balance. Using balance for access control, state decisions, or accounting is exploitable.

**What to look for:**
- `my_balance` used in conditional logic
- Balance comparisons for authorization or feature gates
- Accounting that relies on contract balance instead of internal tracking
- Pool/vault logic using raw balance instead of tracked deposits

---

## V41 — Gas Exhaustion in Loops

**What:** Contract iterates over an unbounded data structure (dictionary, list) in a single transaction, exhausting the gas limit.

**Why it matters:** If a dictionary grows large enough, iterating it in one transaction exceeds gas limits. The transaction fails, but state changes from the last successful transaction remain, potentially leaving inconsistent state.

**What to look for:**
- `while` or `do ... until` loops over dictionaries without iteration limits
- Batch operations (send to all holders, update all entries) without pagination
- User-controllable data structures that can grow unbounded
- Missing `MAX_ITERATIONS` constant for loop bounds

**Secure pattern:** Process at most N entries per transaction, use a continuation marker, resume in next transaction via internal message.

---

## V42 — Stack Depth Overflow

**What:** Deeply nested function calls or recursive operations exhaust TVM's stack.

**Why it matters:** TVM has a stack limit. Deep recursion or excessive nesting causes out-of-stack errors, failing the transaction.

**What to look for:**
- Recursive functions without depth limits
- Deeply nested cell unpacking (cells within cells within cells)
- Recursive dictionary traversal without bounds

---

## V43 — Storage Phase Failure

**What:** Contract doesn't maintain enough balance to pay storage fees, causing the storage phase to fail and the contract to be frozen.

**Why it matters:** Frozen contracts are inaccessible — no messages can be processed. This is effectively a permanent DoS if no one sends TON to unfreeze it.

**What to look for:**
- Operations that can reduce balance to near-zero
- No `raw_reserve()` to protect minimum storage balance
- Large dynamic storage without proportional balance requirements
- Long-lived contracts without storage fee monitoring

---

## V44 — Account State Size Limits

**What:** Contract state (c4 register / persistent data) grows beyond TVM limits or becomes so large that storage fees become prohibitive.

**Why it matters:** Oversized state increases storage fees. Dynamic growth from user interactions can make the contract economically unviable.

**What to look for:**
- Dictionaries growing with each user interaction
- No state cleanup or archival mechanism
- User data stored directly in the contract instead of in child contracts
- Missing tokenization pattern (splitting state into per-user contracts)

---

## V45 — Uninit Account Message Handling

**What:** Contract sends messages to addresses that may not have deployed contracts, without considering the destination state.

**Why it matters:** Messages to uninitialized accounts with bounceable flag will bounce. Messages to uninitialized accounts with non-bounceable flag deliver TON but no contract processes the message.

**What to look for:**
- Sending operational messages to user addresses (might not have wallets deployed)
- Missing StateInit in messages that should deploy contracts on first interaction
- Assuming destination contract exists without verification

---

## V46 — Missing StateInit in Deploy Messages

**What:** Contract attempts to deploy a child contract (e.g., Jetton wallet) but doesn't include StateInit in the message, or includes wrong init data.

**Why it matters:** Without correct StateInit, the child contract isn't deployed, messages bounce, and the system fails silently or loses funds.

**What to look for:**
- Deploy messages missing `store_uint(1, 1)` + StateInit cell
- StateInit with wrong code or data cell
- Init data that doesn't match the child contract's expected format
- Address computation (`cell_hash(StateInit)`) not matching the expected address

---

## V47 — Address Computation Mismatch

**What:** The address computed for a child contract doesn't match what the child contract actually produces from its StateInit, causing messages to go to the wrong address.

**Why it matters:** If the parent computes address X for the Jetton wallet but the actual wallet deploys at address Y, messages are sent to an empty account or wrong contract.

**What to look for:**
- `calculate_address()` using different code or data than the actual StateInit
- Hash computation not using the standard `cell_hash` of the StateInit cell
- Workchain ID mismatch between computed and actual address

---

## V48 — Replay Attack on Wallet Operations

**What:** Wallet or multi-sig contract lacks per-message uniqueness, allowing the same signed operation to be submitted multiple times.

**Why it matters:** Without replay protection, a valid withdrawal transaction can be replayed to drain the wallet. This applies to both external messages (seqno) and internal messages (custom nonce).

**What to look for:**
- Missing sequence number increment on external message processing
- Internal message operations without idempotency or nonce
- Multi-sig approval that can be replayed after execution

---

## V49 — Split/Merge Transaction Assumptions

**What:** Contract logic assumes a single transaction execution model, not accounting for TON's potential split/merge of large transactions.

**Why it matters:** In high-load scenarios, the TON blockchain can split transactions. Contracts that assume atomic execution across multiple accounts may face split-execution issues.

**What to look for:**
- Complex multi-account operations assuming atomicity
- Time-sensitive operations that don't account for processing delays

---

## V50 — Logical Time Ordering Bypass

**What:** Contract relies on logical time (lt) for ordering but doesn't account for cases where lt-based ordering doesn't apply (messages from different contracts).

**Why it matters:** Logical time guarantees ordering only for messages between the same pair of contracts. Cross-contract message ordering requires explicit sequencing.

**What to look for:**
- Logic depending on message arrival order from multiple sources
- Auction/bid logic without explicit round/phase management
- State machines relying on implicit message ordering

---

## V51 — Missing Minimum Balance for Contract Operations

**What:** Contract doesn't enforce a minimum balance threshold, allowing operations when balance is too low to complete message chains.

**Why it matters:** Operations that send messages require sufficient balance for gas. If balance is too low, messages aren't sent or sent with insufficient gas, causing downstream failures.

**What to look for:**
- No balance check before sending messages
- Operations that assume sufficient gas without checking
- Missing `throw_unless(error::insufficient_balance, my_balance > min_required)`

---

## V52 — Dangling References After State Update

**What:** State update invalidates references (cell refs, dict entries) held in local variables, but code continues using the stale references.

**Why it matters:** After `set_data()`, the old state slice's references point to outdated data. Continuing to read from them produces stale or incorrect values.

**What to look for:**
- `set_data()` followed by reading from a slice obtained before the set
- Loading state, modifying part, saving, then reading unchanged parts from old slice
- Not re-loading state after `set_data()`

---

## V53 — Incorrect Method ID Collision

**What:** Two `get` methods or functions have colliding method IDs (CRC16 of function name), causing one to shadow the other.

**Why it matters:** Get method calls use numeric method IDs. If two methods hash to the same ID, one is unreachable, potentially hiding important state queries.

**What to look for:**
- Multiple `method_id` functions with similar names
- Custom `method_id(N)` overlapping with auto-assigned IDs
- Get methods that seem unreachable from off-chain queries

---

## V54 — Global Variable Initialization Order

**What:** FunC global variables are loaded lazily from storage, and the loading order matters. Missing `load_data()` at the start of handlers leads to uninitialized globals.

**Why it matters:** If globals are used before `load_data()` is called, they contain default/zero values, bypassing stored configuration like admin addresses.

**What to look for:**
- Handler functions that use global variables without calling `load_data()` first
- `load_data()` called conditionally (not on all paths)
- Globals assumed to be automatically initialized from storage

---

## V55 — Tact Contract Ownership Not Enforced

**What:** Tact contracts using the `Ownable` trait don't call `self.requireOwner()` on all admin operations.

**Why it matters:** Tact's ownership is opt-in per function. Missing `self.requireOwner()` on any admin function opens it to unauthorized access.

**What to look for:**
- Tact receive handlers for admin operations without `self.requireOwner()`
- Custom admin checks that don't use the standard Ownable pattern
- Mixed authorization patterns (some functions use requireOwner, others don't)

---

## V56 — Tact `receive()` Fallback Misuse

**What:** Tact's empty `receive()` handler (fallback for plain TON transfers) either does too much or too little.

**Why it matters:** The fallback handler processes plain TON transfers. If it contains business logic, any TON transfer triggers it. If it's missing, plain transfers are rejected.

**What to look for:**
- `receive()` with complex logic that shouldn't run on plain transfers
- Missing `receive()` causing TON deposits to fail
- Fallback handler that modifies important state

---

## V57 — Tact Struct/Message Encoding Compatibility

**What:** Tact structs and messages serialized to cells don't match the format expected by FunC contracts they interact with.

**Why it matters:** Cross-language interoperability requires matching serialization. Tact uses its own encoding conventions that may differ from hand-crafted FunC message layouts.

**What to look for:**
- Tact contracts sending messages to FunC contracts (or vice versa)
- Custom op codes that don't match between implementations
- Bit-width mismatches in field encoding

---

## V58 — Tact Map Iteration Limitations

**What:** Tact `map<K, V>` types have limitations on iteration and size that developers may not account for.

**Why it matters:** Tact maps are backed by TON dictionaries. Large maps cause gas issues during iteration, and there's no built-in size tracking.

**What to look for:**
- Tact maps used for unbounded collections
- Iteration over maps without gas/size limits
- Missing separate counter for map size

---

## V59 — Incorrect Use of `throw_unless` vs `throw_if`

**What:** Condition polarity is reversed — `throw_unless` is used where `throw_if` is needed, or vice versa, causing the check to pass when it should fail.

**Why it matters:** `throw_unless(code, condition)` throws when condition is FALSE. `throw_if(code, condition)` throws when condition is TRUE. Mixing them up inverts the security check.

**What to look for:**
- `throw_unless` with a condition that should cause rejection when true
- `throw_if` with a condition that should be required
- Complex boolean expressions in throw conditions (easy to get polarity wrong)

---

## V60 — Missing Gas Reservation Before Message Send

**What:** Contract sends messages without first reserving gas for its own storage and future operations.

**Why it matters:** Sending all available value in messages leaves the contract with zero balance, causing it to be frozen by storage fees.

**What to look for:**
- `send_raw_message` without prior `raw_reserve()`
- Mode 64/128 sends without balance protection
- Multiple message sends that cumulatively drain the contract
- Missing `raw_reserve(ton_value, RESERVE_REGULAR)` pattern

**Secure pattern:** Always `raw_reserve(MIN_STORAGE_FEE, RESERVE_REGULAR)` before sending messages, then use mode 64 or 128 on the last message.
