# Raydium AMM Security Audit Report

## Introduction

This document presents the findings of a security audit conducted on the Raydium Automated Market Maker (AMM) program. The audit involved a review of the provided source code (`program/src/*.rs`) and associated documentation (`docs/*.md`) corresponding to commit `jules_wip_4293864860430496111`.

The methodology included:
1.  **Documentation Review:** Understanding the protocol architecture, smart contract components, user flows, and mathematical underpinnings.
2.  **Source Code Analysis:** Detailed review of individual Rust files (`math.rs`, `state.rs`, `instruction.rs`, `entrypoint.rs`, `processor.rs`, `invokers.rs`, `log.rs`).
3.  **Security-Focused Code Review (Initial Broad Phase):** Specific checks for common Solana vulnerabilities, including integer overflows/underflows, precision loss, rounding exploits, division by zero, insecure CPIs, access control issues, and logic flaws across all components.
4.  **Cross-Cutting Analysis (Initial Broad Phase):** Examining interactions between components, dependencies on external programs (OpenBook DEX), potential economic exploits, DoS vectors, and centralization risks.
5.  **Targeted Deep Dive Vulnerability Research (Phase 2):** Focused investigation into complex interactions, mathematical exploit scenarios (inspired by patterns like those seen in Immunefi bug reports), PnL calculations, LP token operations, fee/rounding interactions, quantization effects, and state synchronization with OpenBook DEX.

## Overall Security Posture

The Raydium AMM program demonstrates a mature design with attention to many common security pitfalls in Solana smart contracts and DeFi applications. Key strengths include:

*   **Robust PDA Usage:** Consistent derivation and validation of Program Derived Addresses for owning critical accounts.
*   **Account Validation:** Thorough checks on account signers, writability, ownership, and matching against stored pubkeys in `AmmInfo`.
*   **Use of Large Integer Types:** `U128` and `U256` for intermediate mathematical calculations mitigate simple overflow risks.
*   **Checked Arithmetic:** Frequent use of `checked_*` methods for arithmetic safety.
*   **Defensive Rounding:** Rounding strategies generally favor the pool or existing LPs.
*   **State Machine for Order Management:** The `MonitorStep` logic includes mechanisms to reset or revert to safe states.
*   **PnL Accounting:** Separation of PnL calculation and its quarantine before LP operations enhances fairness.

The primary areas of risk identified, and reinforced during the deep dive analysis, are not typically flaws leading to direct, unauthorized fund extraction from user vaults by external attackers under normal OpenBook DEX operations, but rather:
*   **Economic Risks:** Related to arbitrage, price manipulation on the external OpenBook DEX impacting AMM order fills, and the inherent complexities of managing an AMM curve via CLOB orders.
*   **Centralization Risks:** Significant control vested in the `AmmOwner` for parameter changes.
*   **Liveness/Efficiency Risks:** Potential for compute limit exhaustion in certain scenarios.
*   **External Dependencies:** Reliance on the OpenBook DEX and SPL Token Program.

## Summary of Findings from Initial Broad Audit

| Finding ID | Title                                                                  | Severity         | Category                               |
| :--------- | :--------------------------------------------------------------------- | :--------------- | :------------------------------------- |
| SR-01      | Admin Control: `MaxPriceMultiplier` Validation Logic Issue             | Low              | Logic Error / Admin Function           |
| SR-02      | Admin Control: Potential to Set `AmmOwner` to Default Pubkey           | Low              | Access Control / Admin Function        |
| SR-03      | Logging: Information Disclosure and Compute Overhead                   | Low-Medium       | Info Disclosure / Performance          |
| SR-04      | Compute Limits: Potential DoS from OpenBook Event Queue / Order CPIs   | Medium           | Denial of Service / Liveness           |
| SR-05      | Economic Risk: OpenBook Price Manipulation Impacting AMM LPs         | Medium           | Economic Exploit / Market Manipulation |
| SR-06      | Data Staleness: OpenBook Event Queue Lag Impacting PnL/Order Optimality | Low              | Data Integrity / Economic              |
| SR-07      | Admin Controls & Centralization Risks (Overall)                        | High             | Centralization / Access Control        |
| SR-08      | Math: Minor Truncation Risk with `.as_u64()` in Invariant Calcs        | Very Low         | Precision / Implementation Detail      |
| SR-09      | Documentation: `CheckedCeilDiv` Comment vs. Implementation Clarity     | Very Low         | Documentation                          |

## Detailed Findings from Initial Broad Audit

---

**Finding ID:** SR-01
*   **Title/Summary:** Admin Control: `MaxPriceMultiplier` Validation Logic Issue
*   **Description:** In `process_set_params`, when setting `AmmParams::MaxPriceMultiplier`, the validation check `if value > amm.max_price_multiplier` is used. This was likely intended to be `if value > amm.min_price_multiplier` to ensure the new maximum is greater than the existing minimum. The current logic does not prevent setting `max_price_multiplier` to a value less than `min_price_multiplier` if the new value is also less than the old `max_price_multiplier`.
*   **Affected Components:** `program/src/processor.rs` (within `process_set_params`)
*   **Potential Impact & Severity:** Low (Admin Error). Could lead to an inconsistent state where `max_price_multiplier < min_price_multiplier`. This would likely cause the `do_idle_state` logic (which checks if the current pool price is within these bounds) to always determine the price is out of bounds, potentially leading to continuous cancellation of orders or preventing new orders if any exist.
*   **Root Cause:** Incorrect comparison logic in the validation of a new `MaxPriceMultiplier`.
*   **Conceptual Scenario:**
    1. Admin sets `min_price_multiplier` to 500.
    2. Current `max_price_multiplier` is 1000.
    3. Admin attempts to set `max_price_multiplier` to 400. The check `if 400 > 1000` is false. The parameter is set.
    4. State becomes `min_price_multiplier = 500`, `max_price_multiplier = 400`.
*   **Recommendations/Mitigations:**
    *   When setting `MaxPriceMultiplier`, validate `if new_max_multiplier > amm.min_price_multiplier && new_max_multiplier > 0`.
    *   When setting `MinPriceMultiplier`, validate `if new_min_multiplier < amm.max_price_multiplier && new_min_multiplier > 0`.

---

**Finding ID:** SR-02
*   **Title/Summary:** Admin Control: Potential to Set `AmmOwner` to Default Pubkey
*   **Description:** The `process_set_params` instruction, when setting `AmmParams::AmmOwner`, does not check if the `new_pubkey` is `Pubkey::default()`.
*   **Affected Components:** `program/src/processor.rs` (within `process_set_params`)
*   **Potential Impact & Severity:** Low (Admin Error/Griefing). If the AMM owner is set to the default pubkey (`11111111111111111111111111111111`), administrative functions for that specific AMM pool (via `SetParams`) would become permanently inaccessible, as no one possesses the private key for the default pubkey.
*   **Root Cause:** Missing validation against `Pubkey::default()` for the new owner address.
*   **Recommendations/Mitigations:** Add a check in `process_set_params` for the `AmmParams::AmmOwner` case: `if new_pubkey == Pubkey::default() { return Err(AmmError::InvalidInput.into()); }`.

---

**Finding ID:** SR-03
*   **Title/Summary:** Logging: Information Disclosure and Compute Overhead
*   **Description:** The `encode_ray_log` function in `log.rs` serializes detailed transaction data (reserves, PnL baselines, amounts, some user balances) into base64 strings and logs them via `msg!`. This logging appears unconditional.
*   **Affected Components:** `program/src/log.rs`, all instructions calling `encode_ray_log`.
*   **Potential Impact & Severity:**
    *   **Information Disclosure (Low):** While blockchain data is public, structured logs significantly simplify reconnaissance for analyzing AMM behavior, potentially aiding sophisticated actors in understanding pool dynamics or identifying arbitrage opportunities more easily.
    *   **Compute Overhead (Low-Medium):** Serialization (`bincode`), base64 encoding, and the `msg!` invocation itself add compute overhead to each logged transaction. This could be a factor in transaction costs and processing times, especially during high network load or for very active pools.
*   **Root Cause:** Unconditional detailed logging.
*   **Recommendations/Mitigations:** Consider making detailed logging conditional via a Cargo feature flag (e.g., `debug-logs`) that can be disabled in production builds to reduce compute overhead and information verbosity.

---

**Finding ID:** SR-04
*   **Title/Summary:** Compute Limits: Potential DoS from OpenBook Event Queue / Order CPIs
*   **Description:**
    1.  `Calculator::calc_exact_vault_in_serum` iterates OpenBook event queues. Very long queues (e.g., from many AMM order fills) could cause this function, and thus callers like `calc_total_without_take_pnl`, to exceed the transaction compute budget.
    2.  `process_monitor_step` (and helpers like `do_cancel_all_orders_state`, `do_place_orders`) and `process_withdraw` can issue multiple CPIs to OpenBook for order management.
*   **Affected Components:** `program/src/math.rs`, `program/src/processor.rs`.
*   **Potential Impact & Severity:** Medium (Denial of Service / Liveness). If critical operations like `Deposit`, `Withdraw`, or `MonitorStep` consistently hit compute limits, the functionality of the affected AMM pool could be impaired.
*   **Root Cause:** Iteration over potentially unbounded external data (event queues); multiple CPIs within a single transaction.
*   **Conceptual Scenario:** An attacker (or high market volatility) causes many small fills of the AMM's orders on OpenBook, significantly lengthening the event queue for that market. Subsequent `MonitorStep` calls or LP operations for that pool fail due to compute exhaustion in `calc_exact_vault_in_serum`.
*   **Recommendations/Mitigations:**
    *   The batching limits (`plan_order_limit`, etc.) in `MonitorStepInstruction` are a good existing mitigation for CPI-heavy operations.
    *   For `calc_exact_vault_in_serum`, consider if its full iteration is always necessary in every path it's called. For less critical paths (e.g., pure simulation if applicable, or less frequent PnL updates), an estimate or a bounded iteration might be an alternative, though this could trade off accuracy. For critical reserve calculations, accuracy is paramount.
    *   Encourage off-chain keepers/services to call `settle_funds` on AMM OpenOrders accounts periodically to keep event queues shorter and ensure funds are promptly reflected in AMM vaults.

---

**Finding ID:** SR-05
*   **Title/Summary:** Economic Risk: OpenBook Price Manipulation Impacting AMM LPs
*   **Description:** The AMM places limit orders on OpenBook based on its internal curve. If an attacker manipulates the OpenBook market price, they can cause these AMM orders to be filled at prices disadvantageous to the AMM LPs, leading to a loss of value from the pool compared to its internal valuation.
*   **Affected Components:** `program/src/processor.rs` (especially `MonitorStep` logic).
*   **Potential Impact & Severity:** Medium (Economic Loss for LPs).
*   **Root Cause:** Dependency on an external market (OpenBook) for fills, where the AMM acts as a price taker for its orders.
*   **Conceptual Scenario:** Attacker uses a flash loan to significantly pump the price of Token A on OpenBook. Raydium's sell orders for Token A (which are part of its AMM liquidity) get filled at this inflated price. The attacker then sells Token A back at a lower price on another venue or as the price corrects. Raydium AMM is left with more PC tokens but fewer Coin tokens, and the overall value held by LPs might be less than if the trade had occurred at the "true" market price.
*   **Recommendations/Mitigations:**
    *   The `min_price_multiplier` and `max_price_multiplier` parameters in `AmmInfo` provide a crucial safeguard by preventing the AMM from placing orders if its internal price calculation deviates extremely from what it perceives as a normal range (relative to its own lot sizes), potentially due to manipulation.
    *   The PnL mechanism aims to capture some of the value from arbitrage.
    *   The spread introduced by `min_separate_numerator` and trade fees makes it more expensive to exploit the AMM's orders.
    *   Further tuning of `order_num`, `depth`, and `vol_max_cut_ratio` can adjust the AMM's sensitivity and exposure to market fluctuations.

---

**Finding ID:** SR-06
*   **Title/Summary:** Data Staleness: OpenBook Event Queue Lag Impacting PnL/Order Optimality
*   **Description:** `calc_exact_vault_in_serum` reads OpenBook's event queue to determine AMM's settled funds in its `OpenOrders` account. If OpenBook's event queue processing is slow or if Raydium's view is limited by compute, the AMM might make decisions based on slightly stale data regarding its exact balances on the DEX.
*   **Affected Components:** `program/src/math.rs` (`calc_exact_vault_in_serum`), `program/src/processor.rs` (functions using `calc_total_without_take_pnl`).
*   **Potential Impact & Severity:** Low.
    *   This primarily affects the precise timing and amount of PnL calculated, as PnL depends on the exact current reserves.
    *   Order planning in `MonitorStep` might be based on slightly outdated reserve information, leading to marginally suboptimal order placement.
    *   Direct fund loss is unlikely as critical operations (withdrawals, parameter changes) force explicit fund settlements. Swaps also use `calc_exact_vault_in_serum`, so they get the most current view possible via this mechanism.
*   **Root Cause:** Eventual consistency of external market data and potential compute limitations in reading long queues.
*   **Recommendations/Mitigations:** The current design, where key flows explicitly settle funds and PnL calculations are performed at the beginning of LP operations, largely mitigates this. For `MonitorStep`, the impact is minor sub-optimality.

---

**Finding ID:** SR-07
*   **Title/Summary:** Admin Controls & Centralization Risks (Overall)
*   **Description:** The `config_feature::amm_owner::ID` (a hardcoded pubkey per deployment feature) has significant control over pool parameters via `process_set_params` and global AMM configuration via `process_update_config_account`.
*   **Affected Components:** `program/src/processor.rs` (`process_set_params`, `process_update_config_account`), `program/src/state.rs` (`AmmInfo.fees`, `AmmInfo.amm_owner`, `AmmConfig.pnl_owner`, etc.).
*   **Potential Impact & Severity:** High (Centralization Risk / Potential for Admin Malice or Key Compromise).
    *   **Economic Disruption:** Admin can set extreme fees (e.g., swap fees near 100%), effectively trapping funds or making the pool unusable for swaps. Can set PnL fee to capture almost all PnL for the `pnl_owner`.
    *   **Operational Disruption:** Can change `order_num`, `depth`, price multipliers to values that halt effective market making or create highly unfavorable order book conditions. Can change pool status to `Disabled` or `WithdrawOnly`.
    *   **Ownership/Control Transfer:** Admin can change `AmmInfo.amm_owner` (transferring operational control of that pool's parameters) and `AmmConfig.pnl_owner` (redirecting PnL revenue).
    *   **Direct Fund Theft (Limited):** Admin cannot directly drain user funds from AMM vaults or user LP positions. However, they can make it economically infeasible for users to interact or profit.
*   **Root Cause:** Centralized administrative privileges.
*   **Recommendations/Mitigations:**
    *   Implement a timelock for critical parameter changes.
    *   Transition admin controls to a multisig wallet or a DAO structure.
    *   Further harden validation for all settable parameters in `process_set_params` (e.g., stricter bounds, cross-validation between min/max multipliers).

---

**Finding ID:** SR-08
*   **Title/Summary:** Math: Minor Truncation Risk with `.as_u64()` in Invariant Calculations
*   **Description:** In `InvariantToken` and `InvariantPool` methods within `math.rs`, `U128` results from multiplications/divisions are converted back to `u64` using `.as_u64()`. This method truncates if the `U128` value exceeds `u64::MAX`.
*   **Affected Components:** `program/src/math.rs` (methods of `InvariantToken`, `InvariantPool`).
*   **Potential Impact & Severity:** Very Low. The contexts where this is used (calculating proportional token amounts for deposits, or token/LP amounts for withdrawals/mints) usually involve inputs that are already `u64`, and the intermediate `U128` products are unlikely to result in a final value (after division) that exceeds `u64::MAX` in realistic scenarios. If it did, it would be a silent loss of precision.
*   **Root Cause:** Direct conversion from `U128` to `u64` without explicit overflow checking via `Calculator::to_u64`.
*   **Recommendations/Mitigations:** For maximum robustness, replace `.as_u64()` with `Calculator::to_u64(...)` which returns a `Result` and would allow explicit error handling for `AmmError::ConversionFailure`.

---

**Finding ID:** SR-09
*   **Title/Summary:** Documentation: `CheckedCeilDiv` Trait Comment vs. Implementation Clarity
*   **Description:** The descriptive comment for the `CheckedCeilDiv` trait in `math.rs` suggests a more complex divisor recalculation logic than what is implemented for the `U128` and `u128` types. The implementations simply ceiling the quotient and return the original divisor.
*   **Affected Components:** `program/src/math.rs` (comments for `CheckedCeilDiv` trait).
*   **Potential Impact & Severity:** Very Low (Documentation Clarity). Could slightly mislead developers relying solely on the trait comment to understand the precise behavior of the `U128`/`u128` implementations.
*   **Root Cause:** Mismatch between the general concept described in the comment and the specific, simpler implementation chosen.
*   **Recommendations/Mitigations:** Update the comment for the `CheckedCeilDiv` trait or add specific comments to the `U128`/`u128` implementations to accurately reflect their simpler behavior (ceiling quotient, return original divisor).

---

## Phase 2: Deep Dive Vulnerability Research

### Methodology

This second phase of the audit involved a more intensive, attacker-centric approach. The focus was on identifying complex or subtle vulnerabilities that might arise from:
*   Interactions between mathematical calculations (fees, rounding, normalization, quantization).
*   State discrepancies or timing issues in interactions with the external OpenBook DEX.
*   Scenarios that could lead to critical outcomes such as direct theft of funds, permanent freezing of funds, permanent Denial of Service (DoS), or illegitimate LP minting, emulating patterns seen in Immunefi bug reports and other AMM exploits.

Specific areas investigated included:
*   Detailed tracing of PnL calculations and LP token minting/burning logic (`Processor::calc_take_pnl`, `InvariantPool` methods, `process_deposit`, `process_withdraw`).
*   Analysis of swap fee arithmetic and its interaction with core swap calculations (`process_swap_base_in`, `process_swap_base_out`).
*   Impact of `sys_decimal_value` and quantization parameters (OpenBook lot sizes, `AmmInfo.min_size`, price tick rounding) on precision and potential biases.
*   State synchronization vulnerabilities related to `process_monitor_step`, its helper functions (`do_plan_orderbook`, `do_place_orders`, etc.), and the usage of `math.rs::Calculator::calc_exact_vault_in_serum` when reading OpenBook event queues.

### Overall Conclusion of Deep Dive Phase

This intensive deep dive vulnerability research phase **did not identify any new, previously unknown critical vulnerabilities** in the Raydium AMM's core logic that would allow a non-admin attacker to directly steal funds from AMM vaults, permanently freeze user funds within the AMM, cause a permanent Denial of Service to core AMM functionalities (swaps, deposits, withdrawals) under normal OpenBook DEX operations, or illegitimately mint LP tokens to dilute other providers.

The AMM's internal accounting for fees, the consistent application of rounding rules (generally favoring the pool/LPs), the sequence of PnL calculation prior to LP operations, and the mechanisms for handling state with OpenBook (like `calc_exact_vault_in_serum` and explicit settlements) were found to be robust against the specific types of mathematical and timing exploits investigated.

This deep dive reinforced the understanding of the risks identified in the initial broader audit phase, primarily:
*   **Centralization/Admin Controls (Finding SR-07, SR-01, SR-02):** The `AmmOwner` possesses significant power to alter pool parameters, which, if misused or compromised, could lead to economic disruption or PnL redirection.
*   **Compute Limit DoS (Finding SR-04):** Operations interacting heavily with OpenBook event queues (`calc_exact_vault_in_serum`) or involving multiple CPIs remain susceptible to Solana's compute limits, potentially causing liveness issues for certain AMM functions if external conditions are adverse (e.g., very long event queues).
*   **Economic Risks from OpenBook Interaction (Finding SR-05):** The AMM, when placing orders on OpenBook, is subject to broader market dynamics, including potential price manipulation on OpenBook, which can affect LP returns.
*   **Minor Data Staleness (Finding SR-06):** While generally well-handled, slight discrepancies due to the eventual consistency of OpenBook event queue data are possible but are unlikely to lead to significant value loss.

### New Recommendations from Deep Dive Phase

While no new critical vulnerabilities were found, the deep dive did highlight one area where a preventative improvement could be made:

**Finding ID:** SR-10 (New from Deep Dive)
*   **Title/Summary:** Validate Token Decimals During Pool Initialization
*   **Description:** The `Calculator::normalize_decimal_v2` and `Calculator::restore_decimal` functions in `math.rs` use `U128::from(10).checked_pow(native_decimal.into())`. If `native_decimal` (sourced from `amm.coin_decimals` or `amm.pc_decimals` in `AmmInfo`, which are in turn read from SPL Mint accounts during `process_initialize2`) is an extremely large value (e.g., >38, which is beyond standard SPL token decimal values but theoretically possible for a malicious mint), the `checked_pow` operation could return `None`, leading to a panic due to an `unwrap()` call within these math functions. This would cause the `process_initialize2` instruction to fail.
*   **Affected Components:** `program/src/math.rs` (normalization functions), `program/src/processor.rs` (`process_initialize2`).
*   **Potential Impact & Severity:** Low-Medium (DoS for Pool Creation). An attacker could craft a malicious token mint with excessively large decimals and prevent new Raydium pools from being created with this token. Existing pools are unaffected.
*   **Root Cause:** Lack of an upper bound check on token decimal values during pool initialization.
*   **Conceptual Scenario:** Attacker creates a new SPL Token with `decimals = 50`. Attacker then attempts to call `process_initialize2` to create a Raydium pool with this token. The `amm.initialize()` call within `process_initialize2` would eventually lead to normalization/denormalization calls in `math.rs` that use `10^50`, which would overflow `U128::checked_pow`, causing a panic and preventing pool creation.
*   **Recommendations/Mitigations:** In `process_initialize2`, after unpacking `coin_mint` and `pc_mint`, add validation to ensure `coin_mint.decimals` and `pc_mint.decimals` are within a practical and safe upper bound (e.g., `<= 18` or a system-wide chosen maximum like 15, which aligns with common UI representations and avoids `U128` `checked_pow` overflow for base 10). Return `AmmError::InvalidInput` if they exceed this bound.

---

## Final Conclusion (Incorporating All Phases)

The Raydium AMM protocol is a sophisticated system that integrates deeply with OpenBook DEX. The audit, encompassing both a broad review and targeted deep dives, found that the core mechanics for swaps, liquidity provision, PnL distribution, and fee handling are logically sound and robust against direct theft by non-admin actors or common mathematical exploits. Rounding strategies consistently favor the pool or existing LPs.

The most significant risks identified are inherent to its design and operational context:
1.  **Centralized Admin Privileges:** The `AmmOwner` has substantial control over pool parameters, posing a risk if the key is compromised or misused.
2.  **External Market Dependencies:** The AMM's performance and LP returns can be affected by price manipulation or volatility on the OpenBook DEX.
3.  **Solana Compute Limits:** Certain operations involving extensive iteration over OpenBook event queues or multiple CPIs may face liveness or efficiency challenges under heavy load.

Minor issues related to parameter validation (SR-01, SR-02, new SR-10), potential information disclosure/compute overhead via logging (SR-03), and documentation clarity (SR-09) have been noted with recommendations. The deep dive phase confirmed the robustness of core financial calculations against specific exploit patterns but reinforced the importance of managing the identified systemic risks.
