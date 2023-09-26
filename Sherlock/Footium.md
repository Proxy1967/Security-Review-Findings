# [M-01] Use `SafeERC20` functions where applicable

## Summary

Use `safeApprove()`, `safeTransfer()` and `safeTransferFrom()` from OpenZeppelin [SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) instead of regular IERC20 `approve()`, `transfer()` and `transferFrom()` functions.

## Vulnerability Detail

Some ERC20 tokens like USDT require resetting the approval to 0 first before being able to reset it to another value.
Some tokens (like USDT) don't correctly implement the ERC20 standard and their `transfer` `transferFrom` functions return void instead of a success boolean. Calling these functions with the correct ERC20 function signatures will always revert.

## Impact 

Tokens that don't actually perform the transfer and return false are still counted as a correct transfer and tokens that don't correctly implement the latest ERC20 spec, like USDT, will be unusable in the protocol as they revert the transaction because of the missing return value.

## Code Snippet

- [FootiumEscrow.sol#L80](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L80)
- [FootiumEscrow.sol#L110](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L110)
- [FootiumPrizeDistributor.sol#L130](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumPrizeDistributor.sol#L130)

## Tool used

Manual Review

## Recommendation

Use OpenZeppelin's [SafeERC20](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol) with the `safeApprove()`, `safeTransfer()` and `safeTransferFrom()` functions that handle the return value check as well as non-standard-compliant tokens.