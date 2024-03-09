Quiet Cornflower Sheep

medium

# Price Manipulation Due to Initial Liquidity Addition with Zero Reserves

## Summary
Price Manipulation Due to Initial Liquidity Addition with Zero Reserves in `_addLiquidity` function.

## Vulnerability Detail
When both reserveA and reserveB are zero, the function allows any amountADesired and amountBDesired to be added to the liquidity pool. This scenario occurs only when the liquidity pair is first created and before any liquidity has been added. 

```solidity
if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
        } else {
            uint256 amountBOptimal = JalaLibrary.quote(amountADesired, reserveA, reserveB);
            if (amountBOptimal <= amountBDesired) {
                if (amountBOptimal < amountBMin) revert InsufficientBAmount();
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint256 amountAOptimal = JalaLibrary.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                if (amountAOptimal < amountAMin) revert InsufficientAAmount();
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
```
A malicious actor could exploit this by setting an arbitrary price ratio that does not reflect the market rate, potentially allowing them to purchase large amounts of one token at an unfairly low price once trading begins.

## Impact
There are no checks on the ratio of `amountADesired` to `amountBDesired`, which could lead to a highly skewed pool that does not accurately reflect market conditions or that could be easily manipulated.

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L31

## Tool used

Manual Review

## Recommendation
1. Implement a mechanism to ensure the initial price ratio reflects a fair market rate. This could involve oracle usage to get a reference price or requiring the first liquidity provider to supply liquidity within a certain range of a predefined price.

2. Require a minimum amount of liquidity to be added when creating a new pair to prevent the pool from being easily manipulated with very small amounts of tokens.