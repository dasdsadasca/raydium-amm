# Raydium AMM Protocol Documentation

Welcome to the official documentation for the Raydium Automated Market Maker (AMM) protocol. This documentation aims to provide a comprehensive understanding of Raydium's architecture, smart contract components, user interaction flows, and key terminology.

Whether you are a developer looking to integrate with Raydium, a security researcher, or a user seeking to understand its inner workings, these documents should provide valuable insights.

## Documentation Structure

This documentation is organized into several key sections, each housed in a separate Markdown file:

*   **`architecture.md`**:
    *   Provides a high-level overview of the Raydium AMM's design.
    *   Explains core concepts like the constant product formula, integration with the OpenBook DEX, and how different components interact.
    *   [Read more:](./architecture.md)

*   **`smart_contracts.md`**:
    *   Offers a detailed breakdown of the individual Rust source files that make up the Raydium AMM smart contract.
    *   For each contract file (e.g., `processor.rs`, `state.rs`, `instruction.rs`), it describes its purpose, key functions/structs, and its role within the overall AMM logic.
    *   [Read more:](./smart_contracts.md)

*   **`user_flows.md`**:
    *   Details the step-by-step processes for common user interactions with the Raydium AMM.
    *   Covers flows such as creating a liquidity pool, depositing liquidity, withdrawing liquidity, and swapping tokens.
    *   Includes Mermaid diagrams embedded within the document to visually represent these flows.
    *   [Read more:](./user_flows.md)

*   **`glossary.md`**:
    *   A comprehensive list of important terms, concepts, and acronyms relevant to the Raydium AMM protocol and decentralized finance (DeFi) on Solana.
    *   Each term is provided with a clear and concise definition for easy reference.
    *   [Read more:](./glossary.md)

### Diagrams

Visual diagrams illustrating the user flows described in `user_flows.md` are defined as Mermaid code. The source `.mmd` files for these diagrams are located in the `docs/diagrams/` directory. However, for ease of reading, these diagrams are directly embedded within the relevant sections of the [`user_flows.md`](./user_flows.md) document.

## Suggested Reading Order

For a comprehensive understanding, we suggest the following reading order:

1.  **`README.md`** (this file) - To understand the documentation structure.
2.  **`glossary.md`** - To familiarize yourself with key terminology that will be used throughout the documentation.
3.  **`architecture.md`** - To get a high-level understanding of how the Raydium AMM protocol is designed and operates.
4.  **`user_flows.md`** - To see how users interact with the protocol for common operations, supported by diagrams.
5.  **`smart_contracts.md`** - For a deeper dive into the specific implementation details of each smart contract module (more technical).

---

We hope this documentation serves as a valuable resource. If you have any questions or find areas for improvement, please consider reaching out to the Raydium team or community.
