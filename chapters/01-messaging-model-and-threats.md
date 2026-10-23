# Cross-Chain Messaging: Model and Threats

A cross-chain messaging protocol lets a contract on one chain cause an action on another. Token bridges are the best-known use, but the same machinery carries governance votes, oracle updates, and arbitrary application calls. Every protocol in this book answers one question differently: **how does the destination chain come to believe that something happened on the source chain?** The answer decides what an attacker must compromise to forge a message.

This book compares LayerZero, Chainlink CCIP, Hyperlane, Wormhole and Axelar as their public documentation describes them. These designs change often. Treat every specific claim here as a starting point and check the current documentation and deployed contracts before relying on it.

## The generic pipeline

```mermaid
flowchart LR
  A[Source app] -->|send(dst, payload)| SE[Source endpoint / gateway]
  SE -->|event| OFF[Off-chain verifiers & relayers]
  OFF -->|attestation / proof| DE[Destination endpoint / gateway]
  DE -->|verify| V[Verification module]
  DE -->|deliver| B[Destination app]
  B -->|checks source chain + sender| B
```

Five roles appear in nearly every design, though names differ:

| Role | Job | If it misbehaves |
|---|---|---|
| Source endpoint | Assigns nonce, emits message | Bugs in nonce or encoding |
| Verifier(s) | Attest that the message was emitted and final | Forged messages accepted |
| Relayer / executor | Carries data and pays gas on destination | Delay or censorship, but not forgery if verification is sound |
| Destination endpoint | Checks attestation, enforces ordering and replay protection | Replay, wrong ordering |
| Receiving application | Checks message origin and acts on payload | Accepts messages from anyone |

The split between **verification** (is this message real?) and **execution** (deliver it and run the app logic) is the most useful mental model. Verification is where forgery happens. Execution is where liveness and ordering problems happen.

## Verification approaches

| Approach | What you trust | Examples (per public docs) |
|---|---|---|
| External committee / multisig | A threshold of known signers | Wormhole Guardians; Hyperlane multisig ISMs |
| Proof-of-stake validator set | A stake-weighted majority of a chain's validators | Axelar |
| Configurable verifier sets | Whichever verifiers the app chooses, at a threshold | LayerZero DVNs; Hyperlane aggregation ISMs |
| Committee plus independent veto network | Two independently run networks | Chainlink CCIP (DONs plus the Risk Management Network) |
| Light client / ZK proof | Source chain consensus, checked in a contract or circuit | Some ZK light-client verifiers, usable as LayerZero DVNs or Hyperlane ISMs |
| Optimistic | At least one honest watcher during a challenge window | Optimistic designs (for example, Nomad's original design) |

Chapter 2 goes protocol by protocol.

## Threat model

| Threat | Typical cause | Impact |
|---|---|---|
| Forged message | Verifier key compromise, verification bug, misconfigured threshold | Unlimited minting or arbitrary calls on destination |
| Replay | Missing nonce or message-hash tracking; same app on several chains | Double mints, repeated actions |
| Unauthorised sender | Receiver does not check source chain and sender | Anyone can trigger privileged logic |
| Configuration drift | App relies on defaults that the protocol owner can change | Security silently weakens |
| Reorg on source | Message attested before finality | Destination acts on a message that no longer exists on source |
| Liveness failure | Relayer, executor or verifier offline; message reverts on destination | Funds stuck; ordered channels blocked |
| Admin compromise | Protocol or app upgrade keys | Everything above |

Historical bridge losses show these categories. Public post-mortems describe the Wormhole Solana exploit in February 2022 (around US$320M) as a signature-verification bypass, the Nomad bridge in August 2022 (around US$190M) as an initialisation bug that made unproven messages acceptable, and the Ronin bridge in March 2022 (over US$600M) as a compromise of validator keys. See **cross-chain-bridge-security** for bridge-specific detail. This book focuses on the messaging layer and on how applications integrate it.

## Shared responsibility

Messaging protocols secure delivery of bytes. Applications must still:

1. Restrict who can call their receive function (usually only the endpoint).
2. Check the source chain identifier and source sender against an allowlist.
3. Choose and pin a security configuration rather than inheriting defaults blindly.
4. Handle failed, delayed and out-of-order messages.
5. Bound the damage of a forged message with rate limits.

Most application-level incidents come from steps 1 to 3.

## Worked example

_Not executed in this book's CI; verify locally._

Trace a message through a protocol's test environment on a local machine.

1. Install Foundry and clone LayerZero's `devtools` repository, which includes OApp examples and a test helper that deploys mock endpoints for several chain IDs in one Foundry test.
2. Run the example OApp tests with `forge test` in the example directory (follow the repo README for setup).
3. Add `-vvvv` and read the trace: find the `send` call, the packet emission, the mock verification step, and `lzReceive` on the destination OApp.
4. Write down which contract enforces each of the five roles in the table above.
5. Repeat with Hyperlane's Foundry test utilities (`MockMailbox` in the Hyperlane monorepo) and compare the traces.

## Common pitfalls

- Assuming the messaging protocol checks who sent the message on behalf of your app.
- Treating "decentralised" in marketing as a precise trust model.
- Ignoring source-chain finality when choosing confirmation settings.
- Relying on old blog posts; verification designs and defaults change.
- Mixing up relayer liveness risk with verifier forgery risk.

## Exercises

1. For one protocol, write the five-role table with the real contract or service name for each role, citing current docs.
2. Pick one historical bridge incident from DeFiHackLabs and classify its root cause using the threat table.
3. Draw the pipeline diagram for a governance message that upgrades a contract on another chain, marking where a forged message would be most damaging.

## Further reading

- LayerZero v2 documentation: <https://docs.layerzero.network>
- Chainlink CCIP documentation: <https://docs.chain.link/ccip>
- Hyperlane documentation: <https://docs.hyperlane.xyz>
- Wormhole documentation: <https://wormhole.com/docs>
- Axelar documentation: <https://docs.axelar.dev>
- Zamyatin et al., "SoK: Communication Across Distributed Ledgers", Financial Cryptography 2021
