# Raydium AMM Glossary

This glossary provides definitions for important terms and concepts related to the Raydium AMM protocol.

---

**A**

*   **AMM (Automated Market Maker):** A type of decentralized exchange (DEX) protocol that relies on mathematical formulas (like the constant product formula) to price assets. Liquidity is provided by users into pools, and trades are executed against these pools.
*   **AMM Authority:** A Program Derived Address (PDA) that has control over the AMM's token vaults, LP token mint, and OpenOrders account on the OpenBook DEX. It signs on behalf of the AMM program for these operations.
*   **AMM ID / Pool ID:** The public key of the `AmmInfo` account for a specific liquidity pool. It uniquely identifies the pool within the Raydium protocol.

**C**

*   **Coin Token / Base Token:** In a trading pair (e.g., RAY/USDC), the Coin Token (or Base Token) is the token being priced (RAY in this example). Its value is expressed in terms of the PC Token.
*   **Constant Product Formula (x\*y=k):** The core mathematical formula used by many AMMs, including Raydium for its internal pool logic. It states that the product of the quantities of two tokens in a liquidity pool (`x` and `y`) must remain constant (`k`) during trades (before fees). This formula determines the price at which swaps occur.

**F**

*   **Fees (Trading Fees, LP Fees):**
    *   **Trading Fees:** Small fees charged on token swaps. A portion of these fees typically goes to liquidity providers as a reward, and another portion may go to the protocol.
    *   **LP Fees:** Fees earned by liquidity providers from the trading activity in the pool. These are implicitly accrued as the value of their LP tokens increases due to fee accumulation in the pool.
*   **Fibonacci Orders:** Refers to Raydium's strategy of placing multiple limit orders on the OpenBook DEX order book at price points often determined by a Fibonacci sequence (or a similar distribution logic). This creates a graduated depth of liquidity around the current market price.

**I**

*   **Instructions:** Operations that can be invoked on the Raydium AMM smart contract. Key instructions include `Initialize2` (create pool), `Deposit` (add liquidity), `Withdraw` (remove liquidity), `SwapBaseIn`/`SwapBaseOut` (exchange tokens), and `MonitorStep` (manage OpenBook orders).

**L**

*   **Limit Order:** An order to buy or sell an asset at a specific price or better. These are placed on an order book (like OpenBook's) and are only executed if the market price reaches the specified limit price. Raydium AMM places limit orders on OpenBook.
*   **Liquidity Pool:** A collection of two different tokens locked in a smart contract (held in Token Vaults) that traders can trade against. Users who provide tokens to these pools are called Liquidity Providers.
*   **LP Tokens (Liquidity Provider Tokens):** Special tokens minted to users when they deposit assets into a liquidity pool. These tokens represent their proportional share of the pool's total liquidity and can be burned to redeem their underlying assets and accrued fees.

**M**

*   **Market Order:** An order to buy or sell an asset immediately at the best available current price on the order book.
*   **MonitorStep:** An instruction in the Raydium AMM used to manage its order book presence on OpenBook DEX. It handles the state machine for planning, placing, cancelling, and purging orders.

**O**

*   **OpenBook DEX:** A decentralized, central limit order book (CLOB) exchange built on the Solana blockchain. Raydium AMM integrates with OpenBook by placing its liquidity as limit orders on OpenBook's markets, allowing for shared liquidity between the AMM and the order book.
*   **OpenOrders Account:** An account required by the OpenBook DEX for each user (or program like Raydium) trading on a specific market. It tracks the user's open orders and settled/unsettled funds for that market. Raydium uses an OpenOrders account (controlled by its AMM Authority PDA) for each of its liquidity pools to manage its orders on OpenBook.
*   **Order Book:** A list of buy orders (bids) and sell orders (asks) for a specific asset, organized by price level. It's a core component of traditional exchanges and CLOB DEXs like OpenBook.

**P**

*   **PC Token / Quote Token:** In a trading pair (e.g., RAY/USDC), the PC (Price Currency) Token (or Quote Token) is the token in which the price of the Coin Token is expressed (USDC in this example).
*   **Pool ID:** See AMM ID.

**S**

*   **Serum:** The original name for the DEX protocol on Solana that OpenBook is a community-led fork of. Functionally, OpenBook DEX carries forward the Serum architecture.
*   **Slippage:** The difference between the expected price of a trade and the price at which the trade is actually executed. It often occurs in AMMs when a trade is large relative to the pool's liquidity, causing the price to move as the trade is processed.
*   **SPL Token Program:** The standard program on Solana for creating, managing, and transferring fungible and non-fungible tokens (SPL Tokens). Raydium AMM relies on it for its token vaults and LP tokens.
*   **State Accounts:** On-chain accounts that store the persistent data and state of the Raydium AMM. The primary state account for a pool is the `AmmInfo` account. Other state accounts include `TargetOrders` and `AmmConfig`.

**T**

*   **TargetOrders Account:** A Raydium-specific state account associated with each AMM pool. It stores the AMM's planned limit orders (prices and volumes) that are intended to be placed on the OpenBook DEX, along with parameters for managing these orders.
*   **Token Vaults:** SPL Token accounts controlled by the AMM Authority PDA. Each liquidity pool has two vaults: one for the Coin Token and one for the PC Token. These vaults hold the underlying liquidity of the pool.
