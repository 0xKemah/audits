**Severity**: Medium

# [M-1] Missing Slippage Protection In KuruAMMVault::withdraw

## Summary 
The `KuruAMMVault::withdraw` function lacks slippage protection, exposing users to receiving an unpredictable mix of asset tokens during withdrawal from the vault.

## Brief Description
When vault users initiate a withdrawal via the `KuruAMMVault::withdraw` function, they expect to get a predictable mix of base and quote tokens based on the amounts previewed before the withdrawal transaction. However, due to lack of slippage check in the function, the withdrawal transaction can be frontrun by a large swap which can significantly alter the amounts of base or quote tokens the user will obtain from the vault. 
This means that even if the resulting token distribution deviates substantially from what the user expected, the withdrawal will still succeed.

## Impact Explanation
The lack of slippage protection can result in users receiving significantly different proportions of base and quote token than expected when initiating the withdrawal transaction. This allows withdrawal transaction to be frontrun by large swaps that can alter the vault's reserve and unfairly impacting the user.

## Likelihood Explanation 
This issue is evident every time a vault user initiates a withdrawal transaction, and as a result, making the likelihood of occurrence high.

## Proof Of Concept 
Consider the following scenario where a vault market maker initiates a withdrawal from the vault:
- A user calls `KuruAMMVault::previewWithdraw` to see the amount of base and quote tokens to be received for a given amount of shares, for example, 10 base tokens and 1000 quote tokens.
- The user calls `KuruAMMVault::withdraw` to initiate the withdrawal expecting to receive 10 base tokens and 1000 quote tokens.
- The user's withdrawal transaction is frontrun by a whale swap on the vault, which alters the mix of base and quote tokens available in the vault.
- The user's withdrawal transaction succeeds after the swap with the user receiving 5 base tokens and 1500 quote tokens instead of the expected 10 and 1000.

Here is a coded PoC:
Add this test function in `test/OrderBookTest.t.sol` and run the test via `forge test --mt testKuruAmmVaultNoSlippageProtectionForWithdrawal -vvv`

```solidity 
function testKuruAmmVaultNoSlippageProtectionForWithdrawal() public {
        // create and fund vaultMaker
        address vaultMaker = makeAddr("vaultMaker");
        uint256 baseAssetAmount = 10 * 10 ** eth.decimals();
        uint256 quoteAssetAmount = 10000 * 10 ** usdc.decimals();
        eth.mint(vaultMaker, baseAssetAmount);
        usdc.mint(vaultMaker, quoteAssetAmount);

        // vaultMaker approves and deposits tokens to the vault
        vm.startPrank(vaultMaker);
        eth.approve(address(vault), baseAssetAmount);
        usdc.approve(address(vault), quoteAssetAmount);
        uint256 shares = vault.deposit(baseAssetAmount, quoteAssetAmount, vaultMaker);
        vm.stopPrank();

        // create and fund user1
        address user = makeAddr("user");
        uint256 userBaseAmount = 5 * 10 ** eth.decimals();
        eth.mint(user, userBaseAmount);

        // user swaps base token to quote token via vault
        uint256 userSwapSize = (userBaseAmount * SIZE_PRECISION) / 10 ** eth.decimals();
        uint96 sizeToU96 = uint96(userSwapSize);
        require(sizeToU96 == userSwapSize, "u96 overflow");
        vm.startPrank(user);
        eth.approve(address(orderBook), userBaseAmount);
        orderBook.placeAndExecuteMarketSell(sizeToU96, 0, false, true);
        vm.stopPrank();

        // vaultMaker checks the amount of base and quote token to receive on withdrawal
        (uint256 baseAssetWithdrawable, uint256 quoteAssetWithdrawable) = vault.previewWithdraw(shares);

        // A whale frontruns the vaultMaker's withdrawal with a market sell transaction
        address whale = makeAddr("whale");
        uint256 whaleBaseAmount = 50 * 10 ** eth.decimals();
        eth.mint(whale, whaleBaseAmount);
        uint256 whaleSwapSize = (whaleBaseAmount * SIZE_PRECISION) / 10 ** eth.decimals();
        uint96 whaleswapSizeToU96 = uint96(whaleSwapSize);
        require(whaleswapSizeToU96 == whaleSwapSize, "U96 overflow");

        vm.startPrank(whale);
        eth.approve(address(orderBook), whaleBaseAmount);
        orderBook.placeAndExecuteMarketSell(whaleswapSizeToU96, 0, false, true);
        vm.stopPrank();

        vm.prank(vaultMaker);
        (uint256 baseAssetWithdrawn, uint256 quoteAssetWithdrawn) = vault.withdraw(shares, vaultMaker, vaultMaker);

        // check that the asset withdrawn does not match the expected amounts
        assert(baseAssetWithdrawn > baseAssetWithdrawable);
        assert(quoteAssetWithdrawn < quoteAssetWithdrawable);

        console.log("baseAssetWithdrawable :", baseAssetWithdrawable);
        console.log("quoteAssetWithdrawable :", quoteAssetWithdrawable);

        console.log("baseAssetWithdrawn :", baseAssetWithdrawn);
        console.log("quoteAssetWithdrawn :", quoteAssetWithdrawn);
    }

```  
![Screenshot 2025-09-01 at 10.14.15.png](https://imagedelivery.net/wtv4_V7VzVsxpAFaxzmpbw/27409df1-939b-4cd1-f316-5a81a3406200/public)  

## Recommendation 
To mitigate this issue, add a minimum expected amount of base and quote tokens as parameters in the `withdraw` function. Implement a check that ensures the actual amount of base and quote tokens withdrawn, is within the threshold of the expected amount of base and quote tokens specified by the user.




**Severity**: Informational

# [I-1] `KuruAMMVault::previewDeposit` Reverts When `totalSupply` Is Zero

## Summary
When the total supply of vault shares is zero, the `KuruAMMVault::previewDeposit` function reverts with an unexpected error. 

## Brief Description 
The `KuruAMMVault::previewDeposit` function is expected to return the amount of shares a user will receive when a given amount of base and quote tokens is deposited in the vault. However, when the total supply of shares is zero, as a result of no base and quote tokens deposited, the function reverts with an `arithmetic underflow or overflow error` due to division by zero as shown in the snippet below:

```solidity 
function _convertToShares(uint256 baseAmount, uint256 quoteAmount, uint256 _currentAskPrice)
        internal
        view
        returns (uint256)
    {
        (uint256 _reserve1, uint256 _reserve2) = totalAssets();
        (_reserve1, _reserve2) = _returnVirtuallyRebalancedTotalAssets(_reserve1, _reserve2, _currentAskPrice);

        return FixedPointMathLib.min(
@>          FixedPointMathLib.mulDiv(baseAmount, totalSupply(), _reserve1),
@>          FixedPointMathLib.mulDiv(quoteAmount, totalSupply(), _reserve2)
        );
    }
```
This happens because in the `KuruAMMVault::_convertToShares` function, the amount of `_reserve1` and `_reserve2` returned from `totalAssets` are zero when no base or quote tokens have been deposited. Therefore an attempt to calculate the number of shares via the `FixedPointMathLib.mulDiv` function will result in division by zero (_reserve1 and _reserve2) causing a revert.

## Impact Explanation 
Users cannot preview the number of shares they will receive when there are zero deposits (i.e, the reserve balance of base and quote tokens are zero). This results in reverts and deviates from the expected behavior of the `previewDeposit` function. While the overall impact is low, the user experience is negatively affected.

## Likelihood Explanation 
The likelihood is medium to high as it occurs everytime a vault's reserve is empty.

## Proof of Concept 
Add this test function to `test/OrderBookTest.t.sol` and run with `forge test --mt testPreviewDepositReverts`:

```solidity 
function testPreviewDepositReverts() public {
        // set user deposit amount
        uint256 baseAmount = 2 * 10 ** eth.decimals();
        uint256 quoteAmount = 1000 * 10 ** usdc.decimals();

        // call previewDeposit
        vm.expectRevert();
        uint256 shares = vault.previewDeposit(baseAmount, quoteAmount);
        assertEq(vault.totalSupply(), 0);
    }
```

## Recommendation 
This issue can be mitigated by adding a condition that :
- Reeturns zero, or 
- Calculates the shares to be received for a given amount of base and quote deposit using the formula:
 `_shares = FixedPointMathLib.sqrt(baseDeposit * quoteDeposit)` 
when the totalSupply or the reserve balances is zero.


# [I-2] Denial Of Service In `KuruAMMVault::batchUpdate` During `SOFT_PAUSED` State.

## Summary 
The `marketNotHardPaused` modifier allows the `KuruAMMVault::batchUpdate` function to be executed when market state is not `HARD_PAUSED` (i.e, market state can be `SOFT_PAUSED` or `ACTIVE`). However, it reverts when the market state is `SOFT_PAUSED`, causing a denial of service for users.

## Brief Description 
The `KuruAMMVault::batchUpdate` function internally calls the `addBuyOrder` and `addSellOrder` functions, both of which are restricted by the `marketActive` modifier. This means they can only execute when the market state is `ACTIVE`. 

However, the `batchUpdate` function itself is only restricted by the `marketNotHardPaused` modifier, which allows it to be executed during the `SOFT_PAUSED` state. This inconsistency leads to reverts when users attempt to perform batch updates in `SOFT_PAUSED` state, effectively preventing them from using the function.

```solidity
    function batchUpdate(
        uint32[] calldata buyPrices,
        uint96[] calldata buySizes,
        uint32[] calldata sellPrices,
        uint96[] calldata sellSizes,
        uint40[] calldata orderIdsToCancel,
        bool postOnly
    ) external marketNotHardPaused {
        ...
        for (uint256 i = 0; i < buyPrices.length; i++) {
            addBuyOrder(buyPrices[i], buySizes[i], postOnly);
        }

        for (uint256 i = 0; i < sellPrices.length; i++) {
            addSellOrder(sellPrices[i], sellSizes[i], postOnly);
        }
        ...
    }

    function addBuyOrder(uint32 _price, uint96 size, bool _postOnly) public marketActive nonReentrant {
        ...
    }

    function addSellOrder(uint32 _price, uint96 _size, bool _postOnly) public marketActive nonReentrant {
        ...
    }
```

## Impact Explanation 
This issue results in a denial of service for users who want to execute a batch update via `KuruAMMVault::batchUpdate` when the market is `SOFT_PAUSED`. Any attempt to add buy or sell orders in this state will revert, limiting functionality for users.

## Likelihood Explanation 
The likelihood of occurrence of this issue is medium to low as it requires certain preconditions :
- Market state is `SOFT_PAUSED`
- User calls `batchUpdate` with buy and sell orders to be added to the order book.

## Proof of Concept 
Add this test function to `test/OrderBookTest.t.sol` and run with `forge test --mt testBatchUpdateOrderReverts`:

```solidity 
function testBatchUpdateOrderReverts() public {
        // prepare orders
        uint8 arrayLength = 2;
        uint32[] memory buyPrices = new uint32[](arrayLength);
        buyPrices[0] = 1010 * PRICE_PRECISION;
        buyPrices[1] = 1000 * PRICE_PRECISION;

        uint32[] memory sellPrices = new uint32[](arrayLength);
        sellPrices[0] = 1100 * PRICE_PRECISION;
        sellPrices[1] = 1200 * PRICE_PRECISION;

        uint96[] memory buySizes = new uint96[](arrayLength);
        buySizes[0] = 10 * SIZE_PRECISION;
        buySizes[1] = 9 * SIZE_PRECISION;

        uint96[] memory sellSizes = new uint96[](arrayLength);
        sellSizes[0] = 2 * SIZE_PRECISION;
        sellSizes[1] = 1 * SIZE_PRECISION;

        uint40[] memory orderIdsToCancel = new uint40[](arrayLength);
        orderIdsToCancel[0] = 1;
        orderIdsToCancel[1] = 2;

        // add orders to cancel
        uint32 buyPrice = 1500 * PRICE_PRECISION;
        uint32 sellPrice = 2000 * PRICE_PRECISION;
        uint96 buySize = 10 * SIZE_PRECISION;
        uint96 sellSize = 1 * SIZE_PRECISION;
        address maker = makeAddr("maker");
        _addBuyOrder(maker, buyPrice, buySize, 0, false);
        _addSellOrder(maker, sellPrice, sellSize, false);

        // toggle market state to SOFT_PAUSED
        address[] memory markets = new address[](1);
        markets[0] = address(orderBook);
        IOrderBook.MarketState state = IOrderBook.MarketState.SOFT_PAUSED;
        router.toggleMarkets(markets, state);

        // attempt batch update
        vm.startPrank(maker);
        vm.expectRevert(OrderBookErrors.MarketStateError.selector);
        orderBook.batchUpdate(buyPrices, buySizes, sellPrices, sellSizes, orderIdsToCancel, true);
        vm.stopPrank();
    }
```

## Recommendation 
Update the modifier on `batchUpdate` from `marketNotHardPaused` to `marketActive` to ensure consistency with `addBuyOrder` and `addSellOrder`. This guarantees that batch updates are only possible when the market state is `ACTIVE`.

Users can still cancel orders via `batchCancelOrders` while the market is `SOFT_PAUSED`.


