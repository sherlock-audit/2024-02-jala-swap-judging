Elegant Ocean Moose

medium

# issues related to Fan token paused.

## Summary
1. Since Fan Tokens like BAR and PSG are supported in the system , those type of token can be [paused](https://scan.chiliz.com/token/0xFD3C73b3B09D418841dd6Aff341b2d6e3abA433b/write-proxy), according to the source code, if the token is puased, the token can't be transferred anymore.
2. Our system can wrap those Fan Tokens, and even if the Fan Token is paused, the wrapped token can still be transferred.

In such case, there might some issues.

## Vulnerability Detail
1. A technical user can use the wrapped token as a safe harbor to reduce the risk of the Fan Token pused.
    After Alice got a BAR token, she can call [ChilizWrappedERC20.depositFor](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L32-L43) to convert her BAR to WBAR. In the case of BAR is paused, she can use WBAR to swap to other token like WPSG or ETH to reduce her loss
2. add liquidity:
In the case of a non-technical user calls [JalaMasterRouter.wrapTokensAndaddLiquidity](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L32-L69) to add liquidity, if BAR is paused, the user won't remove his liquidity because when calling [JalaMasterRouter.removeLiquidityAndUnwrapToken](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L100-L149), the function will calls [ChilizWrapperFactory.unwrap](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol#L24-L27) in [JalaMasterRouter.sol#L136](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L136) and `ChilizWrapperFactory.unwrap` will call [ChilizWrappedERC20.withdrawTo](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L45-L60) in [ChilizWrapperFactory.sol#L26](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol#L26). And in `ChilizWrappedERC20.withdrawTo` the function will revert in [ChilizWrappedERC20.sol#L52](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L52)
    But for a technical user, he can find the wrapped token pair address, and calls `router.removeLiquidity` with the wrapped token as parameter. In such case, after he receives the wrapped token, he can swap the transfer the wraped token to other unpaused token.

## Impact
By current design, non-technical user suffer more risk.

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L33-L60

## Tool used

Manual Review

## Recommendation
