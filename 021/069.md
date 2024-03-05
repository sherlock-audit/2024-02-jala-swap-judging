Bent Navy Swallow

medium

# M-1: JalaSwap createPair abi.encodePacked Allows Hash Collision in createPair

## Summary
The `createPair` function in the JalaSwap contract utilizes `abi.encodePacked(token0, token1)` as a salt for the `create2` function, potentially leading to hash collisions. This vulnerability could allow attackers to manipulate the hash value, resulting in unexpected behavior and potential security risks.

## Vulnerability Detail
The vulnerability arises in the `createPair` function, specifically in the following code block:

```solidity
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
```

As indicated in the Solidity documentation: 

https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode 

 the usage of `abi.encodePacked` can lead to hash collisions. Since these values are user-specified function arguments in an external function, an attacker can potentially manipulate these arguments to produce the same `encodePacked` value, bypassing certain validations.

## Impact
The vulnerability allows an attacker to manipulate the hash value used in the `create2` function, potentially leading to the creation of pairs with the same hash but different token addresses. This could result in unexpected behavior and potential security risks.

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaFactory.sol#L48

## Tool used

Manual Review

## Recommendation
To mitigate this vulnerability, it is recommended to use  `abi.encode()`  instead .