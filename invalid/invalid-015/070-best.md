Brisk Rose Snake

high

# ChilizWrapperFactory does not use Ownable

## Summary
ChilizWrapperFactory does not use Ownable

## Vulnerability Detail
for ChilizWrapperFactory, it Inherits from Ownable but does not use Ownable. 

## Impact
mismanagement of privileges and gas fee waste

## Code Snippet
```solidity
contract ChilizWrapperFactory is IChilizWrapperFactory, Ownable {
```
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol?plain#L11
## Tool used

Manual Review

## Recommendation
use modifier like ```onlyOwner``` or delete Ownable