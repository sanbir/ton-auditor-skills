# Attack Vectors Reference (4/4) — Oracle, DeFi, Jetton/NFT & Platform-Level

## V91 — Stale Oracle Price

**What:** Contract uses a price feed without checking the timestamp of the last update.

**Why it matters:** Stale prices may not reflect current market conditions. Attackers wait for the price to diverge significantly from the stale feed, then exploit the discrepancy for profitable trades or liquidation avoidance.

**What to look for:**
- Price data consumed without freshness check (`now - last_update > MAX_STALENESS`)
- Oracle update messages not carrying timestamps
- No maximum acceptable age for price data

---

## V92 — Oracle Confidence Interval Ignored

**What:** Price oracle provides a confidence band (e.g., Pyth) but the contract only uses the midpoint price.

**Why it matters:** During volatile periods, the confidence interval widens. Using the midpoint without checking confidence can accept unreliable prices.

**What to look for:**
- Price used without checking associated confidence/deviation
- Missing `throw_unless(confidence < MAX_CONFIDENCE)` check
- Calculations that should use worst-case price within confidence band

---

## V93 — Fake Oracle Contract

**What:** Contract doesn't validate that the oracle address is the legitimate oracle, accepting price updates from any contract.

**Why it matters:** Attacker deploys a contract that sends fake price updates, manipulating the protocol to accept arbitrary prices.

**What to look for:**
- Price update handler without sender address validation
- Oracle address not stored or not checked on updates
- Missing `throw_unless(error::wrong_oracle, equal_slices(sender, oracle_address))`

---

## V94 — Single Oracle Dependency

**What:** Protocol relies on a single oracle source with no fallback or circuit breaker.

**Why it matters:** If the single oracle goes down, becomes stale, or is manipulated, the entire protocol is affected with no alternative price source.

**What to look for:**
- Only one oracle address stored
- No fallback oracle mechanism
- No circuit breaker for extreme price movements
- No manual override for emergency situations

---

## V95 — Flash Loan Price Manipulation

**What:** Attacker uses a flash loan (or large temporary position) to manipulate spot prices, then exploits protocols that use those prices.

**Why it matters:** On-chain spot prices are manipulable within a single transaction or block. Protocols using AMM spot prices for oracle-like purposes are vulnerable.

**What to look for:**
- Price derived from pool reserves (spot price) instead of TWAP or external oracle
- Lending protocols using DEX spot prices for collateral valuation
- Vault share prices derivable from manipulable on-chain state
- Missing TWAP or time-weighted averaging

---

## V96 — Vault Share Inflation Attack

**What:** Attacker inflates the value of vault shares by donating tokens directly to the vault contract, causing subsequent depositors to receive 0 shares.

**Why it matters:** Same as V65 (first-depositor attack), but applied specifically to vault/yield protocols. Combined with flash loans, this can be executed atomically.

**What to look for:**
- Share price calculated from vault's raw token balance
- No virtual shares mechanism
- First deposit has no minimum amount
- Direct token transfer to vault affects share price

---

## V97 — Staking Reward Index Manipulation

**What:** Reward-per-token accumulator can be manipulated by flashstaking or by manipulating the total staked amount.

**Why it matters:** If an attacker can temporarily inflate their stake just before a reward distribution and withdraw immediately after, they capture disproportionate rewards.

**What to look for:**
- No minimum staking duration
- Rewards distributed based on point-in-time balance (not time-weighted)
- `rewardPerToken` updated without checking for flash deposit/withdraw
- Missing cooldown period for unstaking

---

## V98 — Flash Stake/Unstake Reward Capture

**What:** User stakes just before reward distribution, claims rewards, and unstakes immediately — capturing rewards without genuine long-term staking.

**Why it matters:** Flash stakers dilute rewards for genuine long-term stakers. The economic incentive for actually staking is undermined.

**What to look for:**
- No minimum staking duration or warmup period
- Rewards claimable immediately after staking
- No time-weighted balance for reward calculation
- Missing cooldown on unstaking

---

## V99 — Reward Dilution via Direct Transfer

**What:** Attacker transfers tokens directly to the reward pool contract, inflating the reward rate denominator or confusing the reward calculation.

**Why it matters:** If reward rate is calculated from the contract's token balance, direct transfers manipulate the rate. This can either dilute rewards or create phantom rewards.

**What to look for:**
- Reward rate derived from raw balance instead of tracked deposits
- No distinction between tracked rewards and unexpected transfers
- Missing internal accounting for reward token balance

---

## V100 — Liquidation Incentive Insufficient

**What:** Liquidation bonus doesn't cover the gas cost of executing the liquidation transaction.

**Why it matters:** If liquidation is unprofitable for liquidators, no one liquidates underwater positions, leading to bad debt accumulation.

**What to look for:**
- Liquidation bonus calculated as percentage but minimum positions are too small
- Gas costs on TON (storage fees + message fees) not considered in incentive design
- Minimum liquidatable amount not enforced

---

## V101 — Self-Liquidation Profit

**What:** User can liquidate their own position for a net profit due to the liquidation bonus exceeding the cost.

**Why it matters:** User takes a loan, then liquidates themselves, keeping the liquidation bonus as profit. This drains the protocol.

**What to look for:**
- No check that liquidator != borrower
- Liquidation bonus > (debt - collateral value) allowing profit
- Self-liquidation in a single message chain

---

## V102 — Interest Accrual During Pause

**What:** Protocol pause mechanism stops new operations but doesn't stop interest accrual, leading to unexpected liquidations when unpaused.

**Why it matters:** While paused, interest keeps accumulating. Positions that were healthy become underwater. On unpause, mass liquidations occur, potentially cascading.

**What to look for:**
- Pause mechanism that only blocks entry points but not interest calculation
- No interest freeze during pause
- No grace period after unpause for users to adjust positions

---

## V103 — Bad Debt Not Socialized

**What:** When a position is liquidated but debt exceeds collateral value, the bad debt has no clear owner or distribution mechanism.

**Why it matters:** Unsocialized bad debt creates a hidden protocol deficit. Eventually, withdrawals exceed available funds, causing a bank run.

**What to look for:**
- Liquidation that can result in debt > collateral with no handling
- No insurance fund or bad debt socialization mechanism
- Protocol balance sheet that doesn't track bad debt

---

## V104 — DEX Slippage Not From User Input

**What:** Swap slippage tolerance is calculated on-chain from pool state instead of being specified by the user.

**Why it matters:** If minimum output is calculated from current (manipulable) pool state, a sandwich attacker can manipulate the pool, set a favorable slippage tolerance, then extract value.

**What to look for:**
- `min_amount_out` calculated from pool reserves instead of user-supplied
- Missing user-specified slippage parameter in swap messages
- Hardcoded slippage tolerance

---

## V105 — Missing Swap Deadline

**What:** Swap or trade operations have no expiry timestamp, allowing them to be held and executed at a disadvantageous time.

**Why it matters:** A pending swap message can be delayed by validators and executed when the price has moved significantly against the user.

**What to look for:**
- Swap operations without `deadline` or `valid_until` parameter
- No `throw_unless(error::expired, now <= deadline)` in swap handlers
- Time-sensitive operations without expiry

---

## V106 — AMM Constant Product Invariant Violation

**What:** Swap or liquidity operation doesn't verify the AMM invariant (x*y=k or equivalent) after the operation.

**Why it matters:** If the invariant isn't checked, tokens can be extracted without providing equivalent value, draining the pool.

**What to look for:**
- Post-swap invariant check missing
- LP add/remove not verifying proportional contribution
- Rounding errors that systematically violate the invariant

---

## V107 — Bridge Message Replay

**What:** Cross-chain bridge doesn't deduplicate messages, allowing the same bridge transfer to be executed multiple times on the destination chain.

**Why it matters:** Replaying bridge messages mints tokens multiple times for a single lock on the source chain, breaking supply invariants.

**What to look for:**
- No nonce or message hash tracking for processed bridge messages
- Missing `dict_set` / `dict_get` for processed message IDs
- No replay protection on bridge claim operations

---

## V108 — Bridge Supply Invariant Violation

**What:** Minted tokens on destination chain don't match locked tokens on source chain.

**Why it matters:** If mint > lock, the bridge creates unbacked tokens. If lock > mint, user funds are stuck. Both break the bridge's fundamental guarantee.

**What to look for:**
- No supply cap matching locked amount on source
- Rate limits that can be bypassed
- Decimal conversion errors between chains
- Minting without corresponding lock verification

---

## V109 — Governance Flash-Vote

**What:** User acquires governance tokens, votes, then immediately transfers/sells tokens — getting disproportionate voting power.

**Why it matters:** Flash voting allows buying temporary voting power to pass self-serving proposals without long-term commitment.

**What to look for:**
- Vote weight based on current balance (not historical snapshot)
- No vote locking period
- Token transfers allowed during active voting periods
- Missing `vote_weight_at(slot/time)` snapshot mechanism

---

## V110 — Governance Proposal Execution Without Timelock

**What:** Passed governance proposals execute immediately without a delay for users to review and potentially exit.

**Why it matters:** Malicious proposals pass and execute before affected users can take protective action (withdraw funds, revoke permissions).

**What to look for:**
- No delay between proposal passage and execution
- Missing timelock parameter
- No cancel mechanism during timelock period

---

## V111 — NFT Transfer Policy Bypass

**What:** NFT collection enforces transfer rules (royalties, allowlists) but they can be bypassed through alternate transfer mechanisms.

**Why it matters:** Creators lose royalty revenue; banned addresses receive NFTs; marketplace restrictions are circumvented.

**What to look for:**
- Transfer validation only in one handler but not in `transfer` variants
- Direct ownership change without going through the transfer policy
- Missing royalty enforcement in all transfer paths

---

## V112 — NFT Ownership Spoofing

**What:** NFT item's owner field can be modified by unauthorized parties.

**Why it matters:** Changing ownership without consent steals the NFT from the legitimate owner.

**What to look for:**
- Owner update handler without checking current owner or collection authority
- Missing `throw_unless(error::not_owner, equal_slices(sender, owner_address))`
- Ownership change through non-standard messages

---

## V113 — Unaudited Dependency Risk

**What:** Contract depends on external contracts (libraries, oracles, routers) that haven't been audited or can be upgraded by third parties.

**Why it matters:** A vulnerability or malicious upgrade in a dependency affects all contracts that depend on it. The contract's security is only as strong as its weakest dependency.

**What to look for:**
- Hardcoded addresses of external contracts without considering their upgrade risk
- No validation of dependency contract code hash
- Trusting return values from unaudited external contracts

---

## V114 — Masterchain/Basechain Interaction Risks

**What:** Contract in basechain (workchain 0) interacts with masterchain (workchain -1) contracts, or vice versa, without accounting for cross-workchain semantics.

**Why it matters:** Cross-workchain messages have different costs, timing, and guarantees. Incorrect handling can lead to stuck messages or unexpected gas costs.

**What to look for:**
- Messages sent across workchains without adjusted gas values
- Address validation not checking workchain compatibility
- Assumptions about message delivery timing that differ across workchains

---

## V115 — Jetton Wallet Code Verification Missing

**What:** Contract interacting with Jetton wallets doesn't verify the wallet's code matches the expected Jetton wallet code.

**Why it matters:** An attacker can deploy a contract at the expected Jetton wallet address (if they can predict/manipulate the address) with malicious code that always reports large balances.

**What to look for:**
- Jetton wallet interaction without verifying wallet code hash
- No `calculate_user_jetton_wallet_address` to derive expected address
- Trusting arbitrary addresses claimed to be Jetton wallets

---

## V116 — Multi-Sig Approval Counting Errors

**What:** Multi-signature contract has bugs in counting approvals: double-counting, not tracking who approved, or incorrect threshold checking.

**Why it matters:** If approvals can be double-counted, a single signer can approve multiple times to reach threshold. If tracking is wrong, the threshold can be bypassed.

**What to look for:**
- Approval tracking using a counter instead of per-signer flags
- Missing check that signer hasn't already approved
- Threshold check using `>=` vs `>` incorrectly
- Approval state not cleared after execution

---

## V117 — Proxy/Router Pattern Without Proper Forwarding

**What:** A proxy or router contract forwards messages to implementation contracts but doesn't properly forward all message fields (value, sender, bounce flags).

**Why it matters:** Lost context in forwarding breaks sender validation, gas propagation, or bounce handling in the destination contract.

**What to look for:**
- Router that doesn't forward original sender address
- Value lost in routing (not forwarded to implementation)
- Bounce flags not preserved through routing

---

## V118 — Tact `init` Function Re-entrancy

**What:** Tact contract's `init` function can be called multiple times through deployment replay or specific message sequences.

**Why it matters:** Re-initialization overwrites all state, potentially changing the owner and resetting balances.

**What to look for:**
- No `is_initialized` check beyond initial deployment
- Init state accessible through message handlers
- Contract that can receive `StateInit` messages post-deployment

---

## V119 — Tact `require` vs FunC `throw_unless` Semantics

**What:** Tact's `require(condition, message)` and FunC's `throw_unless(code, condition)` have the same polarity, but Tact's error messages don't provide numeric codes for programmatic handling.

**Why it matters:** If a Tact contract interfaces with FunC contracts that check specific error codes, Tact's string-based errors may not be handled correctly.

**What to look for:**
- Cross-language contract interactions relying on error codes
- Tact contracts without numeric error identifiers for external consumers
- Error handling that depends on specific throw codes

---

## V120 — Undocumented TVM Opcodes / Behavior

**What:** Contract uses raw TVM assembly (asm functions) or undocumented opcodes that may have unexpected behavior or change between TVM versions.

**Why it matters:** Undocumented behavior can change without warning, breaking contracts. Assembly code is harder to audit and may have subtle bugs.

**What to look for:**
- `asm` functions with raw TVM opcodes
- Reliance on specific TVM version behavior
- Gas cost assumptions for specific opcodes that may change
- Missing comments explaining assembly code purpose
