# Integration Audit Checklist

This checklist is for auditing an application that uses LayerZero, Chainlink CCIP, Hyperlane, Wormhole, Axelar or a similar protocol. It assumes the messaging protocol itself is out of scope, but records the trust you place in it. Each item names how to verify it. Because these protocols change, start every audit by recording the protocol versions, library addresses and documentation dates you checked against.

## Scoping

1. Which protocol(s), which versions, and which chains (with protocol chain identifiers)?
2. Which contracts send and receive messages, and what each message type does.
3. Which verification configuration applies on each pathway (chapter 2), read from chain, not from docs.
4. Who owns configuration: owners, delegates, admins, and their key types.
5. What assets move, and the maximum value at risk per message and per day.

## Verification and configuration

| # | Check | How to verify |
|---|---|---|
| V1 | Security config pinned explicitly (LayerZero DVNs, Hyperlane ISM), not inherited from defaults | On-chain reads (chapter 2); `config_diff.py` |
| V2 | Verifier set has at least two independent parties where the design allows | Identify DVN/ISM operators; record independence |
| V3 | Confirmations appropriate for source-chain finality | Compare config to chain finality guidance |
| V4 | Config owners/delegates are multisigs with timelocks | `cast call` owner/delegate; inspect Safe threshold |
| V5 | Peers/routers/trusted remotes set on every chain, symmetric, and pointing at current deployments | Script across all chains |
| V6 | Protocol-level trust documented (CCIP DON+RMN, Wormhole Guardians, Axelar validators) | Written trust statement in the report |

## Receive path

| # | Check | How to verify |
|---|---|---|
| R1 | Only endpoint/router/mailbox/gateway can call receive | Unit test calling from another address |
| R2 | Source chain and sender checked against allowlist | Test with a wrong chain and wrong sender (chapter 3) |
| R3 | Address encodings compared correctly (padding, case, non-EVM) | Tests with differently encoded equal and unequal addresses |
| R4 | Replay protection at the right layer; VAAs or signatures marked consumed | Duplicate-delivery test (chapter 4) |
| R5 | Payload binds destination chain and contract when the app is multi-deployed | Cross-deployment replay test |
| R6 | Payload decoding validated; no unchecked lengths or types | Fuzz `_lzReceive` / `_ccipReceive` / `handle` with random bytes |

## Execution and liveness

| # | Check | How to verify |
|---|---|---|
| E1 | Ordering assumptions documented and enforced | Out-of-order delivery test |
| E2 | Failed messages retryable without privileged keys | Revert-then-retry test |
| E3 | No catch-all that silently drops value | Code review; test with reverting handler |
| E4 | Gas limits have safe minimums; users cannot grief refunds | Test with minimum gas |
| E5 | Refund path for permanently failed transfers | Scenario test |
| E6 | Nonce skip/clear/burn functions restricted and logged | Access-control test |

## Damage limits and operations

| # | Check | How to verify |
|---|---|---|
| D1 | Inbound and outbound rate limits sized to normal flow | Read config; burst test (chapter 5) |
| D2 | Destination token has a mint cap independent of the messaging layer | Read token code |
| D3 | Pause levers on every chain, with documented holders | Role table; pause test |
| D4 | Cross-chain supply monitor and config-change alerts | Alert rules and a test alert |
| D5 | Incident runbook, protocol security contacts, bug bounty, safe harbor | Documents; Immunefi or SEAL safe harbor adoption |

## Using AI assistants

An LLM can be useful for enumerating message types, decoding config bytes, or drafting the per-chain peer script. Treat each output as a proposal. Configuration claims are checked by on-chain reads; code claims are checked by tests or Slither output you have traced. If a model states "the DVN threshold is 2 of 3", the `getConfig` read decides. Note in the report which checks were tool-verified. See **ai-assisted-audit-playbook**.

## Worked example

_Not executed in this book's CI; verify locally._

Produce a configuration evidence table for one real application from a local fork, reading only.

1. Choose an application deployed with LayerZero v2 on two chains. Start two forks: `anvil --fork-url $RPC_A --port 8545` and `anvil --fork-url $RPC_B --port 8546`.
2. On each fork, read `peers(eid)` for the other chain, the receive library, and the ULN config with `cast call` (chapter 2). Decode with your Python script.
3. Read the OApp's owner and the endpoint delegate; check whether each is a contract and, if a Safe, read `getThreshold()` and `getOwners()`.
4. Fill rows V1, V4 and V5 with the values and the exact commands used.
5. Repeat for a Hyperlane warp route (ISM type, validators and threshold) if the app uses one.
6. Attach the commands and outputs to the report so the client can rerun them.

## Common pitfalls

- Auditing receive code while skipping on-chain configuration.
- Reading docs instead of deployed config.
- Checking one chain of a multi-chain deployment.
- Treating rate limits and pause plans as out of scope.
- Copying AI-generated config claims into a report without on-chain reads.

## Exercises

1. Apply sections V and R to an open-source OApp or warp route and record evidence for each row.
2. Turn R1, R2 and R4 into a reusable Foundry test base contract that takes the app's receive function as a parameter.
3. Write a short trust statement for a CCIP-based app explaining what an attacker must compromise to forge a message, citing current CCIP docs.

## Further reading

- LayerZero v2 docs: <https://docs.layerzero.network>
- Chainlink CCIP docs: <https://docs.chain.link/ccip>
- Hyperlane docs: <https://docs.hyperlane.xyz>
- Wormhole docs: <https://wormhole.com/docs>
- Axelar docs: <https://docs.axelar.dev>
- Immunefi: <https://immunefi.com>
