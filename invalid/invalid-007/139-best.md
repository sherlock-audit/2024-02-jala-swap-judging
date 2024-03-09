Fancy Cloth Robin

medium

# Attacker can frontrun creation of pool by calling `createPair` directly from JalaFactory

## Summary
Attacker can create LP pair directly through JalaFactory, which could assign the LP with an unexpected address and cause issues for 3rd party protocols integrating with JalaSwap.

## Vulnerability Detail
JalaPair (LP) addresses are designed to be deterministic due to the use of `CREATE2` in JalaFactory. A defi protocol that wants to create a Jala LP may have an expectation of what the LP address will be and use/declare it in their own swap contracts even before the LP is deployed.

`CREATE2` addresses are a function of a) `0xFF` a constant, b) sender's address (this is important), c) salt and d) to-be-deployed bytecode.

The defi protocol would expect that sender's address in this case should be JalaRouter02 contract since LP's are created in the `addLiquidity` function when liquidity is added for the first time.

```solidity
// JalaRouter02.sol
function _addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountADesired,
        uint256 amountBDesired,
        uint256 amountAMin,
        uint256 amountBMin
    ) internal virtual returns (uint256 amountA, uint256 amountB) {
        // create the pair if it doesn't exist yet
        if (IJalaFactory(factory).getPair(tokenA, tokenB) == address(0)) {
            IJalaFactory(factory).createPair(tokenA, tokenB);
        }
        
    ...
```

However, an attacker could easily bypass the router contract and call `createPair` directly on JalaFactory. Sender in this case would be the attacker's address (instead of router), resulting in a different address from the one that the Defi protocol expected. However, the Defi protocol now cannot deploy a new LP with the same token pair since it has already been created and pushed into the `allPairs` mapping in the factory contract.

## Impact
External protocols may face unexpected issues and difficulty when integrating with Jalaswap. 

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaFactory.sol#L42
## Tool used
Manual Review

## Recommendation
In Factory.sol, only allow JalaRouter to call `createPair` to ensure that addresses of LPs remain deterministic and predictable.
