Odd Holographic Carp

medium

# `ChilizWrappedERC20` is vulnerable to address collision

## Summary
The factory `ChilizWrapperFactory` creates a new `ChilizWrappedERC20` using `CREATE2`. A meet-in-the-middle attack at finding an address collision against an undeployed wrapped ERC20 is possible. Such attack could set attacker allowance of the underlying token to the max value and drain the wrapper.
## Vulnerability Detail
Before going in details, this issue idea is from [EIP-3607](https://eips.ethereum.org/EIPS/eip-3607).

Past issues with a similar approach:
- https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90
- https://github.com/sherlock-audit/2023-12-arcadia-judging/issues/59

To carry out this attack, one must find a collision (or more) and then create a malicious smart contract that sets the attacker's wallet allowance to the maximum on the underlying token.

In `ChilizWrapperFactory::_createWrappedToken` the salt is the result of a common encoding of the underlying token address: `bytes32 salt = keccak256(abi.encode(underlyingToken));` 

https://github.com/sherlock-audit/2024-02-jala-swap/blob/030d3ed54214754301154bce0e58ea534100a7e3/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol#L34
```solidity
assembly {
            wrappedToken := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
```

The attacker will need to find a collision of at least one of the many ERC20 Fan tokens (any token actually but as those are expected, risk is higher for them) that will be wrapped in this factory.

Then, the user will need to deploy its malicious smart contract with `CREATE2`, changing the salt value until a collision is found.

The following articles describe the feasibility of finding a collision: 
- https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses
- https://kevingal.com/blog/collisions.html

At the time of this writing, there are 82 fan tokens available on Chiliz Chain and 1,332,095,893 USD of market cap. The current market data and the "low" cost of execution ( [just a few million dollars](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses)), make this attack a really profitable strategy not only for existing tokens, but also for future new tokens. 

It is then easy to show that such an attack is massively profitable.

This type of vulnerability has been extensively discussed in the two issues I mentioned in the beggining.
## Impact

Draining of underlying assets from wrapped ERC20 contracts. Denial-of-Service of each `JalaPair` associated with these wrapped token.
## Code Snippet

The idea behind this PoC was taken from this [issue](https://github.com/sherlock-audit/2023-12-arcadia-judging/issues/59) found in the Arcadia audit at Sherlock.

First,
- Deploy the attack contract onto collided address.
- Set infinite allowance for the underlying token to the attacker wallet.
- Destroy the contract with `selfdestruct`. See [this](https://eips.ethereum.org/EIPS/eip-6780).

Then, 
- Wait till the TVL on the contract is considered profitable to extract.
- Transfer all underlying tokens to the attacker wallet.

All funds have been stolen and every `JalaPair` with a wrapped token from the factory has been DoSed.
## Tool used

Manual Review

## Recommendation

Do not use the underlying token as `salt`. Use the `CREATE` contract creation which relies on the address of the factory and its internal nonce. This will make the resulting address almost unpredictable so attempting a collision will be impractical.

To replace the `ChilizWrapperFactory::wrappedTokenFor` function use in `JalaMasterRouter`, I recommend using `ChilizWrapperFactory::getUnderlyingToWrapped` that has the practical same functionality.  
