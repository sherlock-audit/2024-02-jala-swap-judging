Narrow Garnet Boa

medium

# Big flash loans will fail due to fee

## Summary

The introduction of additional flash loan fees will prevent users from requesting flash loans of large amounts, since the associated fees are transferred from the pool before the callback is executed.

## Vulnerability Detail

When a user requests a flash loan, the implementation will:

1. Transfer the requested amount to the recipient.
2. Transfer the associated `flashFee()` to the `feeTo` address.
3. Execute the callback `JalaCall()`.

```solidity
if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out); // optimistically transfer tokens
if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out); // optimistically transfer tokens
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

This implies that the pool should not only have the requested funds for the loan, but also additional funds to also pay for the fees, since these are paid **before** executing the callback.

## Impact

Flash loans of large amounts won't be possible as the implementation will revert when trying to optimistically transfer the associated fees.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L222-L242

## Tool used

Manual Review

## Recommendation

Transfer the associated fees **after** the callback, so the requester can have the chance to return the funds and the additional fees:

1. Transfer the requested amount to the recipient.
2. Callback is executed.
3. Recipient returns the requested amount and the flash loan fees.
4. Fees are transferred to `feeTo`.
