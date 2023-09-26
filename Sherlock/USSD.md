# [H-01] Not using slippage parameter or deadline while swapping on UniswapV3

- **Selected for report**

## Summary

While making a swap on UniswapV3 the caller should use the slippage parameter `amountOutMinimum` and `deadline` parameter to avoid losing funds.

## Vulnerability Detail

[`UniV3SwapInput()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L227-L240) in `USSD` contract does not use the slippage parameter [`amountOutMinimum`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L237) nor [`deadline`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L235).

`amountOutMinimum` is used to specify the minimum amount of tokens the caller wants to be returned from a swap. Using `amountOutMinimum = 0` tells the swap that the caller will accept a minimum amount of 0 output tokens from the swap, opening up the user to a catastrophic loss of funds via [MEV bot sandwich attacks](https://medium.com/coinmonks/defi-sandwich-attack-explain-776f6f43b2fd).

`deadline` lets the caller specify a deadline parameter that enforces a time limit by which the transaction must be executed. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time potentially resulting in a worse price for the user.

## Impact

Loss of funds and not getting the correct amount of tokens in return.

## Code Snippet

- Function [UniV3SwapInput()](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L227-L240)
  - Not using [`amountOutMinimum`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L237)
  - Not using [`deadline`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L235)

## Tool used

Manual Review

## Recommendation

Use parameters `amountOutMinimum` and `deadline` correctly to avoid loss of funds.

# [H-02] Precision loss can cause protocol to sell no collateral and leaving it unable to rebalance to peg

## Summary

In contract `USSDRebalancer` precision loss in [`BuyUSSDSellCollateral()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L109) will cause variable [`amountToSellUnits`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L121) to return 0.

## Vulnerability Detail

`amountToSellUnits` is calculated like:

```solidity
uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
```

It does not take into account that `collateral[i].token` can be `WBTC` which uses 8 decimals. Also `((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18)` will always be 0 because of excessive division by `1e18`. For better understanding here is a POC.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

contract AmountToSellUnitsTest is Test {
    uint256 collateralTokenBalanceDecimals = 1e18; // can be 1e18 (DAI, WETH, WBGL) or 1e8 (WBTC)
    uint256 amountToBuyLeftUSDDecimals = 1e18;
    uint256 collateralvalDecimals = 1e18;

    uint256 collateralTokenBalance = 20000 * collateralTokenBalanceDecimals; // representative of `IERC20Upgradeable(collateral[i].token).balanceOf(USSD)`
    uint256 amountToBuyLeftUSD = 1000 * amountToBuyLeftUSDDecimals;
    uint256 collateralval = 20000 * collateralvalDecimals; // same as collateralTokenBalance if DAI

    function test_PrecisionLoss_AmountToSellUnits() public view {
        // MUST BE: collateralval > amountToBuyLeftUSD
        uint256 amountToSellUnits = collateralTokenBalance * ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18) / 1e18;
        console.log("amountToSellUnits:     ", amountToSellUnits); // OUTPUT: 0
        // console.log("collateralTokenBalance:", collateralTokenBalance);
        // console.log("amountToBuyLeftUSD:    ", amountToBuyLeftUSD);
        // console.log("collateralval:         ", collateralval);
        console.log("middle value:          ", ((amountToBuyLeftUSD * 1e18 / collateralval) / 1e18)); // OUTPUT: 0
    }

    function test_Corrected_AmountToSellUnits() public view {
        // MUST BE: collateralval > amountToBuyLeftUSD
        uint256 amountToSellUnits =
            collateralTokenBalance * (amountToBuyLeftUSD * 1e18 / collateralval) / collateralTokenBalanceDecimals;
        console.log("amountToSellUnits:     ", amountToSellUnits); // OUTPUT: 1000000000000000000000 or 1000 * 1e18
        // console.log("collateralTokenBalance:", collateralTokenBalance);
        // console.log("amountToBuyLeftUSD:    ", amountToBuyLeftUSD);
        // console.log("collateralval:         ", collateralval);
        console.log("middle value:          ", (amountToBuyLeftUSD * 1e18 / collateralval)); // OUTPUT: 50000000000000000 or 0.05 * 1e18
    }
}
```

## Impact

Protocol is unable to rebalance to $1 peg

## Code Snippet

[USSDRebalancer.sol#L121](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L121)

## Tool used

Manual Review

## Recommendation

Change `amountToSellUnits` to:
```solidity
uint256 amountToSellUnits = IERC20Upgradeable(collateral[i].token).balanceOf(USSD) * (amountToBuyLeftUSD * 1e18 / collateralval) / (10**IERC20MetadataUpgradeable(collateral[i].token).decimals());
```

# [H-03] Oracle will return too expensive price of DAI because of excessive precision scaling

## Summary

Oracle will return too expensive price because of excessive precision scaling.

## Vulnerability Detail

In [StableOracleDAI.sol#L52](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L52) `price` is being multiplied by `1e10` -> `uint256(price) * 1e10` assuming that Chainlink price feed returns 8 decimals. However checking the decimals of the oracle address used [DAI/ETH decimals](https://etherscan.io/address/0x773616E4d11A78F511299002da57A0a94577F1f4#readContract#F3) we see that it returns 18 decimals.

## Impact

The price of `DAI` will not be representative of the real price of `DAI` - too expensive. Causing the protocol to pay way more for collateral while rebalancing. [`rebalance()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L92) calls two functions which use `getPriceUSD()` from `StableOracleDAI` contract.

## Code Snippet

[StableOracleDAI.sol#L52](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L52)

## Tool used

Manual Review

## Recommendation

Change `uint256(price) * 1e10` to `uint256(price)` removing `* 1e10`

# [H-04] Oracle address not setup

## Summary

Oracle address is initiated as `0x0`

## Vulnerability Detail

Not properly initiated oracle address will not return any amount when calling `getPriceUSD()`

## Impact

Not properly initiated oracle address will result in returning 0 price from [`getPriceUSD()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L50-L52) in `StableOracleDAI` contract.

## Code Snippet

- [StableOracleDAI.sol#L30](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L30)
- [StableOracleDAI.sol#L44](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#LL44C56-L44C56)

## Tool used

Manual Review

## Recommendation

Initiate the oracle address properly.

# [H-05] Using wrong oracle address will cause the protocol to buy wrong amount of collateral and mint wrong amount of USSD token

## Summary

There are two instances where the wrong oracle address is used which will report wrong price of an asset.

## Vulnerability Detail

Using the wrong oracle address can cause the protocol to buy either too much amount of collateral or too little when calling [`rebalance()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSDRebalancer.sol#L92) from `USSDRebalancer` contract. This also effects [`calculateMint()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#LL171C90-L171C90) which is linked to [`mintForToken()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L151) from USSD contract causing wrong minting of `USSD` tokens to users. As well as [`collateralFactor()`](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/USSD.sol#L189) also from `USSD` contract used for accounting.

## Impact

Minting wrong amount of `USSD` tokens to users. Paying too much or too little for collateral for the protocol.

## Code Snippet

- [StableOracleWBTC.sol#L17](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L17) is using [ETH/USD](https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd) oracle address and not [BTC/USD](https://data.chain.link/ethereum/mainnet/crypto-usd/btc-usd)
- [StableOracleDAI.sol#L28](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L28) is using [WBGL/ETH](https://etherscan.io/address/0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e) oracle address and not [DAI/ETH](https://etherscan.io/address/0x60594a405d53811d3bc4766596efd80fd545a270)

## Tool used

Manual Review

## Recommendation

- In [StableOracleWBTC.sol#L17](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L17) change address from [0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419](https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd) to [0xf4030086522a5beea4988f8ca5b36dbc97bee88c](https://data.chain.link/ethereum/mainnet/crypto-usd/btc-usd)
- In [StableOracleDAI.sol#L28](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L28) change address from [0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e](https://etherscan.io/address/0x982152A6C7f732Ec7C9EA998dDD9Ebde00Dfa16e) to [0x60594a405d53811d3bc4766596efd80fd545a270](https://etherscan.io/address/0x60594a405d53811d3bc4766596efd80fd545a270)

# [M-01] Chainlink oracle return values are not handled properly

## Summary

Chainlink oracle return values are not handled properly in multiple instances.

## Vulnerability Detail

The `priceFeed.latestRoundData()` will return `uint80 roundID`, `int256 price`, `uint256 startedAt`, `uint256 timeStamp`, `uint80 answeredInRound`. These return values are meant to be used to do some extra checks before updating the price. By just receiving the price, you can get stale prices and incomplete rounds.

## Impact

If there is a problem with Chainlink starting a new round and finding consensus on the new value for the oracle (e.g. Chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) consumers of this contract may continue using outdated stale or incorrect data (if oracles are unable to submit no new round is started).

## Code Snippet

- [StableOracleWBTC.sol#L23](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L23)
- [StableOracleWETH.sol#L23](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleWETH.sol#L23)
- [StableOracleDAI.sol#L48](https://github.com/sherlock-audit/2023-05-USSD/blob/main/ussd-contracts/contracts/oracles/StableOracleDAI.sol#L48)

## Tool used

Manual Review

## Recommendation

Use this code to get all the values and sanitize the returned values.

```solidity
(uint80 roundID, int256 price, , uint256 timeStamp, uint80 answeredInRound) = priceFeed.latestRoundData();
require(price > 0, "Chainlink price <= 0");
require(answeredInRound >= roundID , "Stale price");
require(timeStamp != 0, "Round not complete");
```

Also in addition consider [implementing circuit breakers](https://0xmacro.com/blog/how-to-consume-chainlink-price-feeds-safely/)