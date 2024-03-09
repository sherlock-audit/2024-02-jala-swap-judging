Agreeable Tiger Hippo

medium

# The burning of small liquidity will be revert

## Summary

The burning of small liquidity will be revert

## Vulnerability Detail

In the function `burn`, users burn liquidity in exchange for the corresponding token0 and token1. However, the zero check for amount0 and amount1 will prevent users from burning small liquidity.

```solidity
uint256 _totalSupply = totalSupply; 
amount0 = (liquidity * balance0) / _totalSupply; 
amount1 = (liquidity * balance1) / _totalSupply; 
if (amount0 == 0 || amount1 == 0) revert InsufficientLiquidityBurned();
```

For example:

Alice uses 1e7\*1e18 token0 and 1e1\*1e18 token1 to create pool. So the totalSupply of pool will be 1e4\*1e18. (reserve0 = 1e7\*1e18, reserve1 = 1e1\*1e18)

Then Bob mints liquidity using 1e7 token0 and 1e1 token1. So the Bob’s liquidity will be 1e4.

Finally, if Bob wants to burn liquidity which is smaller than 1000, his request will be reverted as the amount1 is 0. However, the amount0 is 999000.

## POC

Please place the following test function in the file JalaMasterRouter.t.sol.

```solidity
function test_RemoveSmallLiquidity() public {
    tokenA.approve(address(masterRouter), 10000000);
    tokenB.approve(address(masterRouter), 10);

    masterRouter.wrapTokensAndaddLiquidity(
        address(tokenA),
        address(tokenB),
        10000000,
        10,
        10000000,
        10,
        user0,
        block.timestamp
    );

    address wrappedTokenA = wrapperFactory.wrappedTokenFor(address(tokenA));
    address wrappedTokenB = wrapperFactory.wrappedTokenFor(address(tokenB));
    address pairAddress = factory.getPair(address(wrappedTokenA), address(wrappedTokenB));
    JalaPair pair = JalaPair(pairAddress);
    uint256 liquidity = pair.balanceOf(user0);

    uint256 smallLP = 1000 - 1;

    vm.startPrank(user0);
    pair.approve(address(masterRouter), smallLP);

    masterRouter.removeLiquidityAndUnwrapToken(
        address(tokenA),
        address(tokenB),
        smallLP,
        0,
        0,
        user0,
        block.timestamp
    );
    vm.stopPrank();
}
```

Finally, the function removeLiquidityAndUnwrapToken will be reverted.

![Untitled](https://github.com/sherlock-audit/2024-02-jala-swap-zrax-x/assets/52646245/aafa11fe-7079-41c9-9713-8f3e9c136ada)


## Impact

User can’t burn small liquidity, which may causes fund locks. Please note that users may have a small amount of liquidity remaining after multiple burns.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L197

## Tool used

Manual Review

## Recommendation

Change OR logic to AND logic.

```solidity
if (amount0 == 0 && amount1 == 0) revert InsufficientLiquidityBurned();
```