Tricky Khaki Dalmatian

medium

# Rounding Errors in withdrawTo Function Could Lead to Undesirable Outcomes

## Summary

The `withdrawTo` function in [ChilizWrappedERC20 contract](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol) calculates the amount of the underlying token to withdraw (unwrapAmount) by dividing the requested amount by decimalsOffset.

Because Solidity does not support fractional numbers and automatically floors the result of division, this can lead to rounding errors. When the amount of wrapped tokens to be burned does not perfectly convert into an underlying token amount (due to decimalsOffset discrepancies), the division may result in a slightly lower unwrapAmount than theoretically entitled, leaving residual wrapped tokens or allowing for fractional advantages in withdrawals.

## Vulnerability Detail

Consider the below case - 

- The underlying token (UT) has 6 decimals.
- The user deposits 1,000,000 UT (equivalent to 1 UT in a standard 18-decimal format, since decimalsOffset = (10)^12
- The user then decides to withdraw 500,000 wrapped UT tokens.

Given decimalsOffset = (10)^12, the contract calculates: 
unwrapAmount = 500,000 / 10^(12) = 0.0000005 UT in 18-decimal format, which, when converted back to the 6-decimal format of the underlying token, should technically equal 0.5 UT.

However, Solidity does not handle fractions and will truncate this to 0 UT. Thus, burntAmount = 0 * 10^{12} = 0 wrapped UT.

## Impact

The user attempts to withdraw half of their wrapped tokens but ends up with no underlying tokens due to the rounding error.
Moreover, since the contract does not handle the fractional part, the user loses the value of the tokens they attempted to withdraw, resulting in a loss of funds without any actual transfer of the underlying tokens.

## Code Snippet

```solidity
function withdrawTo(address account, uint256 amount) public virtual returns (bool) {
        if (address(underlyingToken) == address(0)) revert NotInitialized();
        uint256 unwrapAmount = amount / decimalsOffset;
        if (unwrapAmount == 0) revert CannotWithdraw();
        address msgSender = _msgSender();
        uint256 burntAmount = unwrapAmount * decimalsOffset;
        _burn(msgSender, burntAmount);
        SafeERC20.safeTransfer(underlyingToken, account, unwrapAmount);
        if (msgSender != account) {
            _transfer(msgSender, account, amount - burntAmount);
        }

        emit Withdraw(account, amount);

        return true;
    }
```

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L45-#L60

## Tool used

Manual Review

## Recommendation

Implement a more sophisticated rounding mechanism that rounds down the unwrapAmount to the nearest whole number but keeps track of the residual wrapped tokens that were not unwrapped due to rounding. This residual could be credited to the user's account in some form, either as a fractional wrapped token they can redeem later or by adjusting the withdrawal amount to ensure that no value is lost.


