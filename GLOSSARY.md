# Glossary

- **Messaging protocol:** infrastructure that delivers verified messages from a contract on one chain to a contract on another.
- **Verification vs execution:** proving a message is real, versus delivering it and running app logic.
- **DVN (Decentralized Verifier Network):** a LayerZero v2 verifier; apps choose required and optional DVNs per pathway.
- **Executor:** a LayerZero service that delivers verified messages to the destination app.
- **RMN (Risk Management Network):** Chainlink CCIP's independently implemented network that blesses or curses message roots.
- **ISM (Interchain Security Module):** Hyperlane's pluggable verification contract chosen by the receiving app.
- **Mailbox:** Hyperlane's core contract for dispatching and processing messages.
- **Guardian:** one of the 19 Wormhole nodes that sign observations.
- **VAA (Verified Action Approval):** a Wormhole message signed by a Guardian quorum.
- **Gateway:** Axelar's destination contract that records approved cross-chain calls.
- **Peer / trusted remote / enrolled router:** the allowlisted sender contract on a source chain.
- **Rate limit (token bucket):** a cap on value moved per period that refills over time.
- **Default configuration:** protocol-owned settings an app inherits when it does not set its own.
