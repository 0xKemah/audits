**Severity**: High

## [H-1] Missing Update for totalSyntheticsIssued in `AlchemistV3::repay` function

## Summary
The `AlchemistV3::repay` function fails to update the `totalSyntheticsIssued` variable to reflect the amount of debt repaid by users.

## Description
When a user calls the `AlchemistV3::repay` function, the repaid amount of debt tokens is correctly deducted from both the user’s individual debt balance and the protocol’s total debt. The state variable `totalSyntheticsIssued` is updated whenever users mint new synthetic debt tokens, serving as a record of the total synthetic debt issued by the protocol. However, the `repay` function does not update `totalSyntheticsIssued` to reflect debt repayments, causing a mismatch between the recorded synthetic debt and the actual outstanding debt.

## Impact Explanation
Failure to update `totalSyntheticsIssued` during repayments causes the protocol to overstate the total synthetic debt issued. Since `totalSyntheticsIssued` is used to calculate the protocol’s `badDebtRatio` in the `Transmuter` contract during user collateral redemptions, this discrepancy leads to an inaccurate `badDebtRatio` calculation, potentially impacting redemption logic and protocol risk assessments.

## Likelihood Explanation
This issue is likely to occur whenever users repay their debt using the `repay` function, making it a frequently occuring issue.

## Proof of Concept
- Add the following code to the `AlchemistV3.t.sol` file and run the `forge` test.

```solidity
    function test_totalSyntheticsIssuedIsNotUpdated() public {
        uint256 amount = 100e18;

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);

        // a single position nft would have been minted to 0xbeef
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));

        // get totalSyntheticsIssued before debt repayment
        uint256 totalSyntheticsIssuedBefore = alchemist.totalSyntheticsIssued();

        vm.roll(block.number + 1);

        // user repays debt
        alchemist.repay(100e18, tokenId);
        vm.stopPrank();

        // get totalSyntheticsIssued after debt repayment
        uint256 totalSyntheticsIssuedAfter = alchemist.totalSyntheticsIssued();

        (, uint256 userDebt, ) = alchemist.getCDP(tokenId);

        assertEq(userDebt, 0);

        // confirm that totalSyntheticsIssued does not change
        assertEq(totalSyntheticsIssuedBefore, totalSyntheticsIssuedAfter);
    }
```
## Recommendation
Update the `totalSyntheticsIssued` state variable within the `AlchemistV3::repay` function to correctly reflect the reduction in debt when users repay. This ensures accurate tracking of total synthetic tokens issued, maintains consistency in protocol accounting, and prevents miscalculations in metrics like the `badDebtRatio`.

Consider making the following changes in `AlchemistV3::repay` function:

```diff
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        ...

        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        // Debt is subject to protocol fee similar to redemptions
        account.collateralBalance -= (creditToYield * protocolFee) / BPS;

        _subDebt(recipientTokenId, credit);
+       totalSyntheticsIssued -= credit;

        ...
    }
```

## [H-2] `yieldToken` Repayment Incorrectly Transferred to `AlchemixV3` Instead of `Transmuter`

## Summary
In the `AlchemixV3::_forceRepay` function, the repaid `yieldToken` amount is incorrectly transferred to the `AlchemixV3` contract instead of the `Transmuter`, which is the intended recipient.

## Description
During the liquidation of a user's position, the `AlchemixV3::_forceRepay` function is invoked if the user has an `earmarked` debt. Within this function, `yieldTokens` equivalent to the user's debt are incorrectly transferred from the user to `address(this)`, which refers to the `AlchemixV3` contract itself. As a result, the `Transmuter` contract does not receive the intended `yieldToken` repayment, potentially leaving it underfunded.

This issue can be seen as shown in the code snippet below:

```solidity
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        account.earmarked -= credit > account.earmarked ? account.earmarked : credit;
        account.collateralBalance -= creditToYield;

        // Transfer the repaid tokens from the account to the transmuter.
@>      TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
        return creditToYield;
    }
```

## Impact Explanation 
This misdirection of `yieldToken` repayments causes the `Transmuter` contract to receive less than the intended amount of tokens, effectively disrupting the protocol's accounting.

## Likelihood Explanation 
This issue is likely to occur during liquidations involving users with `earmarked` debt, as the _forceRepay function is specifically triggered in that scenario. Since earmarked debt can exist under normal usage, the likelihood of occurrence is high.

## Proof of Concept
- Add the following code to `AlchemistV3.t.sol` file and run the `forge` test.

```solidity
    function test_forceRepayDoesNotSendRepaidYieldTokensToTransmuter() external {
        uint256 amount = 200_000e18;
        address bob = makeAddr("bob");
        address alice = makeAddr("alice");
        address _liquidator = makeAddr("liquidator");
        deal(address(fakeYieldToken), bob, amount * 2);
        deal(address(fakeYieldToken), alice, amount * 2);

        vm.prank(someWhale);
        fakeYieldToken.mint(whaleSupply, someWhale);

        vm.startPrank(alice);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, alice, 0);
        vm.stopPrank();

        vm.startPrank(bob);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, bob, 0);

        uint256 tokenIdForBob = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));
        uint256 amountToMint = (alchemist.totalValue(tokenIdForBob) * FIXED_POINT_SCALAR) / minimumCollateralization;
        alchemist.mint(tokenIdForBob, amountToMint, bob);

        uint256 amountToEarmark = IERC20(alToken).balanceOf(bob);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), amountToEarmark);
        transmuterLogic.createRedemption(amountToEarmark);
        vm.stopPrank();

        alchemist.poke(tokenIdForBob);
        vm.roll(block.number + 5_256_000);

        // modify yield token price via modifying underlying token supply
        uint256 initialSupplyOfYieldToken = IERC20(address(fakeYieldToken)).totalSupply();
        fakeYieldToken.updateMockTokenSupply(initialSupplyOfYieldToken);

        // increasing yield token suppy by 59 bps or 5.9%  while keeping the underlying supply unchanged
        uint256 modifiedYieldTokenSupply = ((initialSupplyOfYieldToken * 590) / 10_000) + initialSupplyOfYieldToken;
        fakeYieldToken.updateMockTokenSupply(modifiedYieldTokenSupply);

        // get transmuter balance before liquidation
        uint256 transmuterBalanceBeforeLiquidation = fakeYieldToken.balanceOf(address(transmuterLogic));

        // liquidator calls liquidate on bob's position
        vm.startPrank(_liquidator);
        alchemist.liquidate(tokenIdForBob);
        vm.stopPrank();

        // get transmuter balance after liquidation
        uint256 transmuterBalanceAfterLiquidation = fakeYieldToken.balanceOf(address(transmuterLogic));

        // confirm that transmuter balance of yieldTokens does not change after liquidation
        assertEq(transmuterBalanceAfterLiquidation, transmuterBalanceBeforeLiquidation);
    }
```
## Recommendation
Implement the following changes to the `Alchemix::_forceRepay` function:

```diff
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        account.earmarked -= credit > account.earmarked ? account.earmarked : credit;
        account.collateralBalance -= creditToYield;

        // Transfer the repaid tokens from the account to the transmuter.
-       TokenUtils.safeTransfer(yieldToken, address(this), creditToYield);
+       TokenUtils.safeTransfer(yieldToken, transmuter, creditToYield);
        return creditToYield;
    }
```

## [H-3] Protocol Fee Not Transferred During Token Burn in `AlchemixV3.sol`

## Summary
When users call the `burn` function in the `AlchemixV3` contract, the protocol fee deducted from their collateral is not transferred to the `protocolFeeReceiver`.

## Description
The `protocolFeeReceiver` is not credited with the protocol fee when users call the `AlchemixV3::burn` function. Instead, the fee deducted from their collateral is trapped in the `AlchemixV3` contract. Over time, this leads to a significant loss of revenue for the protocol, as the intended fee recipient is never credited.

This issue can be seen in the code snippet below:

```solidity
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        ...
        TokenUtils.safeBurnFrom(debtToken, msg.sender, credit);

@>      _accounts[recipientId].collateralBalance -= (convertDebtTokensToYield(credit) * protocolFee) / BPS;
@>      // @audit missing transfer to protocolFeeReceiver
        _subDebt(recipientId, credit);
        totalSyntheticsIssued -= credit;

        emit Burn(msg.sender, credit, recipientId);

        return credit;
    }
```
## Impact Explanation 
Failure to transfer the protocol fee to the `protocolFeeReceiver` results in the protocol not collecting its intended fee from user interactions with the `burn` function. This leads to a substantial shortfall in protocol revenue.

## Likelihood Explanation 
The likelihood is high, as this issue occurs everytime a user calls the `burn` function. This makes it consistently reproducible under normal protocol usage.

## Proof of Concept
- Add the following code to AlchemistV3.t.sol file and run the forge test

```solidity
    function test_feeNotTransferredToProtocolFeeReceiver() external {
        // Initialize alice with 10_000 yield tokens
        address alice = makeAddr("alice");
        uint256 _depositAmount = 10_000e18;
        uint256 _borrowAmount = 5_000e18;
        deal(address(fakeYieldToken), alice, depositAmount);

        // Set protocol fee to 10%
        vm.prank(alOwner);
        alchemist.setProtocolFee(1000); // 10% fee

        // Alice deposits 10_000 yield tokens and borrows 5000 alTokens
        vm.startPrank(alice);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), _depositAmount);
        alchemist.deposit(_depositAmount, alice, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(alice, address(alchemistNFT));
        alchemist.mint(tokenId, _borrowAmount, alice);

        // Alice burns all alTokens and pays protocol fees
        vm.roll(block.number + 10);
        SafeERC20.safeApprove(address(alToken), address(alchemist), _borrowAmount);
        alchemist.burn(_borrowAmount, tokenId);
        vm.stopPrank();

        // confirm that balance of protocol fee receiver is 0
        assertEq(fakeYieldToken.balanceOf(alchemist.protocolFeeReceiver()), 0);
    }
```
## Recommendation
To mitigate this issue, make the following changes to the `AlchemixV3::burn` function:

```diff
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        ...
        TokenUtils.safeBurnFrom(debtToken, msg.sender, credit);

        _accounts[recipientId].collateralBalance -= (convertDebtTokensToYield(credit) * protocolFee) / BPS;
+       TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, ((convertDebtTokensToYield(credit) * protocolFee) / BPS));     
    }
```

## [H-4] Fee Bonus not Normalized to Underlying Token Decimal Precision

## Summary
The amount of underlying tokens sent to the `liquidator` during the liquidation of a user's position is not normalized to the `underlyingToken` decimal precision, resulting in an overestimation of the `feeBonus`.

## Description
According to the protocol documentation, `USDC`, which has a decimal precision of six (6), is one of the underlying tokens used in the protocol. Therefore, `alTokens`, which have eighteen (18) decimals by design, must be divided by a `conversion-factor` of `1e12` to obtain their equivalent amount in the `underlyingToken` (USDC).

However, in the `AlchemixV3::_liquidate` function, the `feeBonus`, calculated as a percentage of the `debtToken` amount, is not normalized to match the underlyingToken's decimal precision. This oversight leads to an overestimation of the `feeBonus` which can potentially result in a significant financial loss for the protocol.

This can be seen in the code snippet below:

```solidity
    function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        ...

        uint256 feeBonus = (debtToBurn * liquidatorFee) / BPS;

        ...
        if (feeBonus > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                // @audit-H the feeBonus is not normalized to underlying tokens
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
                IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
            }
        }
        ...
    }
```

## Impact Explanation 
The lack of normalization of the `feeBonus` to the `underlyingToken` decimal precision results in an inflated `feeBonus` amount, which leads to liquidators getting much more underlyingTokens than intended. This causes a significant financial loss for the protocol while enriching liquidators.

## Likelihood Explantion 
This issue is likely to occur whenever a liquidation involves an `underlyingToken` with different decimal precision from the `debtToken`. Since the `feeBonus` calculation does not account for the required normalization, it will consistently overestimate the bonus unless the underlying token's decimal precision is specifically handled. Therefore, the likelihood of this issue impacting liquidations involving USDC or similar tokens with non-eighteen decimal precision is high.

## Proof of Concept 
- Add the following code to `AlchemistV3.t.sol` file and run the `forge` test

```solidity
    function test_feeBonusNotNormalizedCorrectly() external {
        uint256 amount = 200_000e6; // 200,000 usdc
        address bob = makeAddr("bob");
        address alice = makeAddr("alice");
        address _liquidator = makeAddr("liquidator");
        deal(address(fakeYieldToken), bob, amount * 2);
        deal(address(fakeYieldToken), alice, amount * 2);

        vm.prank(someWhale);
        fakeYieldToken.mint(whaleSupply, someWhale);

        vm.startPrank(alice);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, alice, 0);
        vm.stopPrank();

        vm.startPrank(bob);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), amount * 2);
        alchemist.deposit(amount, bob, 0);

        uint256 tokenIdForBob = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));
        uint256 amountToMint = (alchemist.totalValue(tokenIdForBob) * FIXED_POINT_SCALAR) / minimumCollateralization;
        alchemist.mint(tokenIdForBob, amountToMint, bob);
        vm.stopPrank();

        // modify yield token price via modifying underlying token supply
        (, uint256 prevDebt, ) = alchemist.getCDP(tokenIdForBob);
        uint256 initialSupplyOfYieldToken = IERC20(address(fakeYieldToken)).totalSupply();
        fakeYieldToken.updateMockTokenSupply(initialSupplyOfYieldToken);

        // increasing yield token suppy by 59 bps or 5.9%  while keeping the underlying supply unchanged
        uint256 modifiedYieldTokenSupply = ((initialSupplyOfYieldToken * 590) / 10_000) + initialSupplyOfYieldToken;
        fakeYieldToken.updateMockTokenSupply(modifiedYieldTokenSupply);

        vm.startPrank(_liquidator);
        assertEq(IERC20(fakeYieldToken).balanceOf(_liquidator), 0);
        assertEq(IERC20(fakeUnderlyingToken).balanceOf(_liquidator), 0);

        uint256 alchemistCurrentCollateralization = (alchemist.normalizeUnderlyingTokensToDebt(
            alchemist.getTotalUnderlyingValue()
        ) * FIXED_POINT_SCALAR) / alchemist.totalDebt();
        (, uint256 expectedDebtToBurn, ) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdForBob),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );

        uint256 expectedFeeInUnderlying = alchemist.normalizeDebtTokensToUnderlying(
            (expectedDebtToBurn * liquidatorFeeBPS) / 10_000
        );

        (, , uint256 feeInUnderlying) = alchemist.liquidate(tokenIdForBob);
        vm.stopPrank();

        console.log("Expected fee in underlying token: ", expectedFeeInUnderlying);
        console.log("Fee in underlying token: ", feeInUnderlying);

        // confirm that the expected fee is not the actual fee
        assert(expectedFeeInUnderlying != feeInUnderlying);
        assert(
            (IERC20(fakeUnderlyingToken).balanceOf(_liquidator) / alchemist.underlyingConversionFactor()) ==
                expectedFeeInUnderlying
        );
    }
```

## Recommendation
To resolve this issue, ensure that the `feeBonus` calculation in the `AlchemixV3::_liquidate` function includes normalization of the `feeBonus` to the decimal precision of the `underlyingToken`. This adjustment will prevent overestimation of the `feeBounus`.

Consider implementing the following changes in `AlchemixV3::_liquidate` function:

```diff
    function _liquidate(uint256 accountId) internal returns (uint256 debtAmount, uint256 feeInYield, uint256 feeInUnderlying) {
        ...
        if (feeBonus > 0) {
            uint256 vaultBalance = IFeeVault(alchemistFeeVault).totalDeposits();
            if (vaultBalance > 0) {
                // @audit-H the feeBonus is not normalized to underlying tokens
                feeInUnderlying = vaultBalance > feeBonus ? feeBonus : vaultBalance;
-               IFeeVault(alchemistFeeVault).withdraw(msg.sender, feeInUnderlying);
+               IFeeVault(alchemistFeeVault).withdraw(msg.sender, normalizeDebtTokensToUnderlying(feeInUnderlying));
            }
        }
        ...
    }
```


## [H-5] Incorrect Protocol Fee Amount Transferred to `protocolFeeReceiver`

## Summary
In the `AlchemistV3::repay` function, the total amount of `yieldTokens` repaid by the user is incorrectly transferred as protocol fee to the `protocolFeeReceiver`.

## Description 
According to the protocol design, only a specific percentage of the repaid `yieldTokens` should be collected as protocol fee. However, in the `AlchemistV3::repay` function, the entire amount of `yieldTokens` repaid by the user is transferred to the `protocolFeeReceiver`, resulting in the protocol retaining fewer tokens than intended.

The issue can be seen in this code snippet below:

```solidity
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        ...
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        // Debt is subject to protocol fee similar to redemptions
        account.collateralBalance -= (creditToYield * protocolFee) / BPS;
        _subDebt(recipientTokenId, credit);

        // Transfer the repaid tokens to the transmuter.
        TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);
@>      TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
        ...
    }
```
While the correct protocol fee is deducted from the user's collateral, an incorrect amount is sent to the `protocolFeeReceiver` as protocol fee.

## Impact Explanation 
This issue causes the protocol to be left with no `yieldToken`s, as the full repayment amount is sent to the `protocolFeeReceiver` as protocol fee. Consequently, the protocol’s `yieldToken` balance is lower than expected, which could negatively impact liquidity, accounting, and the overall stability of the protocol.

## Likelihood Explanation 
This issue arises from a direct misallocation of the protocol fee in the `AlchemistV3::repay` function. Hence, can be triggered during normal operation whenever users repay debt, making it highly likely to occur.

## Proof of Concept
- Add the following code to AlchemistV3.t.sol file and run the forge test

```solidity
    function test_IncorrectFeeAmountSentToProtocolFeeReceiver() external {
        // initialize bob's wallet with 10_000 yield tokens
        address bob = makeAddr("bob");
        uint256 _depositAmount = 5000e18;
        uint256 _borrowAmount = 4000e18;
        deal(address(fakeYieldToken), bob, 10000e18);

        vm.prank(alOwner);
        alchemist.setProtocolFee(500); // 10%

        // bob deposits yield tokens and borrows debt tokens
        vm.startPrank(bob);
        SafeERC20.safeApprove(address(fakeYieldToken), address(alchemist), type(uint256).max);
        alchemist.deposit(_depositAmount, bob, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(bob, address(alchemistNFT));
        alchemist.mint(tokenId, _borrowAmount, bob);

        // repay debt with yield tokens
        vm.roll(block.number + 10);
        uint256 repaymentAmount = alchemist.repay(_depositAmount, tokenId);
        vm.stopPrank();

        // calculate expected protocol fee
        uint256 expectedProtocolFee = (repaymentAmount * (alchemist.protocolFee())) / 10000;

        uint256 balanceOfTransmuter = fakeYieldToken.balanceOf(address(transmuterLogic));
        uint256 balanceOfProtocolFeeReceiver = fakeYieldToken.balanceOf(alchemist.protocolFeeReceiver());

        console.log("balance of Transmuter: ", balanceOfTransmuter);
        console.log("balance of ProtocolFeeReceiver: ", balanceOfProtocolFeeReceiver);
        console.log("expected protocol fee: ", expectedProtocolFee);

        // confirm that the wrong amount was sent to the protocol fee receiver
        assertEq(balanceOfTransmuter, balanceOfProtocolFeeReceiver);
        assert(balanceOfProtocolFeeReceiver != expectedProtocolFee);
    }
```

## Recommendation 
Update the `AlchemistV3::repay` function to correctly transfer only the calculated protocol fee to the `protocolFeeReceiver`:

```diff
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        ...
        account.collateralBalance -= (creditToYield * protocolFee) / BPS;
        _subDebt(recipientTokenId, credit);

        // Transfer the repaid tokens to the transmuter.
        TokenUtils.safeTransferFrom(yieldToken, msg.sender, transmuter, creditToYield);

-       TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, creditToYield);
+       TokenUtils.safeTransfer(yieldToken, protocolFeeReceiver, (creditToYield * protocolFee) / BPS);

    }
```


**Severity**: Informational 

## [I-1] 
