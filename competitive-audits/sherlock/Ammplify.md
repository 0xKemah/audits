**Severity**: Medium

# [M-1] NFTManager::burnAsset Exposes Users To JIT Penalty

## Summary
The `NFTManager::burnAsset` function collects fees before removing the maker position, which makes the position eligible for JIT penalty deductions, causing financial loss for the user.

## Root Cause
When users call the `NFTManager::burnAsset` function, it collects fees by calling `MAKER_FACET.collectFees` before removing the maker position via `MAKER_FACET.removeMaker`.
```solidity
    function burnAsset(uint256 tokenId, uint128 minSqrtPriceX96, uint128 maxSqrtPriceX96, bytes calldata rftData)  
        external  
        returns (address token0, address token1, uint256 removedX, uint256 removedY, uint256 fees0, uint256 fees1)  
    {  
        ...  
        // First collect fees from the position  
        (fees0, fees1) = MAKER_FACET.collectFees(msg.sender, assetId, minSqrtPriceX96, maxSqrtPriceX96, rftData);  
  
        // Then remove the maker position  
        (token0, token1, removedX, removedY) =  
            MAKER_FACET.removeMaker(msg.sender, assetId, minSqrtPriceX96, maxSqrtPriceX96, rftData);  
        ...  
    }
```
However, `Maker.collectFees` updates the asset's timestamp regardless of the liquidity type (compounding or non-compounding liquidity) setting it to the current `block.timestamp`. An attempt to remove maker via `Maker.removeMaker` will trigger `FeeLib.applyJITPenalties`. Since the duration will be less than the `JITLifetime`, the JIT penalty is applied, causing significant financial loss for users.

## Internal Pre-conditions
N/A

## External Pre-conditions
N/A

## Attack Path
N/A

## Impact
Users who create their maker positions via the `NFTManager` contract will always suffer JIT penalty losses whenever they attempt to close their position by calling the `NFTManager::burnAsset` function. This occurs even if their maker positions have existed long enough to exceed the `JITLifetime`.

## PoC
Add this test function in `NFTManager.t.sol` and run it with `forge test --mt testBurnAssetAppliesJitPenalty -vvv`
```solidity
    function testBurnAssetAppliesJitPenalty() public {  
        // Set JIT lieftime and Penalty  
        uint64 jitLifetime = 1 days;  
        uint64 jitPenalty = 6148914691236516864; // 33%  
        adminFacet.setJITPenalties(jitLifetime, jitPenalty);  
  
        // Fund the test account for RFT operations  
        _fundAccount(address(this));  
  
        // Approve NFTManager to spend tokens  
        token0.approve(address(nftManager), type(uint256).max);  
        token1.approve(address(nftManager), type(uint256).max);  
  
        bytes memory rftData = "";  
  
        // First mint a new maker position via NFTManager  
        (uint256 tokenId, uint256 assetId) = nftManager.mintNewMaker(  
            address(this),  
            address(pools[0]),  
            LOW_TICK,  
            HIGH_TICK,  
            LIQUIDITY,  
            false,  
            uint128(MIN_SQRT_RATIO),  
            uint128(MAX_SQRT_RATIO),  
            rftData  
        );  
  
        // Verify the NFT was minted  
        assertEq(nftManager.totalSupply(), 1);  
        assertEq(nftManager.ownerOf(tokenId), address(this));  
  
        // Update block timestamp to avoid JIT penalty  
        vm.warp(block.timestamp + jitLifetime);  
  
        // Remove asset via NFTManager  
        (,, uint256 removedXNftManager, uint256 removedYNftManager,,) = nftManager.burnAsset(  
            tokenId,  
            uint128(MIN_SQRT_RATIO), // minSqrtPriceX96  
            uint128(MAX_SQRT_RATIO), // maxSqrtPriceX96  
            rftData // rftData  
        );  
  
        // Create maker position directly via Maker facet  
        _fundAccount(address(this));  
        uint256 _assetId = makerFacet.newMaker(  
            address(this),  
            address(pools[0]),  
            LOW_TICK,  
            HIGH_TICK,  
            LIQUIDITY,  
            false,  
            uint128(MIN_SQRT_RATIO),  
            uint128(MAX_SQRT_RATIO),  
            rftData  
        );  
  
        // Verify asset was created  
        assertEq(_assetId, 2);  
  
        // Update block timestamp to avoid JIT penalty  
        vm.warp(block.timestamp + jitLifetime);  
        // Remove asset directly via Maker facet  
        (,, uint256 actualRemovedX, uint256 actualRemovedY) =  
            makerFacet.removeMaker(address(this), _assetId, uint128(MIN_SQRT_RATIO), uint128(MAX_SQRT_RATIO), rftData);  
  
        console.log("removedX via NFTManager is:", removedXNftManager);  
        console.log("actualRemovedX is:", actualRemovedX);  
  
        console.log("removedY via NFTManager is:", removedYNftManager);  
        console.log("actualRemovedY is:", actualRemovedY);  
  
        assert(actualRemovedX > removedXNftManager);  
        assert(actualRemovedY > removedYNftManager);  
    }
```

## Mitigation
Consider the following:

  - Implement a condition in Maker::removeMaker that exempts the NFTManager contract from JIT penalties when the asset's timestamp is equal to the current block.timestamp.
  - Add a check in the NFTManager::burnAsset function that requires the asset's duration to be greater than or equal to the JITLifetime before the function can be executed.



# [M-2] Potential DoS In NFTManager Contract Due To MAX_ASSET_PER_OWNER Limit

## Summary
The `Asset::addAssetToOwner` function does not allow more than 16 assets per owner and this applies to the `NFTManager` contract. Therefore causing reverts leading to denial of service for users when the manager contract reaches this maximum asset limit.

## Root Cause
When a user calls the `NFTManager::mintNewMaker` function, the `NFTManager` attempts to create a new maker position with its address as the owner. However, the `Asset::addAssetToOwner` function enforces a limit of 16 assets per owner without providing an exception for the `NFTManager` contract. This restriction prevents new maker positions from being minted once the contract reaches the maximum limit.

## Internal Pre-conditions
`NFTManager` contract owns up to the `MAX_ASSET_PER_OWNER` number of assets

## External Pre-conditions
N/A

## Attack Path
  - Users interact with the NFTManager contract via NFTManager::mintNewMaker to mint maker positions.
  - Once the contract owns 16 assets, any subsequent mint attempts revert due to the MAX_ASSET_PER_OWNER restriction.
  - This blocks all further users from creating maker positions via the NFTManager.

## Impact
Since the NFTManager contract is not exempt from the MAX_ASSET_PER_OWNER restriction, users are unable to mint new maker positions once the limit is reached, resulting in a denial of service.

## PoC
Add this test function in `NFTManager.t.sol` and run it with `forge test --mt testDosForNftManager -vvv`
```solidity
    function testDosForNftManager() public {  
        // Fund the test account for RFT operations  
        _fundAccount(address(this));  
  
        // Approve NFTManager to spend tokens  
        token0.approve(address(nftManager), type(uint256).max);  
        token1.approve(address(nftManager), type(uint256).max);  
  
        bytes memory rftData = "";  
  
        uint256[] memory tokenIds = new uint256[](16);  
        uint256[] memory assetIds = new uint256[](16);  
  
        // Mint a new maker position via NFT manager  
        for (uint256 i = 0; i < 16; i++) {  
            (uint256 tokenId, uint256 assetId) = nftManager.mintNewMaker(  
                address(this), // recipient  
                address(pools[0]), // poolAddr  
                LOW_TICK, // lowTick  
                HIGH_TICK, // highTick  
                LIQUIDITY / 17, // liq  
                false, // isCompounding  
                uint128(MIN_SQRT_RATIO), // minSqrtPriceX96  
                uint128(MAX_SQRT_RATIO), // maxSqrtPriceX96  
                rftData // rftData  
            );  
            tokenIds[i] = tokenId;  
            assetIds[i] = assetId;  
        }  
  
        // Verify that attempting to mint beyond the limit reverts  
        vm.expectRevert(  
            abi.encodeWithSelector(AssetLib.ExcessiveAssetsPerOwner.selector, AssetLib.MAX_ASSETS_PER_OWNER)  
        );  
        nftManager.mintNewMaker(  
            address(this), // recipient  
            address(pools[0]), // poolAddr  
            LOW_TICK, // lowTick  
            HIGH_TICK, // highTick  
            LIQUIDITY / 17, // liq  
            false, // isCompounding  
            uint128(MIN_SQRT_RATIO), // minSqrtPriceX96  
            uint128(MAX_SQRT_RATIO), // maxSqrtPriceX96  
            rftData // rftData  
        );  
  
        assertEq(tokenIds.length, 16);  
        assertEq(assetIds.length, 16);  
    }
```

## Mitigation
Introduce an exception in `Asset::addAssetToOwner` that exempts the `NFTManager` contract from the `MAX_ASSET_PER_OWNER` limit.

Additionally, to prevent unbounded loop errors when iterating over the assets owned by the `NFTManager`, use the `tokenId` of each asset as its index in the `NFTManager’s` asset array.`
```diff
    function addAssetToOwner(AssetStore storage store, uint256 assetId, address owner) private {  
+       if (owner != address(NFTManger)) {  
            require(  
                store.ownerAssets[owner].length < MAX_ASSETS_PER_OWNER,  
                ExcessiveAssetsPerOwner(store.ownerAssets[owner].length)  
            );  
+       }  
        store.ownerAssets[owner].push(assetId);  
    }
```

# [M-3] `Maker::collectFees` Updates Asset Timestamp For Non-Compounding Maker Liquidity, Exposing Maker To JIT Penalty

## Summary
The `Maker::collectFees` function updates the asset timestamp to the current block.timestamp, exposing the maker to a JIT penalty even though liquidity was not added or removed on the maker's non-compounding liquidity position.

## Root Cause
In `Maker::collectFees`, when a maker collects fees on a non-compounding liquidity position, the asset's timestamp is updated to the current block.timestamp. This behaviour makes the position vulnerable to JIT penalty if the maker attempts to remove liquidity after collecting fees, but enough time has not elapsed to exceed the `jitLifetime`.

This is an issue because the jit penalty should be applicable to makers who add and remove liquidity within the `jitLifetime` and not when fees are claimed. For a non-compounding maker liquidity type, claiming fees does not modify the maker's liquidity amount, and therefore should not update the asset's timestamp.

## Internal Pre-conditions
  - Maker's liquidity is non-compounding.
  - Maker collects fees before removing liquidity within the jitLifetime.

## External Pre-conditions
N/A

## Attack Path
N/A

## Impact
Updating the maker's asset timestamp when fees are collected, unfairly exposes the maker to paying a JIT penalty on non-compounding liquidity that was added long enough ago that it should have exceeded the `jitLifetime`.

## PoC
  - Add this view function to `facets/View.sol`:
```solidity
    function getTime(uint256 assetId) external view returns (uint128 timestamp) {  
        Asset storage asset = AssetLib.getAsset(assetId);  
        return asset.timestamp;  
    }
```
  - Set `Diamond.sol` ViewFacet selectors array length to 7

  - Updates `Diamond.sol` ViewFacet selectors array with :
```solidity
    selectors[6] = ViewFacet.getTime.selector;
```
  - Add this test function to `Maker.t.sol` and run the test in your terminal using `forge test --mt testCollectFeesAllowsJitPenalty -vvv`
```solidity
    function testCollectFeesAllowsJitPenalty() public {  
        // create a new maker position  
        bytes memory rftData = "";  
        uint256 assetId = makerFacet.newMaker(  
            recipient, poolAddr, lowTick, highTick, liquidity, false, minSqrtPriceX96, maxSqrtPriceX96, rftData  
        );  
  
        // get the asset timestamp and jit duration  
        uint256 assetTimestampBefore = viewFacet.getTime(assetId);  
  
        // Set JITLifetime and penalty  
        uint64 jitLifeTime = 2 hours;  
        uint64 jitPenalty = 1e12;  
        adminFacet.setJITPenalties(jitLifeTime, jitPenalty);  
        (,,, uint64 _jitLifetime,) = adminFacet.getDefaultFeeConfig();  
  
        // inflate the blocktimestamp  
        vm.warp(block.timestamp + _jitLifetime);  
        assert(block.timestamp - assetTimestampBefore >= _jitLifetime);  
  
        // execute swaps  
        swapTo(0, TickMath.getSqrtRatioAtTick(600));  
        swapTo(0, TickMath.getSqrtRatioAtTick(-600));  
  
        // prank the user to collect fees  
        vm.prank(recipient);  
        (uint256 fees0, uint256 fees1) =  
            makerFacet.collectFees(recipient, assetId, minSqrtPriceX96, maxSqrtPriceX96, rftData);  
  
        // Verify fees were collected  
        assertGe(fees0, 0);  
        assertGe(fees1, 0);  
  
        // verify that the user's position is prone to JIT penalty after collecting fees  
        uint256 assetTimestampAfter = viewFacet.getTime(assetId);  
        assert(block.timestamp - assetTimestampAfter < _jitLifetime);  
    }  
```

## Mitigation
To mitigate this issue, add a condition in `Maker::collectFees` that checks if the liquidity type is not non-compounding before updating the asset's timestamp. This will ensure that the asset's timestamp is not updated for non-compounding maker liquidity type, and as a result eliminate the potential risk of JIT penalty for the maker.
```diff
    function collectFees(  
        address recipient,  
        uint256 assetId,  
        uint160 minSqrtPriceX96,  
        uint160 maxSqrtPriceX96,  
        bytes calldata rftData  
    ) external nonReentrant returns (uint256 fees0, uint256 fees1) {  
        ...  
+       if (asset.liqType != LiqType.MAKER_NC) {  
            AssetLib.updateTimestamp(asset);  
+       }   
        ...         
    }
```
