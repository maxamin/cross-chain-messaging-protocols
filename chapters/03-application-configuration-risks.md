# Application Configuration Risks

The messaging protocol may be sound and the application still exposed. Most integration bugs live in a handful of lines: who may call the receive function, which source chains and senders are trusted, and which security configuration the app has actually pinned. This chapter goes through each protocol's receive pattern and the configuration mistakes auditors find most often.

## The three checks every receiver needs

1. **Caller check.** Only the protocol's endpoint, router, mailbox or gateway may call the receive function.
2. **Origin check.** The source chain identifier and source sender must match an allowlist ("peer", "trusted remote", "router").
3. **Payload check.** The decoded payload must be validated like any untrusted input.

Base contracts from each protocol do the first and often the second for you, if you use them as intended:

| Protocol | Receive entry | Base contract handles | App must still |
|---|---|---|---|
| LayerZero v2 | `lzReceive` → `_lzReceive(origin, guid, message, executor, extraData)` | Caller is endpoint; `origin.sender` equals `peers[origin.srcEid]` (OApp) | Set peers correctly; pin DVN config |
| LayerZero v1 (legacy) | `lzReceive` → `_blockingLzReceive` | Caller is endpoint; `trustedRemoteLookup[srcChainId]` matches path | Set trusted remotes as packed (remote, local) addresses |
| Chainlink CCIP | `ccipReceive` → `_ccipReceive(Client.Any2EVMMessage)` | Caller is router (`CCIPReceiver`) | Check `sourceChainSelector` and decoded `sender` |
| Hyperlane | `handle(origin, sender, body)` | Caller is mailbox (`MailboxClient` / `Router`) | Check `sender` matches enrolled router; set ISM |
| Wormhole | App-defined; typically receives VAA bytes | Core `parseAndVerifyVM` checks signatures | Check emitter chain and address; mark VAA hash consumed |
| Axelar | `execute` → `_execute(sourceChain, sourceAddress, payload)` | Gateway approval (`AxelarExecutable`) | Check `sourceChain` and `sourceAddress` strings |

## Common misconfigurations

**Missing origin check.** A CCIP receiver that checks only that the router called it accepts messages from any sender on any supported chain.

```solidity
// VULNERABLE: any sender on any source chain can trigger mint
function _ccipReceive(Client.Any2EVMMessage memory m) internal override {
    (address to, uint256 amt) = abi.decode(m.data, (address, uint256));
    token.mint(to, amt);
}

// FIXED
mapping(uint64 => mapping(address => bool)) public allowed;
function _ccipReceive(Client.Any2EVMMessage memory m) internal override {
    address src = abi.decode(m.sender, (address));
    require(allowed[m.sourceChainSelector][src], "untrusted source");
    (address to, uint256 amt) = abi.decode(m.data, (address, uint256));
    token.mint(to, amt);
}
```

**Address encoding mismatches.** Senders are often `bytes` or `bytes32`, and chain IDs are protocol-specific (LayerZero endpoint IDs, CCIP chain selectors, Hyperlane domains, Wormhole chain IDs, Axelar chain name strings). Bugs include comparing a left-padded `bytes32` with a right-padded one, case-sensitive comparison of hex strings (Axelar source addresses are strings), and using EVM chain IDs where protocol IDs are expected. Non-EVM sources (Solana, Move chains) use 32-byte addresses that do not fit in `address`.

**Default security configuration.** On LayerZero and Hyperlane, an app that never sets its DVN config or ISM inherits defaults owned by the protocol. That may be fine, but it means the app's security can change without any app transaction, and the default may be a single verifier on some pathways. Pin configuration explicitly, and set it on both send and receive sides.

**Delegates and owners.** LayerZero OApps have a delegate that can change configuration on the endpoint; Hyperlane routers have owners who can change ISMs and enrolled routers. These keys are as powerful as the verifiers. Put them behind a multisig and timelock.

**Asymmetric peers.** A peer set on chain A for chain B but not the reverse, or set to the wrong contract after a redeploy, causes stuck messages or, worse, trusts an old deployment that someone else now controls.

**Wormhole emitter checks.** Wormhole verifies signatures, not intent. An app that verifies a VAA but skips `emitterChainId` and `emitterAddress` checks accepts any message published by any contract on any chain.

**Confirmations too low.** Several protocols let apps or defaults choose block confirmations. Too few confirmations on a chain with reorgs lets a source transaction be reverted after the destination acted.

## Configuration as code

Treat cross-chain configuration like infrastructure. Keep the intended config in a file, read the deployed config from chain, and diff in CI.

```python
# config_diff.py: compare intended peers with on-chain peers for a LayerZero OApp
import json, sys
from web3 import Web3
intended = json.load(open(sys.argv[1]))   # {"rpc": ..., "oapp": ..., "peers": {"30101": "0x...", ...}}
w3 = Web3(Web3.HTTPProvider(intended["rpc"]))
abi = [{"name":"peers","type":"function","stateMutability":"view",
        "inputs":[{"type":"uint32"}],"outputs":[{"type":"bytes32"}]}]
oapp = w3.eth.contract(address=intended["oapp"], abi=abi)
for eid, want in intended["peers"].items():
    got = "0x" + oapp.functions.peers(int(eid)).call().hex().removeprefix("0x")
    want32 = "0x" + want[2:].lower().rjust(64, "0")
    print(eid, "OK" if got.lower() == want32 else f"MISMATCH got={got}")
```

Extend it to DVN configs, ISM addresses and thresholds.

## Worked example

_Not executed in this book's CI; verify locally._

Exploit and fix a missing origin check in a local Hyperlane setup.

1. Clone the Hyperlane monorepo and use its Solidity `MockMailbox` (or the test utilities in its `solidity` package) in a Foundry project.
2. Deploy two mock mailboxes (domains 1 and 2) and a toy `Minter` on domain 2 that inherits `MailboxClient` and mints in `handle` without checking `sender`.
3. Deploy an attacker contract on domain 1 that dispatches a mint message to `Minter`. Process the message with the mock mailbox and assert the attacker received tokens.
4. Fix: inherit Hyperlane's `Router` (which checks enrolled routers) or add an explicit `require(sender == enrolled[origin])`. Rerun and expect a revert.
5. Add a test that the router owner cannot be changed by a non-owner, and that `setInterchainSecurityModule` is owner-only.

## Common pitfalls

- Checking the caller but not the source chain and sender.
- Comparing addresses across differently padded byte encodings.
- Relying on default DVN config or default ISM without pinning or monitoring.
- Leaving config owners and delegates as single EOAs.
- Forgetting to update peers on every chain after a redeploy.

## Exercises

1. Write the CCIP vulnerable/fixed pair above as Foundry tests using the `CCIPLocalSimulator` from Chainlink's `chainlink-local` package.
2. Extend `config_diff.py` to read LayerZero ULN receive config and flag any pathway with fewer than two required DVNs.
3. Write an Axelar `_execute` receiver that compares `sourceAddress` safely (normalised case) and a test that a differently-cased address is handled as intended.

## Further reading

- LayerZero OApp and configuration docs: <https://docs.layerzero.network>
- Chainlink CCIP receiver best practices: <https://docs.chain.link/ccip>
- Hyperlane Router and ISM docs: <https://docs.hyperlane.xyz>
- Wormhole integration docs: <https://wormhole.com/docs>
- Axelar executable docs: <https://docs.axelar.dev>
