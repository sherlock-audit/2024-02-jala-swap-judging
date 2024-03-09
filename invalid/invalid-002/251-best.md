Exotic Umber Gerbil

medium

# Contract ChilizWrapperFactory.sol cannot be compiled

## Summary
`ChilizWrapperFactory.sol` inherits `Ownable` but does not pass parameters to the constructor of `Ownable`, causing the contract to fail to compile.

## Vulnerability Detail
```solidity
Error:
Compiler run failed:
Error (3415): No arguments passed to the base constructor. Specify the arguments or mark "ChilizWrapperFactory" as abstract.
  --> contracts/utils/ChilizWrapperFactory.sol:11:1:
   |
11 | contract ChilizWrapperFactory is IChilizWrapperFactory, Ownable {
   | ^ (Relevant source part starts here and spans across multiple lines).
Note: Base constructor parameters:
  --> lib/openzeppelin-contracts/contracts/access/Ownable.sol:38:16:
   |
38 |     constructor(address initialOwner) {
   |                ^^^^^^^^^^^^^^^^^^^^^^

Warning (9302): Return value of low-level calls not used.
  --> contracts/mocks/MockWETH.sol:45:9:
   |
45 |         payable(msg.sender).call{value: wad}("");
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

## Impact
`ChilizWrapperFactory.sol` failed to compile successfully

## Code Snippet
https://github.com/sherlock-audit/2024-02-jala-swap/blob/main/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol#L11-L75

## Tool used
Manual Review

## Recommendation
```diff
diff --git a/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol b/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol
index a2c5bc8..3277f93 100644
--- a/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol
+++ b/jalaswap-dex-contract/contracts/utils/ChilizWrapperFactory.sol
@@ -9,6 +9,9 @@ import {SafeERC20} from "../libraries/SafeERC20.sol";
 import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

 contract ChilizWrapperFactory is IChilizWrapperFactory, Ownable {
+    constructor() Ownable (msg.sender){
+
+    }
     mapping(address => address) public underlyingToWrapped;
     mapping(address => address) public wrappedToUnderlying;
```