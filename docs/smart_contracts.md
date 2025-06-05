# Smart Contracts Documentation

This document provides an overview of the smart contracts that make up the Raydium AMM (Automated Market Maker) program.

## Table of Contents

- [lib.rs](#librs)
- [entrypoint.rs](#entrypointrs)
- [instruction.rs](#instructionrs)
- [processor.rs](#processorrs)
- [state.rs](#staters)
- [error.rs](#errorrs)
- [invokers.rs](#invokersrs)
- [log.rs](#logrs)
- [math.rs](#mathrs)

---

## `lib.rs`

**Overall Purpose:**
This file serves as the main library crate for the Raydium AMM program. It declares the various modules that constitute the program and re-exports necessary types and constants for external users or downstream crates. It also includes security contact information and declares the program's ID.

**Key Elements:**
- Module declarations: `entrypoint`, `error`, `instruction`, `invokers`, `math`, `processor`, `state`, `log`.
- Re-export of `solana_program` for consistent SDK types.
- Security .txt declaration providing contact and policy information for security researchers.
- Program ID declaration using `solana_program::declare_id!`.

**AMM Logic Fit:**
`lib.rs` is the root of the smart contract, organizing all other components. It doesn't contain direct AMM logic itself but rather structures the project and provides the necessary exports for the program to function as a cohesive unit and be interoperable within the Solana ecosystem.

---

## `entrypoint.rs`

**Overall Purpose:**
This file defines the main entry point for the Solana program, as required by the Solana runtime. It handles the initial processing of incoming instructions.

**Key Functions/Structs:**
- `process_instruction(program_id: &Pubkey, accounts: &'a [AccountInfo<'a>], instruction_data: &[u8]) -> ProgramResult`: This is the core function that the Solana runtime calls when the program is invoked. It receives the program ID, a slice of account information, and the raw instruction data.
- It calls `Processor::process` to handle the actual instruction logic.
- Includes error handling to print program-specific errors if processing fails.

**AMM Logic Fit:**
The `entrypoint.rs` file is the gateway for all interactions with the AMM. Every instruction sent to the AMM program first passes through `process_instruction`. It then delegates the detailed business logic to the `Processor` module.

---

## `instruction.rs`

**Overall Purpose:**
This file defines the structure and serialization/deserialization logic for all instructions that the AMM program can process. It specifies the data layout for each instruction type.

**Key Enums/Structs:**
- **`AmmInstruction` (enum):** This is the central enum that lists all possible actions a user can perform with the AMM. Each variant corresponds to a specific operation (e.g., `Initialize2`, `Deposit`, `Withdraw`, `SwapBaseIn`, `SwapBaseOut`, `SetParams`).
- **Instruction Data Structs (e.g., `InitializeInstruction2`, `DepositInstruction`, `SwapInstructionBaseIn`, `SetParamsInstruction`):** These structs define the parameters required for each instruction. For example:
    - `InitializeInstruction2`: `nonce`, `open_time`, `init_pc_amount`, `init_coin_amount`.
    - `DepositInstruction`: `max_coin_amount`, `max_pc_amount`, `base_side`.
    - `SwapInstructionBaseIn`: `amount_in`, `minimum_amount_out`.
- **Helper functions for packing and unpacking instructions:**
    - `unpack(input: &[u8]) -> Result<Self, ProgramError>`: Parses a byte slice into an `AmmInstruction`.
    - `pack(&self) -> Result<Vec<u8>, ProgramError>`: Serializes an `AmmInstruction` into a byte vector.
- **Instruction constructor functions (e.g., `initialize2`, `deposit`, `swap_base_in`):** These are utility functions (often used by clients) to easily create `Instruction` objects that can be sent to the Solana runtime. They package the `AmmInstruction` data with the necessary account metadata.

**AMM Logic Fit:**
`instruction.rs` defines the API of the smart contract. It dictates how users and other programs can interact with the AMM, what actions they can request, and what data they need to provide for each action. The `Processor` then uses these structured instructions to perform the corresponding AMM operations.

---

## `processor.rs`

**Overall Purpose:**
This file contains the core business logic for the AMM program. It takes the deserialized instructions from `instruction.rs` and processes them, interacting with the AMM's `state.rs`, performing calculations using `math.rs`, and making external calls via `invokers.rs`.

**Key Functions/Structs:**
- **`Processor` (struct):** A unit struct that houses the main processing logic.
- **`process(program_id: &Pubkey, accounts: &[AccountInfo], input: &[u8]) -> ProgramResult`:** The main dispatch function that routes to specific instruction handlers based on the unpacked `AmmInstruction`.
- **Instruction Processing Functions (e.g., `process_initialize2`, `process_deposit`, `process_withdraw`, `process_swap_base_in`, `process_swap_base_out`, `process_set_params`, `process_monitor_step`):** Each of these functions handles the logic for a specific instruction type. This includes:
    - Validating accounts and parameters.
    - Loading and modifying AMM state (`AmmInfo`, `TargetOrders`).
    - Performing token transfers (e.g., user to vault, vault to user).
    - Minting/burning LP tokens.
    - Calculating swap amounts, fees, and slippage.
    - Interacting with the Serum/Openbook DEX for order placement and settlement.
    - Updating PnL (Profit and Loss) metrics.
    - Managing AMM operational states (e.g., `AmmStatus`, `AmmState`).
- **Helper functions:**
    - `unpack_token_account`, `unpack_mint`: For SPL token account deserialization.
    - `load_serum_market_order`: For loading Serum/Openbook market and open orders states.
    - `get_amm_orders`, `get_amm_best_price`, `get_amm_worst_price`: For interacting with and querying AMM orders on the DEX.
    - `calc_take_pnl`: For calculating and distributing profit and loss.
    - `authority_id`: For generating the program derived address (PDA) for the AMM.
    - `check_accounts`: Validates common accounts passed to various instructions.
    - Functions for generating associated SPL token accounts/mints and other PDA-based accounts.
    - State machine functions for order book management (`do_idle_state`, `do_plan_orderbook`, `do_place_orders`, etc.).

**Constants:**
- `AUTHORITY_AMM`, `AMM_ASSOCIATED_SEED`, etc.: Seeds for generating PDAs.
- `config_feature`: Module containing feature-gated configuration like owner addresses and program IDs.

**AMM Logic Fit:**
`processor.rs` is the heart of the AMM. It implements the algorithms and rules for liquidity provision, token swaps, fee collection, and order book management on the integrated DEX. It orchestrates interactions between user inputs, on-chain state, mathematical calculations, and external Solana programs.

---

## `state.rs`

**Overall Purpose:**
This file defines the data structures that represent the on-chain state of the AMM. These structures store all the persistent information about AMM pools, configurations, and operational parameters.

**Key Structs:**
- **`AmmInfo`:** The primary state object for an AMM pool. It contains:
    - `status`: Current operational status of the pool (e.g., `Initialized`, `SwapOnly`).
    - `nonce`: Nonce used for PDA generation.
    - `order_num`, `depth`: Parameters for AMM order creation strategy.
    - `coin_decimals`, `pc_decimals`: Decimals for the two tokens in the pool.
    - `state`: Current internal machine state for order book management.
    - `reset_flag`: Flag to indicate if orders need resetting.
    - `min_size`, `vol_max_cut_ratio`, `amount_wave`: Parameters for order sizing and pricing.
    - `coin_lot_size`, `pc_lot_size`: Lot sizes from the Serum/Openbook market.
    - `min_price_multiplier`, `max_price_multiplier`: Price range constraints.
    - `sys_decimal_value`: A normalization factor for decimal calculations.
    - `fees`: A nested `Fees` struct.
    - `state_data`: A nested `StateData` struct for statistical and operational data.
    - Pubkeys for associated accounts: `coin_vault`, `pc_vault`, `lp_mint`, `open_orders`, `market`, `target_orders`.
    - `amm_owner`: The administrative owner of the AMM.
    - `lp_amount`: Total outstanding LP tokens.
    - `client_order_id`: Counter for client order IDs on the DEX.
- **`TargetOrder`:** Represents a planned order with price and volume.
- **`TargetOrders`:** Stores arrays of planned buy and sell orders, along_with state for managing these orders (e.g., `target_x`, `target_y`, `placed_x`, `placed_y`, `calc_pnl_x`, `calc_pnl_y`).
- **`Fees`:** Contains various fee parameters:
    - `min_separate_numerator`, `min_separate_denominator`: For minimum order separation.
    - `trade_fee_numerator`, `trade_fee_denominator`: For trading fees on AMM orders.
    - `pnl_numerator`, `pnl_denominator`: For profit and loss sharing.
    - `swap_fee_numerator`, `swap_fee_denominator`: For direct swap fees.
- **`StateData`:** Stores dynamic operational data:
    - `need_take_pnl_coin`, `need_take_pnl_pc`: Accumulated PnL to be collected.
    - `total_pnl_pc`, `total_pnl_coin`: Lifetime PnL.
    - `pool_open_time`, `orderbook_to_init_time`: Timestamps for state transitions.
    - `swap_coin_in_amount`, `swap_pc_out_amount`, etc.: Accumulated swap volumes and fees.
- **`AmmConfig`:** Stores global configuration for the AMM program, like PNL owner and pool creation fee.
- **Enums for state management:** `AmmStatus`, `AmmState`, `AmmParams`, `AmmResetFlag`, `SimulateParams`.
- **Helper traits and macros:** `Loadable` trait for easy account data deserialization.

**AMM Logic Fit:**
`state.rs` defines the blueprint for all data stored on-chain by the AMM. The `Processor` reads from and writes to these state structures to manage pool liquidity, track fees, execute trades, and maintain the AMM's operational integrity. The design of these structures is critical for efficient data access and gas usage.

---

## `error.rs`

**Overall Purpose:**
This file defines custom error types for the AMM program. These errors provide specific reasons for instruction processing failures, making debugging and client-side error handling more robust.

**Key Enum:**
- **`AmmError`:** An enum where each variant represents a specific error condition. Examples include:
    - `AlreadyInUse`: Account is already initialized.
    - `InvalidProgramAddress`: PDA mismatch.
    - `InvalidCoinVault`, `InvalidPCVault`: Incorrect token vault accounts.
    - `InvalidMarket`: Incorrect market account.
    - `InvalidOwner`: Incorrect account owner.
    - `InvalidStatus`: AMM is not in a state that permits the current instruction.
    - `ExceededSlippage`: Swap output is less than the minimum specified.
    - `CalculationExRateFailure`: Error during exchange rate calculation.
    - `InsufficientFunds`: Not enough tokens for the operation.
    - `TooManyOpenOrders`: Cannot place more orders on the DEX.
    - `InvalidInput`: General invalid parameter.

**AMM Logic Fit:**
`error.rs` is crucial for program correctness and usability. When an instruction cannot be processed as expected (e.g., due to invalid inputs, insufficient funds, or incorrect state), the `Processor` returns one of these specific errors. This allows developers and users to understand what went wrong and take appropriate action.

---

## `invokers.rs`

**Overall Purpose:**
This file provides helper functions to facilitate Cross-Program Invocations (CPIs) to other Solana programs, primarily the SPL Token Program and the Serum/Openbook DEX program.

**Key Functions/Structs:**
- **`Invokers` (struct):** A unit struct that namespaces the invoker functions.
- **SPL Token Program Invokers:**
    - `create_ata_spl_token`: Creates an associated token account.
    - `token_burn`, `token_burn_with_authority`: Burns SPL tokens.
    - `token_mint_to`: Mints SPL tokens.
    - `token_transfer`, `token_transfer_with_authority`: Transfers SPL tokens.
    - `token_close_with_authority`: Closes a token account.
    - `token_set_authority`: Sets a new authority for a token account or mint.
- **Serum/Openbook DEX Program Invokers:**
    - `invoke_dex_init_open_orders`: Initializes an OpenOrders account on the DEX.
    - `invoke_dex_close_open_orders`: Closes an OpenOrders account.
    - `invoke_dex_replace_order_by_client_id`, `invoke_dex_new_order_v3`: Places or replaces orders on the DEX.
    - `invoke_dex_cancel_order_v2`, `invoke_dex_cancel_orders_by_client_order_ids`: Cancels orders on the DEX.
    - `invoke_dex_settle_funds`: Settles funds from the DEX into the AMM's vaults.

**AMM Logic Fit:**
The AMM relies heavily on other Solana programs. `invokers.rs` abstracts the low-level details of constructing and sending CPIs. For example, when a user deposits tokens, the `Processor` uses an invoker to transfer tokens from the user's account to the AMM's vault. When the AMM places orders on the DEX, it uses invokers to interact with the Serum/Openbook program. This modularity keeps the `Processor` logic cleaner and focused on AMM-specific operations.

---

## `log.rs`

**Overall Purpose:**
This file defines custom logging utilities for the AMM program. It allows for structured logging of important events and data during instruction processing, which is invaluable for debugging and monitoring on-chain activity.

**Key Elements:**
- **`LOG_SIZE` (constant):** Defines a buffer size for log messages.
- **`check_assert_eq!` (macro):** A macro for asserting equality and logging a detailed message with expected and actual pubkeys if the assertion fails. This is more informative than a simple `assert_eq!`.
- **`log_keys_mismatch` (function):** Helper function used by `check_assert_eq!` to format and log pubkey mismatches.
- **`LogType` (enum):** Enumerates different types of log events (e.g., `Init`, `Deposit`, `Withdraw`, `SwapBaseIn`, `SwapBaseOut`).
- **Log Data Structs (e.g., `InitLog`, `DepositLog`, `WithdrawLog`, `SwapBaseInLog`, `SwapBaseOutLog`):** These structs define the specific data to be logged for each event type. They are serializable.
- **`encode_ray_log<T: Serialize>(log: T)`:** A generic function that takes a serializable log struct, serializes it using `bincode`, encodes it into a base64 string, and then emits it as a Solana program log message prefixed with "ray_log: ".
- **`decode_ray_log(log: &str)` (test/client utility):** A function (likely for off-chain tools) to decode the base64 encoded log messages back into their structured format.

**AMM Logic Fit:**
Custom logging via `log.rs` provides a way to capture detailed, structured information about the AMM's operations as they happen on-chain. This is far more powerful than simple `msg!` calls, as the structured logs can be parsed and analyzed by off-chain tools to monitor pool activity, diagnose issues, or gather analytics. For example, every swap can log the input amounts, output amounts, and pool state, providing a clear audit trail.

---

## `math.rs`

**Overall Purpose:**
This file contains the core mathematical logic and data types used by the AMM for calculations related to pricing, liquidity, fees, and order management. It often involves handling large numbers and ensuring precision.

**Key Structs/Enums/Types:**
- **`U256`, `U128` (custom uint types):** Large unsigned integer types (256-bit and 128-bit respectively) used to handle potentially large token amounts and intermediate calculations in the AMM formulas, preventing overflow that might occur with standard `u64`.
- **`SwapDirection` (enum):** Defines the direction of a swap (PC to Coin or Coin to PC).
- **`RoundDirection` (enum):** Specifies whether to round up (Ceiling) or down (Floor) in calculations, crucial for ensuring fairness in token exchanges and LP token minting/burning.
- **`Calculator` (struct):** A unit struct that namespaces various mathematical utility functions:
    - `to_u128`, `to_u64`: Safe type conversions.
    - `calc_x_power`: Used in PnL calculations.
    - `fibonacci`: Generates Fibonacci numbers, potentially for order distribution.
    - `normalize_decimal`, `restore_decimal`, `normalize_decimal_v2`: Functions to handle conversions between native token decimal representations and a common "system decimal value" for consistent internal calculations.
    - `floor_lot`, `ceil_lot`: Adjusts values to the nearest lot size.
    - `convert_out_pc_lot_size`, `convert_in_pc_lot_size`, `convert_in_price`, `convert_price_out`, `convert_in_vol`, `convert_vol_out`: Functions to convert values between the AMM's internal representation and the Serum/Openbook DEX's representation (which uses lot sizes).
    - `calc_exact_vault_in_serum`: Calculates the precise amount of tokens locked in Serum/Openbook orders by iterating through the event queue.
    - `calc_total_without_take_pnl`, `calc_total_without_take_pnl_no_orderbook`: Calculate the total effective liquidity in the pool, excluding pending PnL.
    - `get_max_buy_size_at_price`, `get_max_sell_size_at_price`: Determine the maximum order size the AMM can place at a given price based on its current liquidity and desired curve.
    - `swap_token_amount_base_in`, `swap_token_amount_base_out`: Core functions to calculate swap output given input, or input given output, based on the constant product formula (or a variation).
- **`InvariantToken`, `InvariantPool` (structs):** These likely encapsulate logic related to the AMM's invariant (e.g., K in X*Y=K), providing methods for calculating exchange rates or LP token amounts based on token deposits/withdrawals.
- **`CheckedCeilDiv` (trait):** A trait for performing ceiling division on `u128` and `U128`, ensuring that division results are rounded up to avoid losing fractional value, which is important for fairness.

**AMM Logic Fit:**
`math.rs` is fundamental to the AMM's operation. It provides the tools to:
- Implement the AMM's pricing curve (e.g., constant product).
- Calculate how many tokens a user receives in a swap.
- Determine how many LP tokens a user gets for a deposit or burns for a withdrawal.
- Calculate fees and slippage.
- Manage decimal precision across different tokens.
- Determine appropriate order sizes and prices when interacting with the DEX.
The accuracy and correctness of these mathematical functions are paramount for the financial integrity of the AMM.
