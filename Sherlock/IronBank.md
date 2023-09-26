# [M-01] No secondary price oracle and no check for L2 sequencer uptime feed and grace period

## Summary

Due to the way lending protocols work, price of an asset is extremely important. However relying on only one oracle can cause problems if the oracle goes down. Also the L2 sequencer could go offline causing problems for borrowers.

## Vulnerability Detail

If Chainlink oracle goes down the protocol does not have a second backup price oracle to fallback to.
If the L2 sequencer goes offline and after a while it goes back online, the oracle will update the price and all price movements that occurred during downtime are applied at once. If these movements are significant, they may cause chaos. Borrowers would rush to save their positions, while liquidators would rush to liquidate borrowers. Since liquidations are handled mainly by bots, borrowers are likely to suffer mass liquidations.

## Impact

This could cause a liquidation of certain users when they were not supposed to be liquidated.

## Code Snippet

[PriceOracle.sol](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L1)

## Tool used

Manual Review

## Recommendation

Consider [implementing circuit breakers](https://0xmacro.com/blog/how-to-consume-chainlink-price-feeds-safely/), an additional second price oracle, either Uniswap TWAP or another off-chain oracle like Tellor. And correctly implement the L2 sequencer. All recommendations can be found in the link provided.

# [M-02] Chainlink oracle return values are not handled properly

## Summary

Chainlink oracle return values are not handled properly in multiple instances. This can cause stale prices.

## Vulnerability Detail

The `registry.latestRoundData()` will return `uint80 roundID`, `int256 price`, `uint256 startedAt`, `uint256 timeStamp`, `uint80 answeredInRound`. These return values are meant to be used to do some extra checks before updating the price. By just receiving the price, you can get stale prices and incomplete rounds.

## Impact

If there is a problem with Chainlink starting a new round and finding consensus on the new value for the oracle (e.g. Chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) consumers of this contract may continue using outdated stale or incorrect data (if oracles are unable to submit no new round is started).

## Code Snippet

- [PriceOracle.sol#L67](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L67)

```solidity
(, int256 price,,,) = registry.latestRoundData(base, quote);
```

- [PriceOracle.sol#L107](https://github.com/sherlock-audit/2023-05-ironbank/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L107)

```solidity
(, int256 price,,,) = registry.latestRoundData(aggrs[i].base, aggrs[i].quote);
```

## Tool used

Manual Review

## Recommendation

Use this code to get all the values and sanitize the returned values.

```solidity
(uint80 roundID, int256 price, , uint256 timeStamp, uint80 answeredInRound) = registry.latestRoundData();
require(price > 0, "Chainlink price <= 0");
require(answeredInRound >= roundID , "Stale price");
require(timeStamp != 0, "Round not complete");
```