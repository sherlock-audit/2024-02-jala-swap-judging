Ancient Cornflower Sawfish

medium

# tokens with approval race protection or not returning a `bool` on `approve` are incompatible with protocol

## Summary

There are two issues here:

1. Some tokens, for example [USDT](https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7) (6 decimals) and [KNC](https://etherscan.io/token/0xdd974d5c2e2928dea5f71b9825b8b646686bd200#code) (18 decimals) have approval race protection mechanism and require the allowance to be either `0` or `uint256.max` when it is updated.

2. There are tokens that do not follow the ERC20 standard (like USDT again) that do not return a bool on approve call. Those tokens are incompatible with the protocol because Solidity will check the return data size, which will be zero and will lead to a revert.

## Vulnerability Detail

In `JalaMasterRouter.sol` and `ChilizWrapperFactory.sol`, ERC20 token approve is done by `token.approve(address, amount);`,

    //ChilizWrapperFactory.sol
    function _transferTokens(IERC20 token, address approveToken, uint256 amount) internal {
        SafeERC20.safeTransferFrom(token, msg.sender, address(this), amount);
        if (token.allowance(address(this), approveToken) < amount) {
            token.approve(approveToken, amount);
        }
    }

    //JalaMasterRouter.sol
    function _approveAndWrap(address token, uint256 amount) private returns (address wrappedToken) {
        IERC20(token).approve(wrapperFactory, amount); // no need for check return value, bc addliquidity will revert if approve was declined.
        wrappedToken = IChilizWrapperFactory(wrapperFactory).wrap(address(this), token, amount);
    }

However, this may not work with tokens that have approval race protection mechanism and will revert when currentAllowance is non-zero, and this will also not work with tokens that do not return a bool on approve call.

## Impact

Some tokens are incompatible with Jalaswap protocol, for example, `USDT`.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaMasterRouter.sol#L329
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol#L72

## Tool used

Manual Review

## Recommendation

Use [`forceapprove`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L76-L83) to solve this problem.