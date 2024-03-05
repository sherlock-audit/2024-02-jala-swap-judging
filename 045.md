Trendy Sandstone Griffin

medium

# Excessive `_mintFee`s due to amplified error term.

## Summary

[`JalaPair`](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol)'s [`_mintFee(uint112,uint112)`](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L110C14-L110C60) function can charge up to a maximum excess of 100% fees when minting or burning liquidity.

## Vulnerability Detail

In [`JalaPair`](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol)'s fork of [`UniswapV2Pair`](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol), the fee logic has been modified from:

### 📄 [UniswapV2Pair.sol](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol)

```solidity
// if fee is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IUniswapV2Factory(factory).feeTo();
    feeOn = feeTo != address(0);
    uint _kLast = kLast; // gas savings
    if (feeOn) {
        if (_kLast != 0) {
            uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
            uint rootKLast = Math.sqrt(_kLast);
            if (rootK > rootKLast) {
                uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                uint denominator = rootK.mul(5).add(rootKLast);
                uint liquidity = numerator / denominator;
                if (liquidity > 0) _mint(feeTo, liquidity);
            }
        }
    } else if (_kLast != 0) {
        kLast = 0;
    }
}
```

To:

### 📄 [JalaPair.sol](https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol)

```solidity
// if fee is on, mint liquidity equivalent to 1/2th of the growth in sqrt(k)
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IJalaFactory(factory).feeTo(); // get feeTo address
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

The issue in question relates to the calculation of the `denominator`. Let's render the diff between between both versions:

```diff
- uint numerator = totalSupply.mul(rootK.sub(rootKLast));
+ uint256 numerator = totalSupply * (rootK - rootKLast);
- uint denominator = rootK.mul(5).add(rootKLast);
+ uint256 denominator = rootK + rootKLast;
uint256 liquidity = numerator / denominator;
```

1. We can see that previously, UniswapV2 [proposes a charge of](https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L88C5-L88C81) around $\frac{\sqrt{\delta_k}}{6}$.
2. In the repo fork, Jalaswap [intends to charge](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaPair.sol#L109C5-L109C81) $\frac{\sqrt{\delta_k}}{2}$.

If we run Uniswap's calculation through fuzz testing (assuming `1e18` to represent the `totalSupply` in order to procure a percentage denominated in ether), we can approximate a maximum `ratio` (normalized representation of fees) of around `0.2 ether`, or 20%:

```solidity
/// forge-config: default.fuzz.runs = 1000000
function testFuzz_uniswapV2Fees(
    uint256 _reserve0,
    uint256 _reserve1,
    uint256 _kLast
) public pure {

    uint256 MAX_UQ112x112 = 2**112 - 1;

    if (_reserve0 < 1_000 || _reserve1 < 1_000) return;
    vm.assume(_reserve0 >= 1_000);
    vm.assume(_reserve1 >= 1_000);

    if (_reserve0 > MAX_UQ112x112 || _reserve1 > MAX_UQ112x112) return;
    vm.assume(_reserve0 <= MAX_UQ112x112);
    vm.assume(_reserve1 <= MAX_UQ112x112);

    vm.assume(_kLast > 0);


    uint rootK = Math.sqrt(_reserve0 * _reserve1);
    uint rootKLast = Math.sqrt(_kLast);

    if (rootK > rootKLast) {
        uint numerator = (rootK - rootKLast) * 1e18;
        uint denominator = (rootK * 5) + rootKLast;
        uint ratio = numerator / denominator;

        assert(ratio < 0.20 ether); /// @audit reduce_for_counterexample
    }

}
```

Conversely, if we execute Jalaswap's fee calculation logic through a similar fuzz test bench, we compute an approximate maximum fee ratio of around 100%:

```solidity
/// forge-config: default.fuzz.runs = 1000000
function testFuzz_jalaswapFees(
    uint256 _reserve0,
    uint256 _reserve1,
    uint256 _kLast
) public pure {

    uint256 MAX_UQ112x112 = 2**112 - 1;

    if (_reserve0 < 1_000 || _reserve1 < 1_000) return;
    vm.assume(_reserve0 >= 1_000);
    vm.assume(_reserve1 >= 1_000);

    if (_reserve0 > MAX_UQ112x112 || _reserve1 > MAX_UQ112x112) return;
    vm.assume(_reserve0 <= MAX_UQ112x112);
    vm.assume(_reserve1 <= MAX_UQ112x112);

    vm.assume(_kLast > 0);


    uint rootK = Math.sqrt(_reserve0 * _reserve1);
    uint rootKLast = Math.sqrt(_kLast);

    if (rootK > rootKLast) {
        uint numerator = (rootK - rootKLast) * 1e18;
        uint denominator = rootK + rootKLast;
        uint ratio = numerator / denominator;

        assert(ratio < 1 ether); /// @audit reduce_for_counterexample
    }

}
```

To put this in perspective, Uniswap's has a maximum fee deviation of around **~20%**:

$$
((0.2 - 0.166667) / 0.166667) * 100 = 19.9967%
$$

By contrast, Jalaswap now possesses a maximum fee deviation of **100%**:

$$
((1 - 0.5) / 0.5) * 100 = 100%
$$

## Impact

Significant amplification of fees leading to dilution of LP shares.

## Code Snippet

```solidity
// if fee is on, mint liquidity equivalent to 1/2th of the growth in sqrt(k)
function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
    address feeTo = IJalaFactory(factory).feeTo(); // get feeTo address
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

## Tool used

Foundry, Halmos

## Recommendation

To reduce the impact of the contribution of the error term when calculating fees, we can take Uniswap's existing calculation and multiply the `numerator` by $3$ to achieve the equivalent of $\frac{\sqrt{\delta_k}}{2}$.

```solidity
/// forge-config: default.fuzz.runs = 1000000
function testFuzz_jalaswapFeesFix(
    uint256 _reserve0,
    uint256 _reserve1,
    uint256 _kLast
) public pure {

    uint256 MAX_UQ112x112 = 2**112 - 1;

    if (_reserve0 < 1_000 || _reserve1 < 1_000) return;
    vm.assume(_reserve0 >= 1_000);
    vm.assume(_reserve1 >= 1_000);

    if (_reserve0 > MAX_UQ112x112 || _reserve1 > MAX_UQ112x112) return;
    vm.assume(_reserve0 <= MAX_UQ112x112);
    vm.assume(_reserve1 <= MAX_UQ112x112);

    vm.assume(_kLast > 0);


    uint rootK = Math.sqrt(_reserve0 * _reserve1);
    uint rootKLast = Math.sqrt(_kLast);

    if (rootK > rootKLast) {

        uint numerator = (rootK - rootKLast) * 1e18;
        uint denominator = (rootK * 5) + rootKLast;
        uint ratio = numerator * 3 / denominator;

        assert(ratio < 0.6 ether); /// @audit reduce_for_counterexample
    }

}
```

This computes a maximum fee ratio of `0.6 ether`, or a upper bound 60% fee, which converts into a maximum deviation of 20%:

$$
(0.6 - 0.5) / 0.5 * 100 = 20%
$$

This closely matches the original UniswapV2 upper bound.
