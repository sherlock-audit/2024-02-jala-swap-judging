Scrawny Cyan Orca

high

# User wrapped tokens get stuck in master router because of incorrect calculation

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