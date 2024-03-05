Gorgeous Bubblegum Whale

medium

# JalaLibrary.getAmountIn uses hardcoded fees, but JalaPairs have custom fees

## Summary
JalaLibrary's fees don't match the fees of JalaPair

## Vulnerability Detail
`getAmountIn` uses fees to determine the `amountIn` necessary to receive a given `amountOut` (see [JalaLibrary.sol#L85-L95](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/libraries/JalaLibrary.sol#L85-L95)):
```solidity
    function getAmountIn(
        uint256 amountOut,
        uint256 reserveIn,
        uint256 reserveOut
    ) internal pure returns (uint256 amountIn) {
        if (amountOut == 0) revert InsufficientOutputAmount();
        if (reserveIn == 0 || reserveOut == 0) revert InsufficientLiquidity();
        uint256 numerator = reserveIn * amountOut * 1000;
        uint256 denominator = (reserveOut - amountOut) * 997;
        amountIn = (numerator / denominator) + 1;
    }
```
This is cognizant of fees (**_997_**), but the fees are hardcoded, which are different from the JalaPairs' fees, since `JalaPairs` allows custom fees to be set via the factory (see [JalaPair.sol#L224-L242](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L224-L242)):
```solidity
          if (IJalaFactory(factory).flashOn() && data.length > 0) {
                if (amount0Out > 0) {
                    _safeTransfer(
                        _token0,
                        IJalaFactory(factory).feeTo(),
                        (amount0Out * IJalaFactory(factory).flashFee()) / 10000
                    );
                    amount0Out = (amount0Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                if (amount1Out > 0) {
                    _safeTransfer(
                        _token1,
                        IJalaFactory(factory).feeTo(),
                        (amount1Out * IJalaFactory(factory).flashFee()) / 10000
                    );
                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                IJalaCallee(to).JalaCall(msg.sender, amount0Out, amount1Out, data);
          }
```
## Impact
Doing this may cause the math to be incorrect

## Code Snippet
[JalaLibrary.sol#L85-L95](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/libraries/JalaLibrary.sol#L85-L95)

[JalaPair.sol#L224-L242](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L224-L242)

## Tool used

Manual Review

## Recommendation
Consider adding support for custom fees on JalaPairs
