**Severity**: Medium

# [M-1] Lack of Minimum Position Size Allows Creation of Dust Positions

## Summary
The protocol does not enforce a minimum position size (collateral amount) allowing users to create dust positions with as little as 0.00001 USDC.

## Root Cause
Users can call `AbstractLendingRouter::enterPosition` with a minimal collateral amount, creating positions that are economically meaningless to the protocol. The current validation only checks that the `depositAmount` is greater than zero, and does not implement a minimum collateral threshold. This allows users to create dust positions, which can accumulate and pose economic and technical risks to the protocol.

## Internal Pre-conditions
N/A

## External Pre-conditions
N/A

## Attack Path
N/A

## Impact
The absence of a minimum collateral threshold allows users to create numerous economically insignificant dust positions that can pose certain economic risk such as:

  1. Potential bad debt accumulation - Liquidators may ignore tiny positions since gas costs exceed potential profits.
  2. Potential interest and fee calculation errors - Very small amounts may cause rouding errors and degrade accounting precision.

## PoC
```solidity
    function test_userCanOpenDustPosition() public {  
        uint256 depositAmount = 10; // 0.00001 USDC  
        uint256 borrowAmount = 90; // 0.00009 USDC  
  
        _enterPosition(msg.sender, depositAmount, borrowAmount);  
  
        assert(lendingRouter.balanceOfCollateral(msg.sender, address(y)) > 0);  
    }
```

## Mitigation
Implement a minimum collateral threshold in the AbstractLendingRouter::enterPosition function to ensure that all new positions meet a minimum economic value.

```diff
+  uint256 public minimumCollateral;  
  
     function enterPosition(  
        address onBehalf,  
        address vault,  
        uint256 depositAssetAmount,  
        uint256 borrowAmount,  
        bytes calldata depositData  
    )  
        public  
        override  
        isAuthorized(onBehalf, vault)  
    {  
+        if (depositAssetAmount < minimumCollateral) {  
+            revert DepositBelowMinimumCollateral();  
+        }  
        _enterPosition(onBehalf, vault, depositAssetAmount, borrowAmount, depositData, address(0));  
    }
```
