Faithful Lilac Alpaca

medium

# Minting fees to `DEAD` address

## Summary
`JalaPair` contract mints fees to `0x000000000000000000000000000000000000dEaD` which no one can control.

## Vulnerability Detail
`JalaFactory` initially sets `feeTo` param to a random address in the constructor
```solidity
address public constant DEAD = 0x000000000000000000000000000000000000dEaD;
constructor(address _feeToSetter) {
    feeToSetter = _feeToSetter;
    flashOn = false;
    feeTo = DEAD;
}
```
`JalaPair._mintFee` checks if the fee is enabled by checking if the `feeTo` address is not address(0). It mints fee to the address if its enabled
```solidity
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IJalaFactory(factory).feeTo(); // get feeTo address
    //@audit minting fee tokens to dead address
    feeOn = feeTo != address(0);
    uint256 _kLast = kLast; // gas savings
    if (feeOn) {
        if (_kLast != 0) {
            uint256 rootK = Math.sqrt(uint256(_reserve0) * _reserve1);
            uint256 rootKLast = Math.sqrt(_kLast);
            if (rootK > rootKLast) {
                uint256 numerator = totalSupply * (rootK - rootKLast);
                uint256 denominator = rootK + rootKLast;
                uint256 liquidity = numerator / denominator;
                // distribute LP fee
                if (liquidity > 0) _mint(feeTo, liquidity);
            }
        }
    } else if (_kLast != 0) {
        kLast = 0;
    }
}
```

As the `feeTo` is initially set to random `DEAD` address, the `feeOn` is always true even if the `feeToSetter` intends to keep the fee disabled. Thus the fee tokens are minted to `DEAD` address essentially burning the fee.

## Impact
Fee tokens get burned

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaFactory.sol#L12

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaFactory.sol#L23-L27

https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L110-L129

## Tool used

Manual Review

## Recommendation

Leave the `feeTo` in `JalaFactory` as 0
 