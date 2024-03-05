Active Burgundy Ape

medium

# Migration of liquidity  can be DOSed to never happen

## Summary
The pair can be manipulated to a state where its not possible for migraters to mint liquidity

## Vulnerability Detail

```solidity
   function mint(address to) external lock returns (uint256 liquidity) {
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0 = balance0 - _reserve0;
        uint256 amount1 = balance1 - _reserve1;

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint256 _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
🟥🟥       if (IJalaFactory(factory).migrators(msg.sender)) {
                liquidity = IMigrator(msg.sender).desiredLiquidity();
                if (liquidity == 0 || liquidity == type(uint).max) revert BadDesiredLiquidity();
            } else {
                liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
                _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
            }
        } else {
            liquidity = Math.min((amount0 * _totalSupply) / _reserve0, (amount1 * _totalSupply) / _reserve1);
        }
        if (liquidity == 0) revert InsufficientLiquidityMinted();
        _mint(to, liquidity);

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint256(reserve0) * reserve1; // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```

So two issues,
1. DOS to migraters
2. migrater has the power to do inflation attack. Migrator is not a trusted role according to the readme of the contest.

The migraters have the power to mint desired liquidity first time or whenever the `totalsupply` of the pair is 0.
The issue here is, when the factory gets deployed, since we know the token addresses in the chain, we will backrun the factory deployment and start creating pairs for those tokens. And will mint just 1 wei worth of initial liquidity.

So now `_totalSupply` is > 0, so migraters can never mint their desired liquidity, which dictates the share price of the pair.
And another issue is, since we mint desired liquidity to migrater, the migrater has the chance to do infaltion attack, because `MINIMUM_LIQUIDITY` was never minted. So any malicious migrater can cause inflation attack to the next LP after they mint.


## Impact
Its not possible for migraters to mint the desired liquidity

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaPair.sol#L141-L153

## Tool used

Manual Review

## Recommendation

1. Mint initial liquidity to `_mint(address(0), MINIMUM_LIQUIDITY);`, to avoid inflation attack.
2. And for migrator to always mint desired liquidity, move the code block to not only mint when totalsupply = 0, but whenever.


```diff

   function mint(address to) external lock returns (uint256 liquidity) {
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0 = balance0 - _reserve0;
        uint256 amount1 = balance1 - _reserve1;

        bool feeOn = _mintFee(_reserve0, _reserve1);
        uint256 _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee

+       if (IJalaFactory(factory).migrators(msg.sender)) {
+           liquidity = IMigrator(msg.sender).desiredLiquidity();
+           if (liquidity == 0 || liquidity == type(uint).max) revert BadDesiredLiquidity();
+       }

        if (_totalSupply == 0) {
-          if (IJalaFactory(factory).migrators(msg.sender)) {
-               liquidity = IMigrator(msg.sender).desiredLiquidity();
-               if (liquidity == 0 || liquidity == type(uint).max) revert BadDesiredLiquidity();
-           } else {
-               liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
-               _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
-           }

+           if (IJalaFactory(factory).migrators(msg.sender)) liquidity -= MINIMUM_LIQUIDITY;
+           else liquidity = Math.sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
+           _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens

        } else {
            liquidity = Math.min((amount0 * _totalSupply) / _reserve0, (amount1 * _totalSupply) / _reserve1);
        }
        if (liquidity == 0) revert InsufficientLiquidityMinted();
        _mint(to, liquidity);

        _update(balance0, balance1, _reserve0, _reserve1);
        if (feeOn) kLast = uint256(reserve0) * reserve1; // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```
