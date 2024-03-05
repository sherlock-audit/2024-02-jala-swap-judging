Narrow Garnet Boa

medium

# JalaMasterRouter fails to scale minumum output amounts to wrapped units

## Summary

Several functions in the JalaMasterRouter take amount values that are not scaled to the decimal offset of their wrapped equivalent.

## Vulnerability Detail

In `removeLiquidityAndUnwrapToken()`, both `amountAMin` and `amountBMin` should be scaled to the wrapped units since from the perspective of the pool, these amounts correspond to wrapped tokens.

```solidity
function removeLiquidityAndUnwrapToken(
    address tokenA, // token origin addr
    address tokenB,
    uint256 liquidity,
    uint256 amountAMin,
    uint256 amountBMin,
    address to,
    uint256 deadline
) external virtual override returns (uint256 amountA, uint256 amountB) {
    address wrappedTokenA = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(tokenA);
    address wrappedTokenB = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(tokenB);
    address pair = JalaLibrary.pairFor(factory, wrappedTokenA, wrappedTokenB);
    TransferHelper.safeTransferFrom(pair, msg.sender, address(this), liquidity);

    IERC20(pair).approve(router, liquidity);

    (amountA, amountB) = IJalaRouter02(router).removeLiquidity(
        wrappedTokenA,
        wrappedTokenB,
        liquidity,
        amountAMin,
        amountBMin,
        address(this),
        deadline
    );
```

In `removeLiquidityETHAndUnwrap()`, the `amountTokenMin` variable should also be scaled since the underlying liquidity is using the wrapped token.

```solidity
function removeLiquidityETHAndUnwrap(
    address token,
    uint256 liquidity,
    uint256 amountTokenMin,
    uint256 amountETHMin,
    address to,
    uint256 deadline
) public virtual override returns (uint256 amountToken, uint256 amountETH) {
    address wrappedToken = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(token);
    address pair = JalaLibrary.pairFor(factory, wrappedToken, address(WETH));
    TransferHelper.safeTransferFrom(pair, msg.sender, address(this), liquidity);

    IERC20(pair).approve(router, liquidity);

    (amountToken, amountETH) = IJalaRouter02(router).removeLiquidityETH(
        wrappedToken,
        liquidity,
        amountTokenMin,
        amountETHMin,
        address(this),
        deadline
    );
```

Similarly, in both `swapExactTokensForTokens()` and `swapExactETHForTokens()`, the minimum output amounts `amountOutMin` should be scaled by the decimal offset since the output tokens are the wrapped variants.

```solidity
function swapExactTokensForTokens(
    address originTokenAddress,
    uint256 amountIn,
    uint256 amountOutMin,
    address[] calldata path,
    address to,
    uint256 deadline
) external virtual override returns (uint256[] memory amounts, address reminderTokenAddress, uint256 reminder) {
    address wrappedTokenIn = IChilizWrapperFactory(wrapperFactory).wrappedTokenFor(originTokenAddress);

    require(path[0] == wrappedTokenIn, "MS: !path");

    TransferHelper.safeTransferFrom(originTokenAddress, msg.sender, address(this), amountIn);
    _approveAndWrap(originTokenAddress, amountIn);
    IERC20(wrappedTokenIn).approve(router, IERC20(wrappedTokenIn).balanceOf(address(this)));

    amounts = IJalaRouter02(router).swapExactTokensForTokens(
        IERC20(wrappedTokenIn).balanceOf(address(this)),
        amountOutMin,
        path,
        address(this),
        deadline
    );
    (reminderTokenAddress, reminder) = _unwrapAndTransfer(path[path.length - 1], to);
}
```

```solidity
function swapExactETHForTokens(
    uint256 amountOutMin,
    address[] calldata path,
    address to,
    uint256 deadline
)
    external
    payable
    virtual
    override
    returns (uint256[] memory amounts, address reminderTokenAddress, uint256 reminder)
{
    amounts = IJalaRouter02(router).swapExactETHForTokens{value: msg.value}(
        amountOutMin,
        path,
        address(this),
        deadline
    );
    (reminderTokenAddress, reminder) = _unwrapAndTransfer(path[path.length - 1], to);
}
```

## Impact

Failure to properly scale the minimum output amounts means that slippage control is severely damaged, as the provided quantities are much less than intended.

For example, if the token has 0 decimals and the wrapped variant has 18 decimal, then a value of say `1000` would need to be represented as `1000 * 10**18`, meaning that slippage amounts are off by a factor of `10**18`.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L101-L125

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L151-L172

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L191-L215

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L263-L282

## Tool used

Manual Review

## Recommendation

In each of the highlighted cases, multiply the amounts by their respective decimal offset.
