Faithful Lilac Alpaca

medium

# Permit functions in router can be affected by Dos

## Summary
`JalaRouter02` supports permit functions which can be affected by Dos making them unusable

## Vulnerability Detail
`JalaRouter02` supports ERC20 permit functionality by which users could spend the tokens by signing an approval off-chain. In `JalaRouter02.removeLiquidityWithPermit`, after the permit call is successful there is a call to `removeLiquidity`.

```solidity

    function removeLiquidityWithPermit(
        address tokenA,
        address tokenB,
        uint256 liquidity,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline,
        bool approveMax,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external virtual override returns (uint256 amountA, uint256 amountB) {
        address pair = JalaLibrary.pairFor(factory, tokenA, tokenB);
        uint256 value = approveMax ? type(uint).max : liquidity;
        IJalaPair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
        (amountA, amountB) = removeLiquidity(tokenA, tokenB, liquidity, amountAMin, amountBMin, to, deadline);
    }
```

But while the transactions for `removeLiquidityWithPermit` is in mempool, anyone could extract the signature parameters from the call to `frontrun` the txn with direct `permit` call.

This would indeed increase the approval for the user. But as the `permit` is already been used, the call to `IJalaPair(pair).permit` in `removeLiquidityWithPermit` will revert making whole txn revert. Thus making the victim not able to make successful call to `removeLiquidityWithPermit` to remove his liquidity using permit


The above case holds for all the following functions below which can be affected by Dos.
1. removeLiquidityWithPermit
2. removeLiquidityETHWithPermit
3. removeLiquidityETHWithPermitSupportingFeeOnTransferTokens

## Impact
Users will not be able to use the permit functions for important actions

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L150-L167

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L169-L185

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaRouter02.sol#L202-L225

## References

https://www.trust-security.xyz/post/permission-denied

## Tool used

Manual Review

## Recommendation
Wrap the permit calls in a try catch block
```diff
- IJalaPair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
+ try IJalaPair(pair).permit(msg.sender, address(this), value, deadline, v, r, s) {
+    (amountToken, amountETH) = removeLiquidityETH(token, liquidity, amountTokenMin, amountETHMin, to, deadline);
+ }
+ catch {
+     if (IJalaPair(pair).allowane(msg.sender, address(this)) >=  value)
+         (amountToken, amountETH) = removeLiquidityETH(token, liquidity, amountTokenMin, amountETHMin, to, deadline);
+     else 
+         revert("permit failed")
+ }
```
