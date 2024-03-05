Amusing Fern Wolf

medium

# Since `kLast` is updated after updating current reserves, `_mintFee` function of `JalaPair` cannot mint fee tokens.

## Summary

`_mintFee` function of `JalaPair.sol` calculates the fee amount of liqudity token based on the difference btw last and current reserves. However, since the updating `kLast` is incorrect, `_mintFee` cannot mint fee liquidity.

## Vulnerability Detail

`JalaPair` contract mints liquidity based on growth of `sqrt(reserve0 * reserve1)` as fee whenever `mint` and `burn` actions are performed.
```solidity
    function mint(address to) external lock returns (uint256 liquidity) {
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0 = balance0 - _reserve0;
        uint256 amount1 = balance1 - _reserve1;

@>      bool feeOn = _mintFee(_reserve0, _reserve1);

        // ...
```
```solidity
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo = IJalaFactory(factory).feeTo(); // get feeTo address
        feeOn = feeTo != address(0); // @audit in the case of address DEAD?
        uint256 _kLast = kLast; // gas savings
        if (feeOn) {
            if (_kLast != 0) {
                uint256 rootK = Math.sqrt(uint256(_reserve0) * _reserve1);
                uint256 rootKLast = Math.sqrt(_kLast);
@>              if (rootK > rootKLast) {
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
In `_mintFee` function, `rootK` is calculated based on current reserves and `rootKLast` is calculated based on last reserves.
```solidity
        _mint(to, liquidity);

@>      _update(balance0, balance1, _reserve0, _reserve1);
@>      if (feeOn) kLast = uint256(reserve0) * reserve1; // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```
As you can see in above codebase, `kLast` is updated after the reserves of pair are updated based on new mint/burn actions. Therefore, at the next mint/burn actions, `rootK` and `rootKLast` in `_mintFee` functions will be the same value.
This means that `_mintFee` always calculates the liqudity of fee as 0.

Following is the test file for PoC:
```solidity
function test_mintNoFee() public {
        // set the fee recepient address
        vm.prank(feeSetter);
        factory.setFeeTo(feeSetter);

        // mint tokens A, B to Alice
        address Alice = makeAddr("Alice");
        tokenA.mint(1 ether, Alice);
        tokenB.mint(1 ether, Alice);
        
        // add liquidity to pair of token A and B
        vm.startPrank(Alice);
        tokenA.approve(address(router), 0.1 ether);
        tokenB.approve(address(router), 0.1 ether);

        router.addLiquidity(
            address(tokenA),
            address(tokenB),
            0.01 ether,
            0.01 ether,
            0.01 ether,
            0.01 ether,
            Alice,
            block.timestamp
        );
        vm.stopPrank();

        address pair = factory.getPair(address(tokenA), address(tokenB));

        assertEq(IJalaPair(pair).balanceOf(Alice), 0.01 ether - 1000);

        assertEq(IJalaPair(pair).balanceOf(feeSetter), 0);

        (uint256 reserve0, uint256 reserve1, ) = IJalaPair(pair).getReserves();

        assertEq(IJalaPair(pair).balanceOf(factory.feeTo()), 0); // fee balance is 0 after minting

        vm.startPrank(Alice);
        router.addLiquidity(
            address(tokenA),
            address(tokenB),
            0.01 ether,
            0.01 ether,
            0.01 ether,
            0.01 ether,
            Alice,
            block.timestamp
        );
        vm.stopPrank();

        assertEq(IJalaPair(pair).balanceOf(Alice), 0.02 ether - 1000);

        assertEq(IJalaPair(pair).balanceOf(factory.feeTo()), 0); // fee balance is 0 after second minting

        vm.startPrank(Alice);
        IJalaPair(pair).approve(address(router), 0.1 ether);
        router.removeLiquidity(
            address(tokenA),
            address(tokenB),
            0.01 ether,
            0.01 ether - 1000,
            0.01 ether - 1000,
            Alice,
            block.timestamp
        );
        vm.stopPrank();

        assertEq(IJalaPair(pair).balanceOf(factory.feeTo()), 0); // fee balance is 0 after burning
    }
```

## Impact

`JalaPair` always calculates the fee liquditity incorrectly as 0, so `feeTo` cannot receive fee.

## Code Snippet

https://github.com/sherlock-audit/2024-02-jala-swap/blob/b43ff651f431e97dae2a6c96f4e7e1b06284ec10/jalaswap-dex-contract/contracts/JalaPair.sol#L155-L156
https://github.com/sherlock-audit/2024-02-jala-swap/blob/b43ff651f431e97dae2a6c96f4e7e1b06284ec10/jalaswap-dex-contract/contracts/JalaPair.sol#L110-L129

## Tool used

Manual Review

## Recommendation

Update the `mint` and `burn` function of `JalaPair.sol` as follow:
```diff
        _mint(to, liquidity);

+       if (feeOn) kLast = uint256(reserve0) * reserve1;
        _update(balance0, balance1, _reserve0, _reserve1);
-       if (feeOn) kLast = uint256(reserve0) * reserve1; // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
```
Or following is also possible:
```diff
        _mint(to, liquidity);

        _update(balance0, balance1, _reserve0, _reserve1);
-       if (feeOn) kLast = uint256(reserve0) * reserve1; // reserve0 and reserve1 are up-to-date
+       if (feeOn) kLast = uint256(_reserve0) * _reserve1;
        emit Mint(msg.sender, amount0, amount1);
```
In addition, even after updating `JalaPair.sol` like above, there maybe small issue in `_mintFee` function.
You can know the issue when you perform the test function again after updating, the mint fee would still be calculated as 0 even when the second minting is performed. This is due to following condition in `_mintFee` funciton.
```solidity
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo = IJalaFactory(factory).feeTo(); // get feeTo address
        feeOn = feeTo != address(0); // @audit in the case of address DEAD?
        uint256 _kLast = kLast; // gas savings
        if (feeOn) {
@>          if (_kLast != 0) {
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
If it is not the developer's intention, I think it would be better to remove the condition.
