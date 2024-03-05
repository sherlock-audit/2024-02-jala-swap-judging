Smooth Sapphire Boa

high

# Swapping will result in 0 output amount for beneficiary

## Summary
When a user swaps he will get zero output amount due to missing upper limit definition in `setFlashFee()`
## Vulnerability Detail
As seen below, `setFlashFee()` allows the owner to set `flashFee` to any value within the range of `uint256` by the `onlyFeeToSetter`  :

```solidity
function setFlashFee(uint _flashFee) external onlyFeeToSetter {
        flashFee = _flashFee;
        emit SetFlashFee(_flashFee);
    }
```


This would allow the owner to obtain up to 100% of the swap fee for all swaps when.

When we look at the `swap()` function in `JalaPair.sol` their is an if condition that checks if `flashOn` and transfers the fee to the owners address if flash swap is done, but if the owner decides to change the flash fee to 100% the user will lose the whole value in the swap and will receive nothing.

```solidity
 if (IJalaFactory(factory).flashOn() && data.length > 0) {
                if (amount0Out > 0) {
                    _safeTransfer(
                        _token0,
                        IJalaFactory(factory).feeTo(),
                        (amount0Out * IJalaFactory(factory).flashFee()) / 10000
                    );
  @>                  amount0Out = (amount0Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                if (amount1Out > 0) {
                    _safeTransfer(
                        _token1,
                        IJalaFactory(factory).feeTo(),
                        (amount1Out * IJalaFactory(factory).flashFee()) / 10000
                    );
@>                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                IJalaCallee(to).JalaCall(msg.sender, amount0Out, amount1Out, data);
            }
```




## Impact
User loses whole value to fee during flash swap.
## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L224-L242
## Tool used

Manual Review

## Recommendation
Consider declaring a reasonable upper limit in setFlashFee(), such as 20%. This would prevent users from losing their value to swaps and increase the trust of users in the contract.