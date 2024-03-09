Eager Turquoise Tarantula

medium

# Any Body can Skim Funds in the JalaPair

## Summary
Any Body can call and Skim Funds in the JalaPair contract due to missing validation
## Vulnerability Detail
```solidity
  function skim(address to) external lock {
        address _token0 = token0; // gas savings
        address _token1 = token1; // gas savings
        _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)) - reserve0);
        _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)) - reserve1);
    }
```
As noted from the skim function provided above no validation is present before fund is skimed from the contract to address(to)
## Impact
Any Body can call and Skim Funds in the JalaPair contract due to missing validation thereby causing loss of fund to the protocol
## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L262
## Tool used

Manual Review

## Recommendation
The protocol should implement necessary validation to ensure only authorized address can skim, if skimming is really necessary another alternative is to add a open and close functionality to the function which gives a form of control over the skim function