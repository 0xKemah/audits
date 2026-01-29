**Severity**: High

# [H-1] `stNXM::removeTokenIdAtIndex` Does Not Check if Staking Pool has Active Stakes Before Removing

## Summary
The `stNXM::removeTokenIdAtIndex` function allows the contract owner to remove a token ID from the vault without verifying whether the corresponding staking pool has active stakes. This oversight enables the owner to delete a pool that still has staked tokens, potentially leading to unintended loss of assets and incorrect accounting within the vault.

## Root Cause
The `removeTokenIdAtIndex` function lacks a check to determine if the staking pool associated with the token ID at the given index has any active stakes. This missing validation allows token IDs with ongoing user stakes to be removed from the vault, which can impact vault balances and share price calculations.

## Internal Pre-conditions
N/A

## External Pre-conditions
N/A

## Attack Path
N/A

## Impact
The vault’s total assets and share price may be artificially reduced, compromising integrity and trust in the protocol. This can lead to incorrect accounting and potential financial loss across all vault participants.

## PoC
Add this test to `stNXM-Contracts/test/foundry/stNXM.t.sol` and run using `forge test --mt testOwnerCanDeletePoolWithActiveStakes -vvv`
```solidity
        function testOwnerCanDeletePoolWithActiveStakes() public {  
        // get a tokenId  
        uint256 tokenId = stNxm.tokenIds(1);  
  
        // get the pool for the tokenId obtained  
        address stakingPool = stNxm.tokenIdToPool(tokenId);  
  
        // confirm that the pool has an active stake  
        uint256 activeStake = IStakingPool(stakingPool).getActiveStake();  
        console2.log("active stake :", activeStake);  
        require(activeStake > 0);  
  
        // owner removes pool with active stake  
        vm.prank(multisig);  
        stNxm.removeTokenIdAtIndex(1);  
  
        //verify pool has been removed  
        require(stNxm.tokenIdToPool(tokenId) == address(0));  
    }
```

## Mitigation
Add a validation in `removeTokenIdAtIndex` to check that the staking pool for the token ID has no active stakes before allowing removal.
