Ancient Cornflower Sawfish

medium

# ChilizWrapper would not work on tokens without `name` or `symbol`

## Summary

[ERC20 standard](https://eips.ethereum.org/EIPS/eip-20) emphasizes that contracts should not assume that token's name or symbol exist: 

>  ### name
> Returns the name of the token - e.g. "MyToken".
> 
> OPTIONAL - This method can be used to improve usability, but interfaces and other contracts MUST NOT expect these values to be present.
> 
>     function name() public view returns (string)
>  ### symbol
> Returns the symbol of the token. E.g. “HIX”.
> 
> OPTIONAL - This method can be used to improve usability, but interfaces and other contracts MUST NOT expect these values to be present.
> 
>     function symbol() public view returns (string)
> 

But, current `ChilizWrapperFactory` expect token's `name()` and `symbol()` both exist, which makes some tokens cannot get wrapped.

## Vulnerability Detail

In `ChilizWrappedERC20`, 

    function initialize(IERC20 _underlyingToken) external {
        if (msg.sender != factory) revert Forbidden();
        if (_underlyingToken.decimals() >= 18) revert InvalidDecimals();
        if (address(underlyingToken) != address(0)) revert AlreadyExists();

        underlyingToken = _underlyingToken;
        decimalsOffset = 10 ** (18 - _underlyingToken.decimals());
        name = string(abi.encodePacked("Wrapped ", _underlyingToken.name()));
        symbol = string(abi.encodePacked("W", _underlyingToken.symbol()));
    }

For tokens with only `name` and no `symbol` or only `symbol` and no `name`, wrapper cannot wrap them because solidity will revert.

It is worth noting that OZ's [IERC20.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol) does not include `name()` and `symbol()`. Some Fan tokens may not have one of the two functions.

## Impact

Tokens without `symbol` or `name` would revert on initialization.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L18-L27

## Tool used

Manual Review

## Recommendation

Use try-catch to ignore such revert.

## Note for judges

I believe this is not a issue targeting weird ERC20, but a issue about improper implementation of EIP-20.

From Contest README:

> Is the code/contract expected to comply with any EIPs? Are there specific assumptions around adhering to those EIPs that Watsons should be aware of?

> ERC-20

In `EIP20`, it is emphasized that interfaces and other contracts MUST NOT expect name&symbol to be present.

From the Hierarchy of truth, I believe this should be a valid medium issue.