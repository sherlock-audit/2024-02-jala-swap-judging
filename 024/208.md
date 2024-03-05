Shaggy White Woodpecker

high

# `JalaPair.sol::_safeTransfer`  function does not check for the existence of the ERC20 token contract

## Summary

The `safeTransfer` function does not check for the existence of the ERC20 token contract , JalaPair.sol performs a transfer with a low-level call without confirming the contract's existence.

## Vulnerability Detail

```solidity
    function _safeTransfer(address token, address to, uint256 value) private {
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(SELECTOR, to, value));
        if (!success || (data.length != 0 && !abi.decode(data, (bool)))) revert TransferFailed();
    }
```
The low-level functions call, delegatecall and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed.
[Docs_soliditylang](https://docs.soliditylang.org/en/develop/control-structures.html#error-handling-assert-require-revert-and-exceptions)

As a result, if the tokens have not yet been deployed or have been destroyed, `_safeTransfer` will return success even though no transfer was executed. If the token has not yet been deployed, no liquidity can be added. However, if the token has been destroyed, the pool will act as if the assets were sent even though they were not.

### Additional 

There is also in this library TransferHelper::safeTransfer function vulnerable to this but this is `out of scope` for this contest

```solidity
    function safeTransfer(
        address token,
        address to,
        uint256 value
    ) internal {
        // bytes4(keccak256(bytes('transfer(address,uint256)')));
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0xa9059cbb, to, value));
        if (!success || (data.length != 0 && !abi.decode(data, (bool)))) revert TransferFailed();
    }
```
## Impact

1. Alice creates a pair of A and B Tokens (For exampleETH - TestToken Pair). 

2. The creator and only owner of TestToken is Alice. 

3. Next, Alice destroys the TestToken with a Selfdestruct based on onlyowner privilege.

4. Bob, unaware of this, deposits ETH into the pool to trade from the ETH-TestToken pair, but cannot receive any tokens because `_safeTransfer` does not check for the existence of the contract.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L65-L69

## Tool used

Manual Review

## Recommendation

Have the `_safeTransfer` function check the existence of the contract before every transaction.

