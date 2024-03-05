Sweet Hickory Shetland

high

# Swapping Fan tokens through JalaMasterRouter leads to tokens being lost

## Summary
[Fan tokens on chilliz chain](https://docs.chiliz.com/chiliz-chain-mainnet/fan-tokens) have 0 decimals, which is one of the reasons `JalaMasterRouter` was created. It wraps the tokens in 18 decimals before interacting with the pool and then unwraps them back to the underlying Fan token before sending them back to the user. Problem is that because of the swap fee and rounding down during unwrapping the user receives either nothing or much less than expected

## Vulnerability Detail
This is how the  `JalaMasterRouter::swapExactTokensForTokens` function looks:

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L191

```solidity
 function swapExactTokensForTokens(
        address originTokenAddress,
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external virtual override returns (uint256[] memory amounts, address reminderTokenAddress, uint256 reminder) {
        ...... 

       // 1. Wrap token to 18 decimals
        _approveAndWrap(originTokenAddress, amountIn);

       ..... 
       
      // 2. Execute Swap With the wrapped token
        amounts = IJalaRouter02(router).swapExactTokensForTokens(
            IERC20(wrappedTokenIn).balanceOf(address(this)),
            amountOutMin,
            path,
            address(this),
            deadline
        );

    // 3. Unwrap the received token and send it
        (reminderTokenAddress, reminder) = _unwrapAndTransfer(path[path.length - 1], to);
    }
```
Here is a short example of how wrapping /unwrapping affects token decimals:
- `token` => `1e6` (6 decimals)
- `wrap(token)` => `wrapped_token` = `1e18` (adds 18 - `token.decimals()` decimals)
- `unwrap(wrapped_token)` => `token` = `1e6` (removes 18 - `token.decimals()` decimals)

The problem here arises for the so called `fan` tokens, which are the most used ERC20 tokens on the chilliz chain -> https://chiliscan.com/tokens.

The specific thing about them is that they have `0` decimals, which means that 1 whole token is just `1` without any zeros after it. This characteristic makes them especially sensitive to rounding, which happens on every swap.

The issue is more evident when only a single token is being swapped. For example if a user decides to swap 1 `Juventus` token for 1 `Barcelona` token he will receive 0 `Barcelona` tokens. This happens because swap fee makes the `amountOut` round down to zero during unwrapping

Normally `1` is considered a dust amount, but in this situation it is significant, because the token has no decimals, meaning that 1 is not some very small % of the whole token, but represent 100% of the token value.

[Also a quick research on chilliz explorer](https://chiliscan.com/tokens) uncovers that `fan` token transfers with amount of `1` constitute about 50% of all transfers, which means this will be quite a common scenario

Depending on the market conditions, the prices per token of some fan tokens may go beyond `50$`, which is not an insignificant amount. Historical examples:
- [FC Barcelona fan token -> ](https://coinmarketcap.com/currencies/fc-barcelona-fan-token/) Mar 2021
-  [Atletico Madrid ->](https://coinmarketcap.com/currencies/atletico-de-madrid-fan-token/)  May 2021

There is another problem attached to this scenario -> no slippage protection can be provided, because any value below 1e18 (1 token wrapped) will effectively be 0 and if the user supplies(1e18) the swap reverts, because the fee cannot be covered.



## Impact
- users loose tokens on single token swaps for fan tokens
- no proper slippage protection can be used for fan tokens
- users  experience losses for bigger swaps

## Code Snippet

A step by step explanation of the bug, complemented with a coded example:
- Bob decides to swap 1 fan token `A` for 1 fan token `B`
- Bob calls  `JalaMasterRouter::swapExactTokensForTokens` with `amountIn` as `1`
- The router wrapps the token to `wrappedA` which is `18 decimals` => `1e18`
- The router calls the pool to execute the swap
- The pool deducts a fee (`3%`) and executes the swap with the rest of the amount
- The final amount of `wrappedB` received after the swap is `9.92e17`  (e.g less than 1e18 - one whole token)
- The router unwraps  `wrappedB` =>  `9.92e17` / `1e18` = 0 - bob receives nothing
- Another problem is that no proper slippage can be provided as well - providing anything less than 1e18 (1 token) is equivalent to 0 as seen from the above calculation, which means slippage should be `1e18` (1 token for at least 1 token), which fails the swap, because after deducing the swap fee the received amount is immediately 0

Put this test inside `JalaMasterRouter.t.sol` and run ` forge test --contracts test/JalaMasterRouter.t.sol 
--mt test_SwapExactTokensForTokens_Loss -vvvv`

```solidity
function test_SwapExactTokensForTokens_Loss() public {
        address wrappedTokenA = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(address(tokenA));
        address wrappedTokenB = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(address(tokenB));

        //FAN tokens have 0 decimals
        assertEq(tokenA.decimals(),0);

        // Bob wants to trade his FAN token
        address _bob = address(101);

        tokenA.approve(address(masterRouter), 10 ether);
        tokenB.approve(address(masterRouter), 10 ether);

        // Transfer bob 100 A-FAN tokens
        tokenA.transfer(_bob, 100);

        // Provide 200 FAN tokens in liquidity to pool
        uint256 liquidityIn=200;

         masterRouter.wrapTokensAndaddLiquidity(
            address(tokenA),
            address(tokenB),
            liquidityIn,
            liquidityIn,
            liquidityIn,
            liquidityIn,
            address(this),
            block.timestamp
        );

         address pairAddress = factory.getPair(
            wrapperFactory.wrappedTokenFor(address(tokenA)),
            wrapperFactory.wrappedTokenFor(address(tokenB))
        );

        // Make sure pool is balanced
        (uint112 _reserve0, uint112 _reserve1, ) = JalaPair(pairAddress).getReserves();

        assertEq(_reserve0,_reserve1);
        assertEq(_reserve0,liquidityIn * IChilizWrappedERC20(wrappedTokenA).getDecimalsOffset());
        assertEq(_reserve1,liquidityIn * IChilizWrappedERC20(wrappedTokenB).getDecimalsOffset());

        // Bob swaps 1 A-FAN token for B-FAN token
        vm.startPrank(_bob);

        tokenA.approve(address(masterRouter), 100);

        address[] memory path = new address[](2);
        path[0] = wrappedTokenA;
        path[1] = wrappedTokenB;

        // Slippage does not work
        // Bob wants to receive at least 1 token => 1e18
        uint256 minTokenReceived = 1e18; // 1 wrapped token is 1 FAN token
        vm.expectRevert(); //InsufficientOutputAmount()
        masterRouter.swapExactTokensForTokens(address(tokenA), 1, minTokenReceived, path, _bob, block.timestamp);

        // Bob must execute the swap without any slippage
       (uint256[] memory amounts, address reminderTokenAddress, uint256 reminder)=  masterRouter.swapExactTokensForTokens(address(tokenA), 1, 0, path, _bob, block.timestamp);
        vm.stopPrank();

        // Bob send 1 A-FAN token
        assertEq(tokenA.balanceOf(_bob),99);
        // BUT received 0 B-FAN tokens
        assertEq(tokenB.balanceOf(_bob),0);
     }

```

## Tool used
Manual review / Foundry

## Recommendation

In my opinion the standard swap mechanism forked from Uniswap V2 is not suited to work well for `fan` tokens, because of everything described above.

One possible solution I can think of is to add a new method for swapping such tokens, where the fee is deducted not from the tokens themselves, but some additional token that is used exclusively to pay the swap fee (`JALA`, or the native `CHZ ` as easiest solution). This way the `fan` tokens will stay intact and there would be no losses
