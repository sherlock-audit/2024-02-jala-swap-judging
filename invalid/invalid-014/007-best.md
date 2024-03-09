Quiet Chiffon Mule

high

# ChilizWrappedERC20 :: initialize() will revert when wrapping tokens with 18 decimals

## Summary

A user will execute  `` ChilizWrapperFactory :: createWrappedToken() ``  with  `` underlyingToken ``  argument equal with  `` [[ A normal & common token with 18 decimals ]] ``  but  `` ChilizWrappedERC20 :: initialize() ``  will not let tokens with 18 decimals to be wrapped because of this line of code  `` if (   _underlyingToken.decimals()   >=   18   )   revert InvalidDecimals();  ``



https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrappedERC20.sol#L20C9-L20C73



Here is the link to the PoC that will showcase the existence of the vulnerability

Execute the PoC using   ``  forge  test  --match-path  test/A.t.sol  --match-test  test____A____  -vv  --via-ir  ``  and read the terminal 


``  https://gist.github.com/UP31JGO3EU91FEN913/432b9c07552f6b18f7863759af5ac307  ``






## Vulnerability Detail

_

## Impact

_

## Code Snippet

_

## Tool used

Manual Review

## Recommendation

Instead of  `` if (   _underlyingToken.decimals()   >=   18   )   revert InvalidDecimals();  ``  use  `` if (   _underlyingToken.decimals()   >   18   )   revert InvalidDecimals();  `` 

With this modification, the tokens with 18 decimals will be accepted too