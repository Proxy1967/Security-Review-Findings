# [M-01] Reorg attack on position creation can be used to steal funds

## Lines of code

- https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/PositionFactory.sol#L30-L34
- https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/PositionFactory.sol#L37-L46

## Vulnerability details

### Impact

Attacker can steal funds via reorg attack if position is funded within a few block of being created

### Proof of concept

[PositionFactory.sol#L30-L34](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/PositionFactory.sol#L30-L34)

```solidity
function clonePosition(address _existing) external returns (address) {
    Position existing = Position(_existing);
    Position clone = Position(createClone(existing.original()));
    return address(clone);
}
```

The `PositionFactory` contract clones positions via `clonePosition` which calls the `createClone` function.

[PositionFactory.sol#L37-L46](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/PositionFactory.sol#L37-L46)

```solidity
function createClone(address target) internal returns (address result) {
    bytes20 targetBytes = bytes20(target);
    assembly {
        let clone := mload(0x40)
        mstore(clone, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
        mstore(add(clone, 0x14), targetBytes)
        mstore(add(clone, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)
        result := create(0, clone, 0x37)
    }
}
```

The `createClone` function uses the `create` opcode. This means the address of the new position is only dependent on the nonce of the factory. An attacker can abuse this to steal funds via reorg.

Example: Imagine a user creates a position and funds it. This now allows an attacker to steal the funds via a reorg attack. They would maliciously insert their own transaction which they would use to create a clone with their own malicious parameters. This contract would have the same address as the legitimate user's did before the reorg attack. Now the users funds are in the attacker's malicious contract which can be taken.

### Tools Used

Manual Review

### Recommended Mitigation Steps

Deploy the position contract via the `create2` opcode which uses `salt`.

Example [OpenZeppelin `cloneDeterministic`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Clones.sol#L50) function

```solidity
function cloneDeterministic(address implementation, bytes32 salt) internal returns (address instance) {
    /// @solidity memory-safe-assembly
    assembly {
        // Cleans the upper 96 bits of the `implementation` word, then packs the first 3 bytes
        // of the `implementation` address with the bytecode before the address.
        mstore(0x00, or(shr(0xe8, shl(0x60, implementation)), 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000))
        // Packs the remaining 17 bytes of `implementation` with the bytecode after the address.
        mstore(0x20, or(shl(0x78, implementation), 0x5af43d82803e903d91602b57fd5bf3))
        instance := create2(0, 0x09, 0x37, salt)
    }
    require(instance != address(0), "ERC1167: create2 failed");
}
```