**Severity**: Medium

# [M-1] Insolvency Risk Due To Inability To Liquidate Insolvent Positions

## Summary
Failure to implement any mechanism that allows the liquidation of insolvent positions, leaves the protocol with bad debt, as liquidation reverts if the borrower's collateral is insufficient to cover the seized tokens.

## Finding Description 
In the `PTokenModule::liquidateBorrowFresh` function, the contract enforces a check requiring the borrower’s collateral balance to be greater than or equal to the `seizeTokens` amount. This check ensures that a liquidator can repay a borrower's debt and seize an equivalent amount of collateral, including an incentive.

However, in scenarios involving sharp price declines, the value of the borrower’s collateral may fall below the amount needed to cover the debt. In such cases, the liquidation transaction reverts, preventing the liquidation of the insolvent position. As a result, the protocol is left with uncollectible bad debt.

## Impact Explanation 
The inability to liquidate insolvent positions leads to the accumulation of bad debt, which gradually erodes the protocol’s available liquidity. This weakens the overall stability of the protocol and may result in insufficient liquidity for borrowers and liquidity providers attempting to withdraw or redeem their funds.

## Likelihood Explanation 
The likelihood of this issue occurring is medium to high especially in periods of high market volatility where collateral assets can rapidly lose value before liquidation can be triggered.

## Proof of Concept 
Add this test function to `PToken.t.sol`.

```solidity
    function testCannotLiquidateInsolventPosition() public {
        address user = makeAddr("user");
        address depositor = makeAddr("depositor");
        address liquidator = makeAddr("liquidator");

        doDeposit(depositor, depositor, address(pUSDC), 2000e6);

        doDepositAndEnter(user, user, address(pWETH), 1e18);
        doBorrow(user, user, address(pUSDC), 1450e6);

        mockOracle.setPrice(address(pWETH), 500e6, 18);

        LiquidationParams memory lp = LiquidationParams({
            prankAddress: liquidator,
            userToLiquidate: user,
            collateralPToken: address(pWETH),
            borrowedPToken: address(pUSDC),
            repayAmount: 500e6,
            expectRevert: true,
            error: abi.encodePacked(bytes4(0x430ffe6f))
        });

        doLiquidate(lp);
    }
```

## Recommendation
Consider implementing an `onlyOwner` function that allows liquidation of insolvent positions without any liquidation incentive, when the borrower's collateral is insufficient to cover the amount owed.
