# Rate Limits, Monitoring and Incident Response

No verification model is perfect. Keys leak, contracts have bugs, and configurations drift. The question for an integrator is how much can be lost before someone notices and stops it. Rate limits cap the loss per unit of time. Monitoring shortens the time to notice. An incident plan shortens the time to stop. This chapter covers all three for cross-chain applications.

## Rate limits by protocol

| Layer | Mechanism (per public docs) | Who configures |
|---|---|---|
| Chainlink CCIP token pools | Token-bucket rate limits per lane, inbound and outbound (capacity and refill rate) | Token admin for cross-chain tokens; Chainlink for some pools |
| LayerZero OFT | Optional rate limiter contracts in LayerZero's OApp/OFT tooling; apps can add their own | App owner |
| Hyperlane warp routes | App-level; rate-limited variants and hooks exist in the monorepo | App owner |
| Wormhole Governor | Per-chain notional outflow limits enforced by Guardians, with delays for large transfers | Wormhole Guardians |
| Wormhole NTT | Per-token inbound and outbound rate limits with queuing | Token owner |
| Axelar Interchain Token Service | Flow limits per token per epoch | Token operator |

Even where a protocol offers limits, the application should consider its own, close to the asset: a mint cap per period on the destination token contract bounds damage from any messaging failure, including one in the limiter itself.

## Designing a limit

A token bucket is the usual shape:

```solidity
struct Bucket { uint128 tokens; uint128 capacity; uint64 lastRefill; uint128 ratePerSec; }

function _consume(Bucket storage b, uint256 amt) internal {
    uint256 elapsed = block.timestamp - b.lastRefill;
    uint256 refill = elapsed * b.ratePerSec;
    uint256 available = b.tokens + refill > b.capacity ? b.capacity : b.tokens + refill;
    require(amt <= available, "rate limited");
    b.tokens = uint128(available - amt);
    b.lastRefill = uint64(block.timestamp);
}
```

Choices to document:

- **Capacity** sized to normal peak flow, not to total supply. If a forged message can mint the whole bucket, the bucket is the maximum loss per refill period.
- **Inbound and outbound** limits separately. An attacker who forges inbound mints then bridges out to a chain with deeper liquidity; outbound limits on that path slow the exit.
- **What happens at the limit**: revert (user retries later) or queue (message waits, then executes). Queuing is friendlier but needs a cancel path.
- **Who can change limits**, and whether a limit can be raised instantly during an attack.

## Monitoring

Things to watch on every chain where the app lives:

| Signal | Why it matters |
|---|---|
| Destination mints or releases without a matching source event | Forged message or accounting bug |
| Sum of supply across chains versus locked collateral | Detects unbacked minting |
| Configuration changes (peers, DVNs, ISMs, owners, delegates, limits) | Config drift or takeover |
| Rate limiter utilisation near capacity | Attack in progress or legitimate surge |
| Messages verified but not executed for a long time | Liveness failure; stuck user funds |
| Protocol-level alerts (CCIP RMN curses, Wormhole Governor delays) | Upstream incident |

A simple cross-chain supply check, run every few minutes:

```python
# supply_check.py: total minted on destinations must not exceed locked on source
from web3 import Web3
# configure: SRC_RPC, SRC_TOKEN, LOCKBOX, DESTINATIONS = [(rpc_url, token_address), ...]
ERC20 = [{"name":"totalSupply","type":"function","stateMutability":"view","inputs":[],"outputs":[{"type":"uint256"}]},
         {"name":"balanceOf","type":"function","stateMutability":"view","inputs":[{"type":"address"}],"outputs":[{"type":"uint256"}]}]
def call(rpc, token, fn, *a):
    c = Web3(Web3.HTTPProvider(rpc)).eth.contract(address=token, abi=ERC20)
    return getattr(c.functions, fn)(*a).call()

locked = call(SRC_RPC, SRC_TOKEN, "balanceOf", LOCKBOX)
minted = sum(call(rpc, tok, "totalSupply") for rpc, tok in DESTINATIONS)
if minted > locked:
    print(f"ALERT: minted {minted} > locked {locked}")   # page on-call
```

Account for in-flight messages to avoid false alarms, and alert on the trend as well as the threshold.

## Incident response

Write the plan before you need it.

1. **Detection to decision.** Who is paged, and who can decide to pause? Minutes matter; a cross-chain exploit often drains in a handful of transactions.
2. **Pause levers.** List every lever and who holds it: app pause on each chain, rate limit to zero, unset peers or routers, protocol-level halts (only the protocol can trigger these; know how to reach them).
3. **Order of operations.** Usually: stop minting on destinations, stop releases on source, then investigate. Pausing only one chain can leave the attacker an exit on another.
4. **Communication.** Contact the messaging protocol's security team, affected DEXs and bridges, and stablecoin issuers who may freeze stolen funds.
5. **Recovery.** Reconcile supply, decide on reimbursement, fix, and get a re-audit before unpausing.

The Security Alliance's SEAL 911 is a public emergency contact channel for active incidents, and its safe harbor framework sets terms for whitehat rescue. Many protocols also run bug bounties on Immunefi. Adopt both before an incident.

## Worked example

_Not executed in this book's CI; verify locally._

Test a rate-limited destination minter with forged messages on a local chain.

1. Reuse the toy receiver from chapter 3 or 4 and add the token bucket above to its mint path, with capacity 100,000 tokens and refill of 100,000 per day.
2. Write a test that simulates a forged-message attacker who can deliver unlimited messages: loop deliveries until revert and assert total minted ≤ capacity.
3. `vm.warp` 12 hours and repeat; assert the additional mint ≤ half the capacity.
4. Add a `pause()` held by a guardian address and a test where the guardian pauses after the first forged mint; assert later deliveries revert.
5. Run `supply_check.py` against two Anvil instances (source and destination) with a deliberately forged mint and confirm the alert fires.

## Common pitfalls

- Rate limits sized to total supply, which bound nothing.
- Only inbound limits, leaving a fast exit path.
- Limit setters that can raise caps instantly, from a single key.
- Monitoring that ignores configuration changes.
- Pausing one chain while the attacker exits on another.

## Exercises

1. Configure a CCIP token pool rate limit in a `chainlink-local` simulation and test a burst that exceeds it.
2. Extend `supply_check.py` to track in-flight messages from source events and reduce false positives.
3. Write a one-page incident runbook for "forged inbound mint on chain B", naming each lever, holder and the order of actions.

## Further reading

- Chainlink CCIP rate limits docs: <https://docs.chain.link/ccip>
- Wormhole Governor and NTT docs: <https://wormhole.com/docs>
- Axelar Interchain Token Service docs: <https://docs.axelar.dev>
- Security Alliance (SEAL 911, safe harbor): <https://securityalliance.org>
- **cross-chain-bridge-security** in this wiki
