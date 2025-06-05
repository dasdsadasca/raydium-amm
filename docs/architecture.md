# Raydium AMM Architecture

This document outlines the architecture of the Raydium Automated Market Maker (AMM), detailing its core components, mathematical underpinnings, and interaction with the broader Solana ecosystem, particularly the OpenBook DEX.

## 1. Overall Architecture

The Raydium AMM is a decentralized exchange protocol built on the Solana blockchain. Its architecture is designed for high-speed, low-cost trading and liquidity provision. The key components are:

*   **Core AMM Program:** This is the central smart contract that houses the business logic for all AMM operations. It processes user instructions, manages liquidity pools, executes swaps, and interacts with other necessary Solana programs.
*   **State Accounts:** Several on-chain accounts store the persistent state of the AMM:
    *   **`AmmInfo` Account:** The primary account for each liquidity pool, storing crucial parameters like token mints, vault addresses, fee configurations, current liquidity state (e.g., total LP tokens, token balances implicitly through vaults), and operational status.
    *   **`TargetOrders` Account:** Associated with each pool, this account stores the AMM's planned limit orders that are to be placed on the OpenBook DEX. It tracks their prices, volumes, and management state.
    *   **`AmmConfig` Account:** A global configuration account for the Raydium AMM program, holding parameters like administrative owners and pool creation fees.
*   **Token Vaults:** For each liquidity pool, there are two SPL Token accounts (vaults) owned by a Program Derived Address (PDA) of the AMM program:
    *   **Coin Vault:** Holds the "coin" token of the pair (e.g., RAY in a RAY-USDC pool).
    *   **PC (Price Currency) Vault:** Holds the "price currency" token of the pair (e.g., USDC in a RAY-USDC pool).
    When users provide liquidity, their tokens are transferred into these vaults. When they swap, tokens are taken from one vault and added to the other.
*   **LP Mint Account:** Each pool has a unique SPL Token mint for its Liquidity Provider (LP) tokens. When users deposit liquidity, they receive newly minted LP tokens representing their share of the pool. These LP tokens can be burned to withdraw their underlying liquidity.
*   **OpenOrders Account (per pool):** An account required by the OpenBook DEX for each trading pair. The AMM uses this account, also controlled by its PDA, to place and manage its limit orders on the OpenBook order book.

**Interaction Flow (Simplified):**
1.  A user initiates an instruction (e.g., swap, deposit) via a frontend or directly.
2.  The instruction is sent to the Raydium AMM Program.
3.  The program's `Processor` decodes the instruction.
4.  Based on the instruction:
    *   It reads from and writes to the relevant `AmmInfo` and `TargetOrders` state accounts.
    *   It may interact with the Token Vaults (via the SPL Token Program) to transfer tokens.
    *   It may mint or burn LP tokens (via the SPL Token Program).
    *   It may place, cancel, or settle orders on the OpenBook DEX (via CPIs to the OpenBook Program).
5.  Mathematical calculations (e.g., swap price, order parameters) are performed using the `math.rs` module.

## 2. Constant Product Formula

The fundamental pricing mechanism for direct swaps within Raydium (when not routing through OpenBook's order book for the best price) and for determining the ratio of tokens when adding/removing liquidity is the **constant product formula**:

`x * y = k`

Where:
*   `x`: The quantity of token A in the liquidity pool.
*   `y`: The quantity of token B in the liquidity pool.
*   `k`: A constant value representing the total liquidity of the pool.

**How it works:**
*   **Constant `k`:** When liquidity is added or removed, tokens are deposited or withdrawn in a way that, ideally, keeps `k` constant relative to the existing liquidity providers (though `k` itself changes with the total liquidity). For a swap, `k` should remain constant (before fees).
*   **Pricing:** When a user swaps token A for token B, they add token A to the pool (increasing `x`) and remove token B from the pool (decreasing `y`). To maintain the constant `k`, the ratio of `x` to `y` changes, effectively determining the price. The price of a token is the ratio of the reserves.
*   **Slippage:** Larger trades relative to the pool's total liquidity (i.e., a significant change in `x` or `y`) will cause a larger price impact or "slippage" because the ratio must shift more dramatically to maintain `k`.
*   **Fees:** A small fee is typically taken from each swap. This fee is usually added back to the liquidity pool, increasing `k` over time and thus accruing value to liquidity providers.

Raydium's AMM uses this formula for its internal pool operations and as a basis for its order placement strategy on OpenBook.

## 3. OpenBook DEX Integration

A key feature of Raydium is its ability to share liquidity with the OpenBook DEX, a central limit order book (CLOB) on Solana. This hybrid approach allows Raydium to offer both AMM-style swaps and order book-based trading.

**Mechanism:**
1.  **Order Placement:** Instead of keeping all its liquidity idle within its own vaults, the Raydium AMM takes a significant portion of its pooled assets and places them as a series of limit orders on the corresponding OpenBook market's order book. These orders are controlled by the AMM program's PDA via its `OpenOrders` account for that market.
2.  **Fibonacci Sequence for Order Distribution:** To create a diverse and resilient order book presence, Raydium distributes these limit orders across various price points. This distribution often follows a pattern related to a **Fibonacci sequence** (or a similar distribution logic defined in its `math.rs` and `processor.rs` modules).
    *   The AMM calculates a set of buy and sell orders at different price levels around the current market price.
    *   The size and price spacing of these orders are determined by parameters within the `AmmInfo` state (like `order_num`, `depth`) and the current pool composition. The Fibonacci sequence helps in creating denser liquidity closer to the current price and sparser liquidity further away, mimicking a typical order book depth.
    *   The `TargetOrders` account stores these planned orders, and the `Processor`'s state machine (`MonitorStep` instruction) periodically updates these orders on OpenBook to reflect changes in the AMM's pool price or market conditions.
3.  **Shared Liquidity:**
    *   When a user on Raydium performs a swap, Raydium can route this trade either against its own internal pool (using the x*y=k formula) or against the OpenBook order book (which includes Raydium's own limit orders and orders from other OpenBook participants). Raydium typically chooses the path that offers the best price for the user.
    *   Traders interacting directly with OpenBook can also trade against Raydium's limit orders, effectively accessing Raydium's liquidity.

**Benefits:**
*   **For Traders:**
    *   **Improved Price Discovery:** Access to deeper liquidity from both the AMM pool and the OpenBook order book often results in better execution prices and reduced slippage.
    *   **Versatility:** Traders can choose between simple AMM swaps or placing limit orders themselves.
*   **For Liquidity Providers (LPs):**
    *   **Increased Capital Efficiency:** Liquidity is not just sitting passively in the pool but is actively working on the OpenBook DEX, potentially earning trading fees from OpenBook trades in addition to Raydium swap fees.
    *   **Reduced Impermanent Loss (Potentially):** While not eliminating impermanent loss, the active management and fee generation from order book participation can help mitigate it compared to a purely passive AMM. The AMM's rebalancing of orders on OpenBook aims to follow the market price.

## 4. Key Contracts Interaction

The Raydium AMM program does not operate in isolation. It interacts with several other key Solana programs:

*   **SPL Token Program (`spl_token::id()`):**
    *   **Token Transfers:** Used for transferring tokens between user accounts and AMM vaults during deposits, withdrawals, and swaps.
    *   **LP Token Minting/Burning:** Used to mint LP tokens to liquidity providers and burn them upon withdrawal.
    *   **Vault Management:** The AMM's token vaults are SPL Token accounts.
*   **OpenBook DEX Program (e.g., `config_feature::openbook_program::id()`):**
    *   **Order Placement:** The AMM places its limit orders onto OpenBook's order books by calling instructions on the OpenBook program.
    *   **Order Cancellation:** Modifies or cancels its existing orders on OpenBook.
    *   **Fund Settlement:** After its orders are matched on OpenBook, the AMM calls OpenBook instructions to settle the resulting token exchanges into its own vaults.
    *   **Market State Reading:** Reads market information (like best bids/asks, lot sizes) from OpenBook's market accounts.
*   **System Program (`solana_program::system_program::id()`):**
    *   Used for creating new accounts on Solana, such as the `AmmInfo`, `TargetOrders`, and various PDA-controlled accounts.
*   **Rent Sysvar (`sysvar::rent::id()`):**
    *   Consulted to ensure new accounts are rent-exempt.
*   **Clock Sysvar (`sysvar::clock::id()`):**
    *   Used to get the current timestamp, which can be relevant for features like pool open times or other time-dependent logic.
*   **Associated Token Account Program (`spl_associated_token_account::id()`):**
    *   Often used by clients (and potentially internally for some PDA setups, though the AMM also uses direct PDA-owned vaults) to manage token accounts.

These interactions are primarily handled by the `invokers.rs` module within the Raydium AMM, which provides safe wrappers for making CPIs to these external programs.
