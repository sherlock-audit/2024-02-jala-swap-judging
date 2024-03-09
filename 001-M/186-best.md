Upbeat Beige Spider

medium

# JalaPair potential permanent DoS due to overflow

## Summary

In the `JalaPair::_update` function, overflow is intentionally desired in the calculations for `timeElapsed` and `priceCumulative`. This is forked from the UniswapV2 source code, and it’s meant and known to overflow. UniswapV2 was developed using Solidity 0.6.6, where arithmetic operations overflow and underflow by default. However, Jala utilizes Solidity >=0.8.0, where such operations will automatically revert.

## Vulnerability Detail

```solidity
uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
    // * never overflows, and + overflow is desired
    price0CumulativeLast += uint256(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
    price1CumulativeLast += uint256(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
}
```

## Impact

This issue could potentially lead to permanent denial of service for a pool. All the core functionalities such as `mint`, `burn`, or `swap` would be broken. Consequently, all funds would be locked within the contract.

I think issue with High impact and a Low probability (merely due to the extended timeframe for the event's occurrence, it's important to note that this event will occur with 100% probability if the protocol exists at that time), should be considered at least as Medium.

## References

There are cases where the same issue is considered High.

https://solodit.xyz/issues/h-02-uniswapv2priceoraclesol-currentcumulativeprices-will-revert-when-pricecumulative-addition-overflow-code4rena-phuture-finance-phuture-finance-contest-git
https://solodit.xyz/issues/m-02-twavsol_gettwav-will-revert-when-timestamp-4294967296-code4rena-nibbl-nibbl-contest-git
https://solodit.xyz/issues/trst-m-3-basev1pair-could-break-because-of-overflow-trust-security-none-satinexchange-markdown_

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L97-L102

## Tool used

Manual Review

## Recommendation

Use the `unchecked` block to ensure everything overflows as expected
