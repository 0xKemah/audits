**Severity**: High

# [H-1] Denial of Service (Dos) via Donation Attack By First Depositor

## Summary
In the `SuccinctStaking::stake` function, the initial depositor can artificially inflate the share price by staking the minimum allowed amount and then donating a large quantity of $PROVE tokens, leading to denial of service (DoS) for other users. 

## Finding Description
During the staking operation, the `SuccinctStaking::stake` function is vulnerable to a donation attack, which allows the first staker to deposit the minimum allowed amount and then donate a large amount of $PROVE to the $iPROVE vault, significantly increasing the vault's share price.

This manipulation results in an unfair share distribution for subsequent users. Specifically, users whose deposits are less than the artificially increased share price will receive zero shares, even if their deposit meets the protocol's minimum requirement. As a result, their transactions revert with the `ZeroReceiptAmount` error, effectively preventing them from staking, thereby causing a denial of service for genuine users of the protocol.

## Impact Explanation 
This issue creates a Denial of Service (DoS) condition for genuine protocol users, as they are unable to stake their $PROVE tokens despite meeting the minimum deposit criteria of the protocol. It also alters the protocol's fair share distribution as the attacker can artificially inflate the $iPROVE share price to ensure that future deposits below the manipulated share price yield zero shares, causing the transaction to revert.

## Likelihood Explanation 
The likelihood of exploitation is medium to high, as any user can become the first depositor and execute the attack through normal protocol interactions without special permissions. However, the high cost of donating a large amount of $PROVE may deter some attackers.

## Proof of Concept 
Consider the following scenario:
1. An attacker becomes the first user to call `SuccinctStaking::stake` function, setting the initial share price of 1:1 ratio of $PROVE to $iPROVE tokens.
2. The attacker immediately after staking (probably in the same transaction), donates a large amount of $PROVE directly to the $iPROVE vault bypassing the staking contract. Since no new shares are minted, the share price increases significantly.
3. Subsequent users attempt to stake amounts that are above the protocol's minimum but below the newly inflated share price.
4. The transaction reverts with the error `ZeroReceiptAmount` causing a denial of service for genuine protocol users. Also users who are able to stake an amount greater than the newly inflated share price, get an unfair share distribution for their staked $PROVE tokens.

Below is a coded Proof of Concept to demonstrate this issue:

```solidity
    function test_donationAttackCausesDoS() public {
        uint256 stakeAmount = MIN_STAKE_AMOUNT;

        // attacker initiates staking of minimum stake amount
        vm.startPrank(STAKER_1);
        IERC20(PROVE).approve(STAKING, stakeAmount);

        SuccinctStaking(STAKING).stake(ALICE_PROVER, stakeAmount);
        vm.stopPrank();

        // attacker donates 1 million tokens to the iPROVE vault bypassing the staking contract
        uint256 donation = 1_000_000e18;
        deal(PROVE, STAKER_1, donation);
        vm.prank(STAKER_1);
        IERC20(PROVE).transfer(I_PROVE, donation);

        // another user attempts to stake twice the minimum amount of the protocol
        vm.startPrank(STAKER_2);
        IERC20(PROVE).approve(STAKING, 2 * stakeAmount);
        // transaction reverts due to zero iPROVE shares as a result of the inflated share price
        vm.expectRevert(abi.encodeWithSelector(ISuccinctStaking.ZeroReceiptAmount.selector));
        SuccinctStaking(STAKING).stake(ALICE_PROVER, stakeAmount);

        assertEq(IERC20(STAKING).balanceOf(STAKER_1), stakeAmount);
        assertEq(IERC20(PROVE).balanceOf(I_PROVE), stakeAmount + donation);
    }
```

## Recommendation 
To prevent this issue, the following mitigations should be considered:
1. Implement a virtual share mechanism that sets the initial share price of the $iPROVE vault during deployment.
2. Decouple donations from share price calculation, or account for them in a way that doesn't affect future stakers disproportionately.
