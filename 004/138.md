Clumsy Fuzzy Monkey

medium

# When flashOn is active, swap fee is applied to (principal + flashFee) instead to principal only

## Summary
When the flashOn feature is active, the swap fee is applied to the sum of the principal and the flashFee, rather than being applied to the principal alone

## Vulnerability Detail

Beside the standard UniV2 swap fee, protocol can [activate](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaFactory.sol#L76) an additional flashFee.
When a flash swap is executed the amountOut is [calculated](https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/JalaPair.sol#L231) as `amountOut * (1 + flashFee)`: 
```solidity
                    amount0Out = (amount0Out * (10000 + IJalaFactory(factory).flashFee())) / 10000;
```

Next, the standard  UniV2 swap fee is applied to this new amount.

```solidity
        uint256 amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
        uint256 amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        if (amount0In == 0 && amount1In == 0) revert InsufficientInputAmount();
        {
            // scope for reserve{0,1}Adjusted, avoids stack too deep errors
            uint256 balance0Adjusted = (balance0 * 1000) - (amount0In * 3);
            uint256 balance1Adjusted = (balance1 * 1000) - (amount1In * 3);
            if (balance0Adjusted * balance1Adjusted < uint256(_reserve0) * uint256(_reserve1) * (1000 ** 2))
                revert InvalidK();
        }
```

The problem lies in the fact 0.3% swap fee is applied to `principal + flashFee` instead of just to the principal. 
For a flashFee = 1%, users will pay 0.3% * 1% = 0.003% more than they should. 
Given the fact flash loans can be used to borrow big amounts of assets (up to the entire pool balance), additional fee paid can be significantly. 

Add the following test in JalaPair.t.sol file 
```solidity
    function test_ExtraFeePaidInFlashloan() public {
        token0.transfer(address(pair), 1 ether);
        token1.transfer(address(pair), 2 ether);
        pair.mint(address(this));

        uint256 flashloanAmount = 1 ether;
        uint256 flashFeeBps = 100;
        uint256 flashloanFee = flashloanAmount * flashFeeBps / 10_000;
        uint256 totalFeesPaid = ((flashloanAmount + flashloanFee) * 1000) / 997 - flashloanAmount + 1;
        uint256 swapFeeAppliedToPrincipal = (flashloanAmount * 1000) / 997 - flashloanAmount + 1;

        Flashloaner fl = new Flashloaner();
        // transfer fees user thinks it has to pay
        token1.transfer(address(fl), swapFeeAppliedToPrincipal + flashloanFee);

        vm.startPrank(feeSetter);
        factory.setFlashOn(true);
        factory.setFeeTo(feeSetter);
        factory.setFlashFee(flashFeeBps);
        vm.stopPrank();
        uint256 feeSetterBefore = token1.balanceOf(feeSetter);

        vm.expectRevert(JalaPair.InvalidK.selector);              
        fl.flashloan(address(pair), 0, flashloanAmount, address(token1));

        // add the difference so user can actually pay for swapFee
        token1.transfer(address(fl), totalFeesPaid - swapFeeAppliedToPrincipal - flashloanFee);
        fl.flashloan(address(pair), 0, flashloanAmount, address(token1));

        assertEq(token1.balanceOf(address(fl)), 0,"1");
        assertEq(token1.balanceOf(address(pair)), 2 ether + totalFeesPaid - flashloanFee, "2");
        assertEq(token1.balanceOf(feeSetter), feeSetterBefore + flashloanFee,"3");
        assertEq(totalFeesPaid - swapFeeAppliedToPrincipal - flashloanFee, 0, "Additional swapFee user paid for a 1ETH flash loan" );
    }
```

## Impact
Users pays more in fees than they should when flash fees are active.

## Code Snippet
Provided in the Details above.

## Tool used

Manual Review

## Recommendation
Consider changing how swap fees are applied . 
Instead applying swapFee to `amountBorrowed + flashLoanFee`, apply swapFee only to `amountBorrowed`. In this way users will pay back the correct amount `amountBorrowed + flashloanFee + swapFee`.

