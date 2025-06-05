# Raydium AMM Glossary

This glossary provides definitions for important terms and concepts related to the Raydium AMM protocol.

---

**A**

*   **AMM (Automated Market Maker):** A type of decentralized exchange (DEX) protocol that relies on mathematical formulas (like the constant product formula) to price assets. Liquidity is provided by users into pools, and trades are executed against these pools.
*   **AMM Authority:** A Program Derived Address (PDA) that has control over the AMM's token vaults, LP token mint, and OpenOrders account on the OpenBook DEX. It signs on behalf of the AMM program for these operations.
*   **AMM ID / Pool ID:** The public key of the `AmmInfo` account for a specific liquidity pool. It uniquely identifies the pool within the Raydium protocol.
*   **`AmmConfig` Account:** A global singleton account that stores configuration parameters for the Raydium AMM program, such as the PNL (Profit and Loss) owner and pool creation fees.
*   **`AmmInfo` Account:** The primary state account for each individual Raydium liquidity pool. It stores crucial parameters like token mints, vault addresses, fee configurations, current liquidity state (e.g., total LP tokens), operational status, and parameters for the AMM's order placement strategy on OpenBook.

**C**

*   **Clock Sysvar:** A Solana system account that provides access to on-chain clock information, including the current slot, epoch, and approximate timestamp. Used by Raydium for time-dependent operations like `pool_open_time`.
*   **Coin Token / Base Token:** In a trading pair (e.g., RAY/USDC), the Coin Token (or Base Token) is the token being priced (RAY in this example). Its value is expressed in terms of the PC Token.
*   **Constant Product Formula (x\*y=k):** The core mathematical formula used by many AMMs, including Raydium for its internal pool logic. It states that the product of the quantities of two tokens in a liquidity pool (`x` and `y`) must remain constant (`k`) during trades (before fees). This formula determines the price at which swaps occur.
*   **CPI (Cross-Program Invocation):** A mechanism on Solana that allows one smart contract (e.g., Raydium AMM) to call instructions on another smart contract (e.g., SPL Token Program, OpenBook DEX Program) within the same transaction.

**E**

*   **Effective Reserves:** The actual amount of tokens considered by the AMM for pricing and liquidity calculations. This typically includes tokens held directly in the AMM's vaults plus any tokens settled in its OpenOrders account on the OpenBook DEX, adjusted for any pending (unclaimed) PnL.

**F**

*   **Fees (Trading Fees, LP Fees, PnL Fees):**
    *   **Swap Fees:** Fees charged on direct token swaps executed against the AMM pool. A portion accrues to LPs.
    *   **Trading Fees (OpenBook):** Fees charged by the OpenBook DEX when Raydium's limit orders are matched. These can also contribute to LP earnings.
    *   **PnL (Profit and Loss) Fees/Distribution:** Raydium's mechanism to separate a portion of accumulated gains (from fees or favorable price movements of its OpenBook orders) into `need_take_pnl_coin/pc` accounts. This PnL is eventually claimable by a designated PnL owner. The PnL numerator/denominator in `AmmInfo.fees` determines this split.
*   **Fibonacci Orders:** Refers to Raydium's strategy of placing multiple limit orders on the OpenBook DEX order book at price points often determined by a Fibonacci sequence (or a similar distribution logic from `math.rs`). This creates a graduated depth of liquidity around the current market price.

**I**

*   **Impermanent Loss:** A potential risk for liquidity providers in AMMs. It's the difference in value between holding assets in an AMM pool versus holding them in a wallet, if the market price of the tokens changes significantly.
*   **Instructions:** Operations that can be invoked on the Raydium AMM smart contract. Key instructions include `Initialize2` (create pool), `Deposit` (add liquidity), `Withdraw` (remove liquidity), `SwapBaseIn`/`SwapBaseOut` (exchange tokens), and `MonitorStep` (manage OpenBook orders).

**L**

*   **Limit Order:** An order to buy or sell an asset at a specific price or better. These are placed on an order book (like OpenBook's) and are only executed if the market price reaches the specified limit price. Raydium AMM places limit orders on OpenBook.
*   **Liquidity Pool:** A collection of two different tokens locked in a smart contract (held in Token Vaults) that traders can trade against. Users who provide tokens to these pools are called Liquidity Providers.
*   **Lot Sizes:** In the context of OpenBook DEX, these define the minimum tradable unit for price (tick size) and quantity (step size). Raydium's internal calculations are normalized but must be converted to respect these lot sizes when placing orders.
*   **LP Tokens (Liquidity Provider Tokens):** Special tokens minted to users when they deposit assets into a liquidity pool. These tokens represent their proportional share of the pool's total liquidity and can be burned to redeem their underlying assets and accrued fees.

**M**

*   **Market Order:** An order to buy or sell an asset immediately at the best available current price on the order book.
*   **MonitorStep:** An instruction in the Raydium AMM used to manage its order book presence on OpenBook DEX. It handles the state machine for planning, placing, cancelling, and purging orders based on the AMM's strategy and current market conditions.

**N**

*   **Normalized Values / `sys_decimal_value`:** Raydium AMM converts token amounts into a common internal representation using `AmmInfo.sys_decimal_value`. This allows consistent mathematical operations across tokens with different native decimal precisions. Results are then denormalized back to native token units when needed.

**O**

*   **OpenBook DEX:** A decentralized, central limit order book (CLOB) exchange built on the Solana blockchain. Raydium AMM integrates with OpenBook by placing its liquidity as limit orders on OpenBook's markets, allowing for shared liquidity between the AMM and the order book.
*   **OpenOrders Account:** An account required by the OpenBook DEX for each user (or program like Raydium) trading on a specific market. It tracks the user's open orders and settled/unsettled funds for that market. Raydium uses an OpenOrders account (controlled by its AMM Authority PDA) for each of its liquidity pools to manage its orders on OpenBook.
*   **Order Book:** A list of buy orders (bids) and sell orders (asks) for a specific asset, organized by price level. It's a core component of traditional exchanges and CLOB DEXs like OpenBook.

**P**

*   **PC Token / Quote Token:** In a trading pair (e.g., RAY/USDC), the PC (Price Currency) Token (or Quote Token) is the token in which the price of the Coin Token is expressed (USDC in this example).
*   **PDA (Program Derived Address):** An address derived from a program ID and a set of seeds, which can be used to sign transactions on behalf of a program. Raydium uses PDAs for its AMM Authority and to own various accounts.
*   **PnL (Profit and Loss):** In Raydium, this refers to gains accrued by the AMM, often from swap fees or favorable movements of its orders on OpenBook. The `calc_take_pnl` function identifies and separates this PnL, which is then stored in `StateData.need_take_pnl_coin/pc` fields to be claimed.
*   **Pool ID:** See AMM ID.

**R**

*   **Rent Sysvar:** A Solana system account that provides information about the current rent-exemption requirements for accounts. Smart contracts consult this to ensure newly created accounts are funded with enough lamports to be rent-exempt.

**S**

*   **Serum:** The original name for the DEX protocol on Solana that OpenBook is a community-led fork of. Functionally, OpenBook DEX carries forward the Serum architecture.
*   **Slippage:** The difference between the expected price of a trade and the price at which the trade is actually executed. It often occurs in AMMs when a trade is large relative to the pool's liquidity, causing the price to move as the trade is processed.
*   **SPL Token Program:** The standard program on Solana for creating, managing, and transferring fungible and non-fungible tokens (SPL Tokens). Raydium AMM relies on it for its token vaults and LP tokens.
*   **State Accounts:** On-chain accounts that store the persistent data and state of the Raydium AMM. Key state accounts include `AmmInfo` (per-pool), `TargetOrders` (per-pool), and `AmmConfig` (global).

**T**

*   **TargetOrders Account:** A Raydium-specific state account associated with each AMM pool. It stores the AMM's planned limit orders (prices and volumes) that are intended to be placed on the OpenBook DEX, along with parameters and state variables for managing these orders and tracking the PnL baseline (`calc_pnl_x`, `calc_pnl_y`).
*   **Tick Size:** The minimum price increment for orders on an OpenBook market, related to `pc_lot_size`.
*   **Token Vaults:** SPL Token accounts controlled by the AMM Authority PDA. Each liquidity pool has two vaults: one for the Coin Token and one for the PC Token. These vaults hold the underlying liquidity of the pool.
