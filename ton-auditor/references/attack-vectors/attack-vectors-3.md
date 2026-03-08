# Attack Vectors Reference (3/4) — Arithmetic, Token Operations & State Management

## V61 — Integer Overflow / Underflow

**What:** Arithmetic operations on 257-bit TVM integers overflow or underflow without checks.

**Why it matters:** TVM integers are 257-bit signed. While overflow throws by default in TVM (unlike EVM pre-0.8), intermediate calculations or explicit bit-width operations (`load_uint`, `store_uint`) can silently truncate.

**What to look for:**
- `store_uint` or `store_int` with a width smaller than the computed value's range
- Multiplication before bounds checking
- Accumulator patterns where sums grow without bounds
- `load_uint(N)` → arithmetic → `store_uint(N)` where arithmetic can exceed N bits

---

## V62 — Division Before Multiplication (Precision Loss)

**What:** Division performed before multiplication in a formula, losing precision in integer arithmetic.

**Why it matters:** Integer division truncates. `(x / z) * y` loses precision compared to `(x * y) / z`. In financial calculations, this compounds into significant value loss.

**What to look for:**
- Fee calculations: `fee = amount / 100 * fee_rate` instead of `fee = amount * fee_rate / 100`
- Share calculations: `shares = deposit / total * supply` instead of `shares = deposit * supply / total`
- Any formula where division precedes multiplication on the same operands

---

## V63 — Division by Zero

**What:** Divisor can be zero in user-influenced calculations, causing a TVM exception.

**Why it matters:** Division by zero throws exit code 4, reverting the transaction. If this occurs in critical paths (withdrawals, claims), it creates a denial of service.

**What to look for:**
- Division where the divisor comes from storage or message (could be zero)
- Pool/vault calculations where `total_supply` can be zero
- Rate calculations with potentially zero denominators
- Missing `throw_unless(error::zero_divisor, divisor > 0)` checks

---

## V64 — Rounding Direction Exploitation

**What:** Rounding always favors one direction (e.g., always truncating), allowing systematic value extraction.

**Why it matters:** In deposit/withdraw protocols, if both operations round in the user's favor, an attacker can extract value through repeated round-trips.

**What to look for:**
- Deposits that round shares UP (user gets more shares than they should)
- Withdrawals that round tokens UP (user gets more tokens than their shares warrant)
- Fee calculations that round DOWN (lower fees than intended)
- Missing "round in protocol's favor" policy

**Secure pattern:** Deposits: round shares DOWN. Withdrawals: round tokens DOWN (or shares UP). Fees: round UP.

---

## V65 — First-Depositor Vault Inflation Attack

**What:** First depositor to an empty vault deposits 1 unit, then donates a large amount directly, inflating the share price so subsequent depositors receive 0 shares.

**Why it matters:** Attacker deposits 1 wei equivalent → donates 1000 TON directly → share price = 1000 TON per share → victim deposits 999 TON → gets 0 shares (rounds down) → attacker withdraws everything.

**What to look for:**
- Vault/pool with share-based accounting and no minimum first deposit
- No virtual shares or dead shares mechanism
- Share price calculable from raw token balance (manipulable via direct transfer)
- Missing minimum deposit amount

**Secure pattern:** Use virtual shares (add virtual offset to both numerator and denominator), enforce minimum first deposit, or use dead shares.

---

## V66 — Coin/Token Amount Truncation

**What:** Token amounts are truncated when converting between different precisions or when packing into smaller bit widths.

**Why it matters:** Storing a 120-bit amount in a 64-bit field silently truncates, losing value. Converting between different decimal bases without proper handling loses precision.

**What to look for:**
- `store_coins()` vs `store_uint(amount, 64)` — `coins` uses variable-length, uint64 truncates
- Cross-token operations with different decimal precisions
- Amount conversions between nanoTON (9 decimals) and other token units

---

## V67 — Fee Bypass on Alternate Paths

**What:** Multiple code paths achieve the same outcome but not all apply fees consistently.

**Why it matters:** Users route through the fee-free path, depriving the protocol of revenue or breaking economic assumptions.

**What to look for:**
- Transfer vs. transfer_notification paths with different fee logic
- Direct mint/burn vs. swap paths
- Admin operations that bypass fees (which users might access)
- Emergency withdraw paths without fees

---

## V68 — Fee-on-Transfer Token Handling

**What:** Contract assumes received amount equals sent amount when interacting with tokens that have transfer fees.

**Why it matters:** If a token takes a 1% fee on transfer, sending 100 tokens delivers 99. If the contract credits the sender with 100, accounting becomes inconsistent.

**What to look for:**
- `transfer_notification` crediting the exact amount from the notification body
- No post-transfer balance verification
- Vault accounting based on expected amounts rather than actual received

---

## V69 — Dust Amount Locking

**What:** Very small amounts (dust) remain locked in the contract because they're too small to transfer (below minimum message value for gas).

**Why it matters:** Accumulated dust across many users can represent significant total value, permanently locked.

**What to look for:**
- Withdraw operations that fail for amounts below minimum gas cost
- No dust collection or sweep mechanism
- Remainder calculations that produce untransferable amounts

---

## V70 — Supply Invariant Violation

**What:** Total supply tracking variable doesn't match the actual sum of all individual balances.

**Why it matters:** Mismatched supply allows inflation (minting from nothing) or deflation (destroying value). Critical for Jetton minters.

**What to look for:**
- `total_supply` not updated on every mint and burn
- Burn notification handler not decrementing `total_supply`
- Paths where balance changes without supply adjustment
- Rounding differences between individual and total calculations

---

## V71 — Double-Spend via Race Condition

**What:** User initiates two withdrawals/transfers simultaneously, and both process before either's state update is visible to the other.

**Why it matters:** In TON's async model, two messages processed in the same block may both read the same balance and both succeed, spending funds twice.

**What to look for:**
- Withdrawal operations that don't use a locking/pending mechanism
- Balance checks that can be satisfied by two concurrent messages
- Missing "processing" state for in-flight operations

---

## V72 — Jetton Wallet Balance Manipulation

**What:** Jetton wallet's internal balance can be manipulated through crafted messages, causing it to report or send more tokens than it holds.

**Why it matters:** If the wallet's balance tracking can be desynchronized from reality, users can extract more tokens than they deposited.

**What to look for:**
- Jetton wallet balance updates without proper sender validation
- `internal_transfer` handler that doesn't verify sender is the minter
- Direct balance modification through non-standard messages

---

## V73 — NFT Collection Index Manipulation

**What:** NFT collection's item index can be manipulated to overwrite existing items or skip indices.

**Why it matters:** Index manipulation allows minting duplicate NFTs, overwriting ownership of existing items, or creating gaps that break enumeration.

**What to look for:**
- `next_item_index` not atomically incremented during mint
- No check that the index is exactly `next_item_index` (allowing arbitrary indices)
- Missing bounds validation on supplied indices

---

## V74 — Jetton Minter Admin Abuse

**What:** Jetton minter admin can mint unlimited tokens, change metadata, or modify critical parameters without governance.

**Why it matters:** Centralized minting authority can inflate supply, devaluing all holders' tokens.

**What to look for:**
- Unlimited minting capability for admin
- No minting cap or rate limit
- Admin can change token metadata to deceive users
- No governance or multi-sig for minting operations

---

## V75 — Incorrect `store_coins` / `load_coins` Usage

**What:** Misusing `coins` serialization (variable-length encoding) or confusing it with fixed-width integer storage.

**Why it matters:** `store_coins` uses a variable-length encoding (4-bit length prefix + value). Mixing `store_coins` / `load_coins` with `store_uint` / `load_uint` corrupts the data format.

**What to look for:**
- `store_coins` paired with `load_uint` (or vice versa)
- Amount fields stored with inconsistent encoding
- Custom serialization that doesn't match standard Jetton message format

---

## V76 — Cross-Contract State Dependency

**What:** Contract relies on state in another contract remaining consistent between reads, but the other contract can be modified between messages.

**Why it matters:** Between sending a query and receiving a response, the queried contract's state may change due to other messages. Decisions based on stale cross-contract state are unreliable.

**What to look for:**
- Query → process response pattern where response data may be stale
- Caching cross-contract values that become outdated
- Price feeds or balances read cross-contract without freshness checks

---

## V77 — Unbounded Data in Messages

**What:** Contract accepts and processes messages with arbitrary-length data payloads without limiting the processing cost.

**Why it matters:** Large payloads increase gas costs. Attackers can send maximum-size cells to force expensive processing, griefing the contract.

**What to look for:**
- Message handlers that iterate over payload content without bounds
- User-provided data stored directly without size validation
- Forward payloads accepted and processed without length checks

---

## V78 — Dictionary Key Type Mismatch

**What:** Dictionary operations use inconsistent key sizes — e.g., storing with 256-bit keys but looking up with 64-bit keys.

**Why it matters:** Mismatched key sizes cause lookups to fail silently (key not found) or return wrong entries. Critical for user balance lookups.

**What to look for:**
- `udict_set` with one key size, `udict_get` with another
- Address-keyed dictionaries using different address representations
- Missing key size constants (hardcoded different values in different places)

---

## V79 — Coupled State Update Inconsistency

**What:** Two or more state fields that must be updated together are updated in separate operations, allowing observation of inconsistent state.

**Why it matters:** If `total_staked` and `user_stake` must be consistent, updating one before the other allows messages in between to see inconsistent state.

**What to look for:**
- Related state fields updated in different code blocks
- State saves between coupled updates (allowing intermediate observation)
- Multi-field invariants (sum, ratio) maintained across separate updates

---

## V80 — Precision Loss in Accumulator Patterns

**What:** Reward-per-token or similar accumulator loses precision due to integer division, causing small stakers to receive zero rewards.

**Why it matters:** When `reward / total_staked` rounds to zero, rewards are lost. Over time, small stakers receive nothing while their rewards accumulate to no one.

**What to look for:**
- Reward distribution formulas dividing by large denominators
- No precision multiplier (e.g., multiplying by 1e18 before division)
- Accumulator updates where `reward_amount / total_supply == 0`

---

## V81 — Unsafe Bit Shifting

**What:** Bit shift operations that exceed the operand width, producing undefined or zero results.

**Why it matters:** Shifting by more than the bit width can silently produce zero or wrap around, causing incorrect calculations in mathematical operations.

**What to look for:**
- Left/right shift by variable amounts without bounds checking
- Shift used in place of multiplication/division without overflow consideration
- Platform-specific shift behavior assumptions

---

## V82 — Raw Balance vs Tracked Balance Confusion

**What:** Contract uses its actual TON balance (`my_balance` or `get_balance()`) for accounting instead of internally tracked deposits.

**Why it matters:** Anyone can send TON to a contract, inflating `my_balance`. Using raw balance for share calculations or withdrawals allows attackers to manipulate the accounting.

**What to look for:**
- Share price calculated from `my_balance`
- Withdrawal amounts based on contract balance
- Pool TVL determined by raw balance instead of deposit tracking
- Missing internal `total_deposits` counter

---

## V83 — Jetton Burn Without Proper Notification

**What:** Jetton tokens are burned but the minter isn't notified (missing `burn_notification`), desynchronizing `total_supply`.

**Why it matters:** If wallets burn tokens without notifying the minter, `total_supply` remains inflated, breaking supply invariants and economic calculations.

**What to look for:**
- Burn operations in Jetton wallets that skip sending `burn_notification` to minter
- Custom burn mechanisms that bypass standard TEP-74 flow
- Minter's `burn_notification` handler not updating `total_supply`

---

## V84 — Incorrect Coins Encoding for Zero Values

**What:** Zero amounts encoded with `store_coins(0)` take 4 bits, but code may assume zero amounts take zero space or different formatting.

**Why it matters:** Incorrect space assumptions cause off-by-N bit errors in message serialization, corrupting subsequent fields.

**What to look for:**
- Manual message building that doesn't account for `store_coins(0)` bit usage
- Conditional amount fields where zero case serializes differently
- Message format calculations that are off for zero amounts

---

## V85 — Missing Minimum Viable Transaction Amount

**What:** Contract accepts transactions with amounts too small to cover gas costs, wasting gas from the contract's balance.

**Why it matters:** Processing tiny transactions costs more in gas than the transaction value. Attackers can grief the contract by flooding it with dust transactions.

**What to look for:**
- No minimum amount check on deposits or swaps
- Missing `throw_unless(error::amount_too_small, amount >= MIN_AMOUNT)`
- Operations that process regardless of economic viability

---

## V86 — State Rollback on Throw After Send

**What:** Developer expects `throw` to revert previously sent messages, but sent messages are NOT reverted by throws in the current transaction.

**Why it matters:** In TVM, `send_raw_message` queues messages in the action list. A subsequent `throw` reverts state (c4) but the action list might or might not be committed depending on the throw type. `throw` during compute phase reverts both c4 and action list. But messages sent in a previous successful transaction are irreversible.

**What to look for:**
- Code that sends messages then throws, expecting full rollback
- Error handling after message sends that assumes reversibility
- Multi-step operations where later throws should "undo" earlier sends

---

## V87 — Token Decimal Mismatch in Multi-Token Operations

**What:** Operations involving multiple tokens don't account for different decimal precisions.

**Why it matters:** Comparing 1 USDT (6 decimals) with 1 TON (9 decimals) as equal amounts is a 1000x error. Collateral calculations, swaps, and conversions must normalize decimals.

**What to look for:**
- Cross-token calculations without decimal normalization
- Price feeds in different decimal bases
- Collateral ratios mixing tokens of different decimals
- Missing decimal metadata for supported tokens

---

## V88 — Event/Log Manipulation for Off-Chain Trust

**What:** Off-chain systems trust on-chain events (messages, logs) without verifying the contract that emitted them.

**Why it matters:** Any contract can emit messages that look like events from a legitimate contract. Off-chain indexers that don't verify the source contract can be deceived.

**What to look for:**
- Off-chain services processing events without verifying sender contract
- Indexers trusting message content over message source
- Bridge relayers that verify event data but not event origin

---

## V89 — Circular Dependency Between Contracts

**What:** Contract A depends on state from Contract B, which depends on state from Contract A, creating circular update dependencies.

**Why it matters:** Circular dependencies can cause infinite message loops, inconsistent state, or deadlocks where neither contract can proceed without the other updating first.

**What to look for:**
- Two contracts that send state-update messages to each other
- Configuration dependencies that form cycles
- Callback patterns where both contracts wait for the other

---

## V90 — Immutable Data in Code Cell (c3) Instead of Data Cell (c4)

**What:** Configuration values stored in code (c3 register) that should be updatable, or mutable values stored in code that can't be changed.

**Why it matters:** Values in code can only change via `set_code()` (full upgrade). If a parameter should be updatable (admin address, fee rate), storing it in code requires a full upgrade to change.

**What to look for:**
- Constants in code that should be configurable parameters
- Admin addresses hardcoded in FunC source instead of stored in data
- Fee rates or limits in code that need governance ability to change
- Using `set_code` just to update a parameter
