**Severity**: Medium 

## [M-1] Liquidation Risk for Users Due to Paused State

## Summary 
The `whenPaused` modifier may block debt repayment during a paused state, potentially exposing borrowers to liquidation.

## Finding Description 
In the `HoldingManager` contract, the `pause` function allows the owner to activate a paused state. While the contract is paused, critical functions such as `repay` are disabled due to the use of the `whenNotPaused` modifier. This prevents users from repaying their outstanding debt during the paused period.

As a result, borrowers are exposed to an increased risk of liquidation, particularly during periods of high market volatility. If the value of their collateral drops significantly while the contract is paused, they may be unable to repay in time to avoid liquidation.

## Impact Explanation 
The inability of users to repay their `jUsd` debt when the contract is paused, exposes borrowers to a high risk of forced liquidation, even when they are willing and able to repay. By preventing timely repayment, the protocol increases the likelihood of user liquidations, potentially resulting in unnecessary loss of user funds.

## Likelihood Explanation 
The likelihood of this issue occurring is moderate to high, depending on how frequently the pause functionality is used.

## Proof of Concept
Consider the following scenario:

- A user has an outstanding jUsd loan position managed by the HoldingManager contract.
- The contract owner triggers the pause function to temporarily halt protocol operations.
- Due to market volatility, the user's collateral value begins to fall, putting the position at risk of liquidation.
- The user attempts to repay their debt to restore solvency and avoid liquidation.
- The transaction reverts because the HoldingManager contract is paused by the owner.
- The user is now unable to repay their debt, even though they have the funds and intent to do so.
- The contract owner triggers the unpause function to allow normal protocol operation.
- The user's position is liquidatable and gets liquidated.
- The user suffers a loss that could have been avoided if repayment were permitted during the pause.

Add the following code to the HoldingManager.t.sol file and run the test: 

```solidity 
    function test_userCannotRepayWhenContractIsPaused() public {
        address user = makeAddr("user");
        uint256 depositAmount = 10_000e18;
        uint256 borrowAmount = 2000e18;

        deal(address(usdc), user, depositAmount);

        vm.startPrank(user, user);
        holdingManager.createHolding();
        usdc.approve(address(holdingManager), depositAmount);
        holdingManager.deposit(address(usdc), depositAmount);
        holdingManager.borrow(address(usdc), borrowAmount, 0, true);
        vm.stopPrank();

        vm.prank(OWNER);
        stablesManager.pause();

        vm.warp(block.timestamp + 1 days);

        vm.startPrank(user);
        vm.expectRevert();
        holdingManager.repay(address(usdc), borrowAmount, true);
    }
```

## Recommendation
To mitigate this issue, consider removing the `whenNotPaused` modifier from the `HoldingManager::repay` function. This will allow users to repay their debt when the contract is paused thereby reducing the risk of forced liquidation during critical periods.

Alternatively, implement a separate emergency repayment mechanism within the `HoldingManager` contract, that remains accessible to users even when the contract is paused.