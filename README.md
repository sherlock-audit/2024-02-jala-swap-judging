# Issue M-1: The functions about ```permit``` won't work and always revert 

Source: https://github.com/sherlock-audit/2024-02-jala-swap-judging/issues/40 

## Found by 
Aizen, AuditorPraise, PawelK, Spearmint, b0g0, bughuntoor, deth, giraffe, jasonxiale, jokr, recursiveEth, santiellena, zhuying
## Summary
The functions about ```permit``` won't work and always revert
## Vulnerability Detail
```JalaRouter02.sol``` has functions (```removeLiquidityWithPermit```/```removeLiquidityETHWithPermit```/```removeLiquidityETHWithPermitSupportingFeeOnTransferTokens```) about ```permit```. These functions will call ```permit``` function in ```JalaPair.sol```. ```JalaPair``` is inherited from ```JalaERC20```. Although ```JalaERC20``` is out of scope. But both ```JalaPair``` and ```JalaERC20``` have no ```permit``` functions. So when you call ```removeLiquidityWithPermit```/```removeLiquidityETHWithPermit```/```removeLiquidityETHWithPermitSupportingFeeOnTransferTokens```, it will always revert.
## POC
Add this test function in ```JalaRouter02.t.sol```.
```solidity
function test_Permit() public {
        tokenA.approve(address(router), 1 ether);
        tokenB.approve(address(router), 1 ether);

        router.addLiquidity(
            address(tokenA), address(tokenB), 1 ether, 1 ether, 1 ether, 1 ether, address(this), block.timestamp
        );

        address pairAddress = factory.getPair(address(tokenA), address(tokenB));
        JalaPair pair = JalaPair(pairAddress);
        uint256 liquidity = pair.balanceOf(address(this));

        liquidity = (liquidity * 3) / 10;
        pair.approve(address(router), liquidity);

        vm.expectRevert();
        router.removeLiquidityWithPermit(
            address(tokenA),
            address(tokenB),
            liquidity,
            0.3 ether - 300,
            0.3 ether - 300,
            address(this),
            block.timestamp,
            true,
            1, // this value is for demonstration only
            bytes32(uint256(1)), // this value is for demonstration only
            bytes32(uint256(1)) // this value is for demonstration only
        );
    }
```
## Impact
We can't remove liquidity by using ```permit```.
## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L150-L167
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L169-L185
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L202-L225
## Tool used
manual review and foundry
## Recommendation
Implement ```permit``` function in ```JalaERC20```. Reference: https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2ERC20.sol.



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/jalaswap/jalaswap-dex-contract/commit/c73dda6e81268bb329ec80ef851707f3b95ff0df.

# Issue M-2: User wrapped tokens get stuck in master router because of incorrect calculation 

Source: https://github.com/sherlock-audit/2024-02-jala-swap-judging/issues/146 

## Found by 
Arabadzhiev, C1rdan, cawfree, den\_sosnovskyi, jah, mahmud, merlinboii, recursiveEth, yotov721
## Summary
Swapping exact tokens for ETH swaps underlying token amount, not wrapped token amount and this causes wrapped tokens to get stuck in the contract.

## Vulnerability Detail
In the protocol the `JalaMasterRouter` is used to swap tokens with less than 18 decimals. It is achieved by wrapping the underlying tokens and interacting with the `JalaRouter02`. **Wrapping** the token gives it decimals 18 (18 - token.decimals()). There are also functions that swap with native ETH. 

In the `swapExactTokensForETH` function the tokens are transferred from the user to the Jala master router, **wrapped**, approved to `JalaRouter2` and then `IJalaRouter02::swapExactTokensForETH()` is called with **the amount of tokens to swap**, to address, deadline and path. 

The amount of tokens to swap that is passed, is the amount before the wrap. Hence the wrappedAmount - underlyingAmount is stuck. 

Add the following test to `JalaMasterRouter.t.sol` and run with `forge test --mt testswapExactTokensForETHStuckTokens -vvv`
```javascript
    function testswapExactTokensForETHStuckTokens() public {
        address wrappedTokenA = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(address(tokenA));

        tokenA.approve(address(wrapperFactory), type(uint256).max);
        wrapperFactory.wrap(address(this), address(tokenA), 100);

        IERC20(wrappedTokenA).approve(address(router), 100 ether);
        router.addLiquidityETH{value: 100 ether}(wrappedTokenA, 100 ether, 0, 0, address(this), type(uint40).max);

        address pairAddress = factory.getPair(address(WETH), wrapperFactory.wrappedTokenFor(address(tokenA)));

        uint256 pairBalance = JalaPair(pairAddress).balanceOf(address(this));

        address[] memory path = new address[](2);
        path[0] = wrappedTokenA;
        path[1] = address(WETH);

        vm.startPrank(user0);
        console.log("ETH user balance before:       ", user0.balance);
        console.log("TokenA user balance before:    ", tokenA.balanceOf(user0));
        console.log("WTokenA router balance before: ", IERC20(wrappedTokenA).balanceOf(address(masterRouter)));

        tokenA.approve(address(masterRouter), 550);
        masterRouter.swapExactTokensForETH(address(tokenA), 550, 0, path, user0, type(uint40).max);
        vm.stopPrank();

        console.log("ETH user balance after:       ", user0.balance);
        console.log("TokenA user balance after:    ", tokenA.balanceOf(user0));
        console.log("WTokenA router balance after: ", IERC20(wrappedTokenA).balanceOf(address(masterRouter)));
    }
```


## Impact
User wrapped tokens get stuck in router contract. The can be stolen by someone performing a `swapExactTokensForTokens()` because it uses the whole balance of the contract when swapping: `IERC20(wrappedTokenIn).balanceOf(address(this))` 
```solidity
        amounts = IJalaRouter02(router).swapExactTokensForTokens(
            IERC20(wrappedTokenIn).balanceOf(address(this)),
            amountOutMin,
            path,
            address(this),
            deadline
        );
```

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L284-L301

## Tool used
Manual Review, foundry

## Recommendation
In `JalaMasterRouter::swapExactTokensForETH()` multiply the `amountIn` by decimal off set of the token:
```diff
    function swapExactTokensForETH(
        address originTokenAddress,
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external virtual override returns (uint256[] memory amounts) {
        address wrappedTokenIn = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(originTokenAddress);

        require(path[0] == wrappedTokenIn, "MS: !path");

        TransferHelper.safeTransferFrom(originTokenAddress, msg.sender, address(this), amountIn);
        _approveAndWrap(originTokenAddress, amountIn);
        IERC20(wrappedTokenIn).approve(router, IERC20(wrappedTokenIn).balanceOf(address(this)));

+        uint256 decimalOffset = IChilizWrappedERC20(wrappedTokenIn).getDecimalsOffset();
+        amounts = IJalaRouter02(router).swapExactTokensForETH(amountIn * decimalOffset, amountOutMin, path, to, deadline);
-        amounts = IJalaRouter02(router).swapExactTokensForETH(amountIn , amountOutMin, path, to, deadline);
    }
```



## Discussion

**nevillehuang**

If a sufficient slippage (which is users responsibility) is set, this will at most cause a revert, so medium severity is more appropriate. (The PoC set slippage to zero)

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/jalaswap/jalaswap-dex-contract/commit/9ed6e8f4f6ad762ef7b747ca2d367f6cfd78973e.

# Issue M-3: Permit functions in router can be affected by Dos 

Source: https://github.com/sherlock-audit/2024-02-jala-swap-judging/issues/177 

## Found by 
0xlucky, Norah, bughuntoor, cats, crypticdefense, deth, giraffe, goluu, honeymewn, jokr, sa9933, smbv-1919, thisvishalsingh, tpiliposian
## Summary
`JalaRouter02` supports permit functions which can be affected by Dos making them unusable

## Vulnerability Detail
`JalaRouter02` supports ERC20 permit functionality by which users could spend the tokens by signing an approval off-chain. In `JalaRouter02.removeLiquidityWithPermit`, after the permit call is successful there is a call to `removeLiquidity`.

```solidity

    function removeLiquidityWithPermit(
        address tokenA,
        address tokenB,
        uint256 liquidity,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline,
        bool approveMax,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external virtual override returns (uint256 amountA, uint256 amountB) {
        address pair = JalaLibrary.pairFor(factory, tokenA, tokenB);
        uint256 value = approveMax ? type(uint).max : liquidity;
        IJalaPair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
        (amountA, amountB) = removeLiquidity(tokenA, tokenB, liquidity, amountAMin, amountBMin, to, deadline);
    }
```

But while the transactions for `removeLiquidityWithPermit` is in mempool, anyone could extract the signature parameters from the call to `frontrun` the txn with direct `permit` call.

This would indeed increase the approval for the user. But as the `permit` is already been used, the call to `IJalaPair(pair).permit` in `removeLiquidityWithPermit` will revert making whole txn revert. Thus making the victim not able to make successful call to `removeLiquidityWithPermit` to remove his liquidity using permit


The above case holds for all the following functions below which can be affected by Dos.
1. removeLiquidityWithPermit
2. removeLiquidityETHWithPermit
3. removeLiquidityETHWithPermitSupportingFeeOnTransferTokens

## Impact
Users will not be able to use the permit functions for important actions

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L150-L167

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L169-L185

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L202-L225

## References

https://www.trust-security.xyz/post/permission-denied

## Tool used

Manual Review

## Recommendation
Wrap the permit calls in a try catch block
```diff
- IJalaPair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
+ try IJalaPair(pair).permit(msg.sender, address(this), value, deadline, v, r, s) {
+    (amountToken, amountETH) = removeLiquidityETH(token, liquidity, amountTokenMin, amountETHMin, to, deadline);
+ }
+ catch {
+     if (IJalaPair(pair).allowane(msg.sender, address(this)) >=  value)
+         (amountToken, amountETH) = removeLiquidityETH(token, liquidity, amountTokenMin, amountETHMin, to, deadline);
+     else 
+         revert("permit failed")
+ }
```

# Issue M-4: JalaPair potential permanent DoS due to overflow 

Source: https://github.com/sherlock-audit/2024-02-jala-swap-judging/issues/186 

The protocol has acknowledged this issue.

## Found by 
0k, 0xMojito, 0xRstStn, 0xloscar01, Stoicov, ZanyBonzy, den\_sosnovskyi, deth, fibonacci, giraffe, mahmud, n1punp, santiellena, sunill\_eth, tank
## Summary

In the `JalaPair::_update` function, overflow is intentionally desired in the calculations for `timeElapsed` and `priceCumulative`. This is forked from the UniswapV2 source code, and it’s meant and known to overflow. UniswapV2 was developed using Solidity 0.6.6, where arithmetic operations overflow and underflow by default. However, Jala utilizes Solidity >=0.8.0, where such operations will automatically revert.

## Vulnerability Detail

```solidity
uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
    // * never overflows, and + overflow is desired
    price0CumulativeLast += uint256(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
    price1CumulativeLast += uint256(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
}
```

## Impact

This issue could potentially lead to permanent denial of service for a pool. All the core functionalities such as `mint`, `burn`, or `swap` would be broken. Consequently, all funds would be locked within the contract.

I think issue with High impact and a Low probability (merely due to the extended timeframe for the event's occurrence, it's important to note that this event will occur with 100% probability if the protocol exists at that time), should be considered at least as Medium.

## References

There are cases where the same issue is considered High.

https://solodit.xyz/issues/h-02-uniswapv2priceoraclesol-currentcumulativeprices-will-revert-when-pricecumulative-addition-overflow-code4rena-phuture-finance-phuture-finance-contest-git
https://solodit.xyz/issues/m-02-twavsol_gettwav-will-revert-when-timestamp-4294967296-code4rena-nibbl-nibbl-contest-git
https://solodit.xyz/issues/trst-m-3-basev1pair-could-break-because-of-overflow-trust-security-none-satinexchange-markdown_

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L97-L102

## Tool used

Manual Review

## Recommendation

Use the `unchecked` block to ensure everything overflows as expected



## Discussion

**nevillehuang**

Request poc

Would like the watson/watsons to present a scenario where a reasonable overflow can be achieved, because based on my discussion with LSW this is likely not reasonable considering frequency of liquidity addition and ratio of reserves required

> #186 - let's put it this way. Ratio is scaled up by 2^112.  Having a ` (ratio * timeElapsed1) + (ratio * timeElapsed2)` is the same as having `(ratio * (timeElapsed1 + timeElapsed2)`. So if we have a token1 max reserve of uint112 and token0 reserve is 1, `uint112.max * uint112.max * totalTimeElapsed`. In order for this to overflow, we need totalTimeElapsed to be > uint32.max, which is approx 132 years. So for this to overflow, we'd need to have the pool running for 132 years with one of the reserve being the max and the other one being just 1 wei for the entirety of the 132 years.


**sherlock-admin3**

PoC requested from @0xf1b0

Requests remaining: **10**

**0xf1b0**

Even if we do not take reserves into account, the timestamp is converted into a 32-bit value.

`uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);`

When the timestamp gets >4294967296 (in 82 years), the `_update` function will always revert.


**nevillehuang**

@Czar102 what do you think? I don’t think its severe enough to warrant medium severity. I remember you rejected one of a similar finding in dodo as seen [here](https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/155)

**Czar102**

The linked issue presented a way to DoS functionality. Is this also the case here? Or are funds locked here as well?
It seems to me that the funds are locked, so I'd accept this as a valid Medium (High impact in far future – a limitation)/

**nevillehuang**

@Czar102 Agree, if long enough, all core functionalities `burn()` (Remove liquidity), `mint()` (add liquidity) and `swap()` that depends on this low level functions have the potential to be DoSed due to `_update()` reverting, will leave as medium severity

