Happy Pistachio Deer

medium

# User overpays flashloan fee

## Summary
User pays more fees than required on flashloan
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/JalaPair.sol#L231-L241

## Vulnerability Detail
This is the interface a smart contract implements when he wants to receive a flashloan
```solidity
interface IJalaCallee {
    function JalaCall(address sender, uint256 amount0, uint256 amount1, bytes calldata data) external;
}
```
When new user searches how communicate with the UniswapV2 fork, he will just go to UniswapV2 documentation which [says](https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/using-flash-swaps):

> amount0 and amount1 are simply amount0Out and amount1Out

So `amount0` and `amount1` are just values of tokens user asked for and we transfered.
In Jala Swap we include in those values protocol fees,
Assume protocol fee is 1% 
If user asked 100 tokens in `amount1Out`, we transfer him 100 tokens and in `amount1` we provide him `101`
because we multiply `amount1Out` by protocol flash fee and provide result to `JalaCall()`
```solidity
                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                IJalaCallee(to).JalaCall(msg.sender, amount0Out, amount1Out, data);
```
Now user knows he needs to pay pool fees which 0.3%
Usually user just takes `amount1Out` and adds 0.3% fee, then transfers sum to the Pair contract.
However here they calculate 0.3% not on the amount they received but on amount they got as argument `amount1Out`, which are different.

## Impact
User pays protocol more than it asked, user loses money, feels robbed :), does not want to take more flashloans from protocol.


## Code Snippet
add this test next to other tests in `jalaswap-dex-contract/test/JalaPair.t.sol`
```solidity
function test_FlashloanV2() public {
        token0.transfer(address(pair), 1 ether);
        token1.transfer(address(pair), 2 ether);
        pair.mint(address(this));

        uint256 flashloanAmount = 0.5 ether;
        uint256 poolFee = (flashloanAmount * 1000) / 997 - flashloanAmount + 1;
        uint256 jalaFee = flashloanAmount * (10000 + 100) / 10000 - flashloanAmount; //JalaPair#L239 fee calculation
        uint256 flashloanFee = poolFee + jalaFee; 

        AverageFlashloanerV2 fl = new AverageFlashloanerV2();

        token1.transfer(address(fl), 2 ether);
        uint256 userBalanceBefore = token1.balanceOf(address(fl));

        vm.startPrank(feeSetter);
        factory.setFlashOn(true);
        factory.setFeeTo(feeSetter);
        factory.setFlashFee(100);//1%
        vm.stopPrank();

        fl.flashloan(address(pair), 0, flashloanAmount, address(token1));

        uint256 userBalanceAfter = token1.balanceOf(address(fl));
        uint256 expectedBalance = userBalanceBefore - flashloanFee;
        uint256 valueLoss = expectedBalance - userBalanceAfter;
        console.log("expected fee: %s", flashloanFee);                      //6504513540621866
        console.log("fee paid: %s", userBalanceBefore - userBalanceAfter);  //6519558676028085
        console.log("valueLoss: %s", valueLoss);                            //  15045135406219
    }
```

and the `AverageFlashloanerV2` contract:
```solidity
contract AverageFlashloanerV2 {
    error InsufficientFlashLoanAmount();

    uint256 expectedBalance;

    function flashloan(address pairAddress, uint256 amount0Out, uint256 amount1Out, address tokenAddress) external {
        if (amount0Out > 0) {
            expectedBalance = amount0Out + ERC20(tokenAddress).balanceOf(address(this));
        }
        if (amount1Out > 0) {
            expectedBalance = amount1Out + ERC20(tokenAddress).balanceOf(address(this));
        }
        JalaPair(pairAddress).swap(amount0Out, amount1Out, address(this), abi.encode(tokenAddress));
    }

    function JalaCall(address, uint256 amount0, uint256 amount1, bytes calldata data) public {
        address tokenAddress = abi.decode(data, (address));
        uint256 balance = ERC20(tokenAddress).balanceOf(address(this));

        if (balance != expectedBalance) revert InsufficientFlashLoanAmount();

        // about 0.3% fee, +1 to round up
        uint poolFee = (amount1 * 3) / 997 + 1;//https://solidity-by-example.org/defi/uniswap-v2-flash-swap/
        uint amountToRepay = amount1 + poolFee;

        ERC20(tokenAddress).transfer(msg.sender, amountToRepay);
    }
}
```

## Tool used

Manual Review

## Recommendation
In my opinion average flashloan taker prefers to calculate fees himself, because uniswapV2 and hence its forks expect him to.
I suggest removeing calculation of protocol fees
```diff
            if (IJalaFactory(factory).flashOn() && data.length > 0) {
                if (amount0Out > 0) {
                    _safeTransfer(
                        _token0,
                        IJalaFactory(factory).feeTo(),
                        (amount0Out * IJalaFactory(factory).flashFee()) / 10000
                    );
-                    amount0Out = (amount0Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                if (amount1Out > 0) {
                    _safeTransfer(
                        _token1,
                        IJalaFactory(factory).feeTo(),
                        (amount1Out * IJalaFactory(factory).flashFee()) / 10000
                    );
-                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
                }
                IJalaCallee(to).JalaCall(msg.sender, amount0Out, amount1Out, data);
            }
```
Or less preferred aproach is to calculate all fees and give it to user
```diff
            if (IJalaFactory(factory).flashOn() && data.length > 0) {
                if (amount0Out > 0) {
                    _safeTransfer(
                        _token0,
                        IJalaFactory(factory).feeTo(),
                        (amount0Out * IJalaFactory(factory).flashFee()) / 10000
                    );
-                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
+                    uint poolFee = (amount0Out * 3) / 997 + 1;
+                    amount0Out = (amount0Out * (10000 + IJalaFactory(factory).flashFee())) / 10000  + poolFee;
                }
                if (amount1Out > 0) {
                    _safeTransfer(
                        _token1,
                        IJalaFactory(factory).feeTo(),
                        (amount1Out * IJalaFactory(factory).flashFee()) / 10000
                    );
-                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
+                    uint poolFee = (amount1Out * 3) / 997 + 1;
+                    amount1Out = (amount1Out * (10000 + IJalaFactory(factory).flashFee())) / 10000  + poolFee;
                }
                IJalaCallee(to).JalaCall(msg.sender, amount0Out, amount1Out, data);
            }
```