**Severity**: High

## [H-1] Missing Validation for Confidence Interval Exposes Protocol to Imprecise Price

## Summary 
The `PrimaryPriceOracle::getPythPrice` function does not validate the `confidence interval` of price data, leaving the protocol vulnerable to use unreliable prices.

## Description 
Confidence intervals are critical for ensuring the accuracy and reliability of price estimates derived from price oracles. The absence of validation in `PrimaryPriceOracle::getPythPrice` introduces the risk of using price data that is less precise across the protocol. Without proper validation, the protocol may rely on erroneus price estimates leading to incorrect valuation of assets and liquidations.

The confidence interval validation, ensures that the reported prices fall within an acceptable range reducing the risk of incorrect pricing or arbitrage opportunities within the protocol.

## Impact Explanation 
The lack of validation for the confidence interval exposes the protocol to the risk of using imprecise price for valuing assets and collateral. This can cause inaccurate collateral valuation and unintended liquidation, resulting in financial losses for both protocol users and the protocol itself.

## Likelihood Explanation 
The chance of this protocol relying on erroneous price feeds is high, as price oracles are vulnerable to price fluctuations and data inconsistencies. Without a proper validation on the accuracy of the reported price, the protocol will frequently encounter situations where imprecise price data impacts critical operations such as collateral valuation and liquidation of opened position.

## Prrof of Concept
Consider the following scenario in which a user opens a position on the protocol:

- The user deposits collateral and opens a position.
- The protocol gets the current price of the assets from the PrimaryPriceOracle contract to determine the value of the user's collateral
- Due to market volatility or oracle manipulation attack, the oracle returns a price with a wide confidence interval (e.g., ±20%)
- The protocol uses the reported price without validating the confidence interval resulting in a possible overvaluation or undervaluation of the collateral.
- Protocol calculates user debt ratio based on the reported price.
- Depending on the direction of the inaccuracy:

    - Overvaluation: The user may close the position and extract more value than justified, even though the actual price would make the debt ratio exceed the liquidation threshold.

    - Undervaluation: The user's position may be incorrectly liquidated, even though the actual price would keep the debt ratio safely below the liquidation threshold. This scenario demonstrates how the lack of confidence interval validation can lead to financial loss for either the protocol or its users, depending on the direction of the price deviation.

## Recommendation
A strict validation check on the confidence interval should be implemented on the reported price before it is used in any critical calculation such as collateral valuation, debt ratio assessments or liquidation checks.

An acceptable threshold should be defined for the confidence interval (e.g., ±1-5%), and the reported price should be rejected if it exceeds the defined threshold. In such cases, an alternative price source should be used or sensitive operations should be paused.

Consider making the following changes the `PrimaryPriceOracle::getPythPrice` to validate confidence interval:

- Create a state variable that stores the maximum allowed price interval in `PrimaryPriceOracle` contract.
- Add a condition to validate the confidence interval in `getPythPrice` function.

```diff
+   uint256 maxConfidenceInterval = 5e16 // 5% in 1e18 format 

    function getPythPrice(address token) public view returns (uint256) {
        bytes32 priceId = pythPriceIds[token];
        if (priceId == bytes32(0)) {
            revert("PriceId not set");
        }

        PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);

        require(priceStruct.publishTime + maxPriceAge > block.timestamp, "Price is too old");
+       require(priceStruct.conf <= maxConfidenceInterval, "Confidence interval too wide");

        uint256 price = uint256(uint64(priceStruct.price));

        return price;
    }
```


**Severity**: Low

## [L-1] Delayed Price Updates Expose Protocol to Stale Price Risk

## Summary
Excessive delay in the frequency at which the price is updated, allows for the use of stale or outdated price during critical operations.

## Description
In `PrimaryPriceOracle::initialize` function, the `maxPriceAge` is set to `1 hour` which implies that price data published within the last hour is permitted for use across the protocol. This allows price data up to 60 minutes old to be used in critical protocol operations such as liquidations, debt repayments and collateral valuations etc.

This issue is enforced through the `getPythPrice` function where the protocol explicitly checks that `publishTime` plus `maxPriceAge` is greater than `block.timestamp`. This condition permits the use of data that may no longer reflect real time market conditions.

This can be seen in `line #32` of `PrimaryPriceOracle::initialize` and `line #68` of `PrimaryPriceOracle::getPythPrice`.

```solidity
    function initialize(
        address _pyth,
        address _oracleManager,
        address[] memory initialTokens,
        bytes32[] memory priceIds
    ) external initializer {
        pyth = IPyth(_pyth);
        oracleManager = _oracleManager;
        require(initialTokens.length == priceIds.length, "Initial tokens and price ids length mismatch");
        for (uint256 i = 0; i < initialTokens.length; i++) {
            pythPriceIds[initialTokens[i]] = priceIds[i];
        }
@>      maxPriceAge = 1 hours;
    }

    function getPythPrice(address token) public view returns (uint256) {
        bytes32 priceId = pythPriceIds[token];
        if (priceId == bytes32(0)) {
            revert("PriceId not set");
        }
        PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);

@>      require(priceStruct.publishTime + maxPriceAge > block.timestamp, "Price is too old");
        uint256 price = uint256(uint64(priceStruct.price));

        return price;
    }
```
Although the intention might be to provide a buffer for oracle update delays, however the 1 hour threshold is excessive and can be exploited in high-volatility market conditions where prices of assets can change significantly within a short period of time. In such events, the protocol will allow users to withdraw overvalued/undervalued collateral, repay debt at low/high cost, avoid liquidation or get unfairly get liquidated.

## Impact Explanation
Using a `maxPriceAge` of `1 hour` allows the protocol to use stale price data, which will potentially expose the protocol to the following impacts:

- Undervaluation or overvaluation of collateral and debt.
- Delayed liquidation leading to insolvency
- Early liquidation of leveraged position below liquidation ratio
- Oracle price manipulation attacks where malicious actors can influence off-chain markets and time interactions during stale window

## Likelihood Explanation 
The likelihood of this issue being exploited is high as the attacker just needs proper timing and observation. Since the protocol permits prices up to 1 hour old, an attacker just needs to wait for a favorable market condition where onchain prices deviate significantly from real time market prices. This is possible in high volatility events where market crashes or surges, oracle update delays due to off-chain failures or congestion and illiquid tokens swap where prices can swing rapidly.

## Proof of Concept 
- A user deposits S tokens and $USDCe to open a leveraged position in S/USDCe pool.
- The price of S token crashes, reducing the collateral value and increasing the debt ratio of the position.
- Due to the sudden crash, the protocol is yet to update price since last updated price is below `maxPriceAge`.
- The user's position is not yet liquidatable since the position is valued using the last updated price.
- The price of S token continues to crash until position becomes insolvent leaving the protocol with bad debt.
- This form of bad debt accumulates over time causing liquidity shortage for other protocol users.

## Recommendation 
This issue can be mitigated by the following recommendations:

- Reducing the `maxPriceAge` parameter to a more conservative value, such as 5 to 15 minutes to ensure the most recent price data is used for protocol operations.
- Implementing a configurble `maxPriceAge` based on the volatility of the asset allowing a shorter time for more volatile tokens and a longer time for stable assets.


## [L-2] Borrowing can Exceed Maximum Leverage Limit Defined in Protocol Documentation

## Summary
Comparing `getDebtRatioFromAmount()` with `liquidationDebtRatio` in `ShadowRangeVault::openPosition` allows users to borrow beyond the protocol's maximum leverage of 5x, as stated in the protocol documentation.

## Description
In the `ShadowRangeVault::openPosition`, the protocol computes a user's debt using `getDebtRatioFromAmounts()` function, which takes in the principal and borrowed amounts as inputs. This calculated debt ratio is then compared against the `liquidationDebtRatio` threshold of `8600` (86% using protocol's 1e4 precision), and the transaction is allowed to proceed only if the resulting value is below this threshold. However, this comparison does not enforce the protocol's maximum leverage of 5x as specified in the protocol documentation. With `liquidationDebtRatio` set to 86 percent, this will allow leveraged positions well above the protocol's maximum leverage to pass the check. As a result, the current check enforces only a liquidation threshold, not a maximum leverage constraint. Users are able to open positions with up to 7x leverage, without violating the current validation logic. This behavior contradicts the protocol's stated parameters and could expose the system to greater risk than anticipated, particularly under volatile market conditions or in scenarios where liquidation process fails or lags.

## Impact Explanation 
Allowing users to exceed the maximum leverage threshold exposes the protocol to the following risks:

- Increased liquidation risk as users who borrow beyond the maximum leverage limit are vulnerable to liquidation in the event of market volatility, which could potentially lead to greater losses for the protocol.
- Protocol liquidity can be drained faster than expected by allowing users to open positions with more leverage than intended.
- User trust and protocol integrity is undermined due to the mismatch between the protocol documentation and its actual implementation.

## Likelihood Explanation
Since the protocol only compares the calculated debt ratio against the `liquidationDebtRatio`, and does not incorporate a hard check for the 5x leverage limit, it would be relatively simple for a user to bypass the intended leverage constraints. As such, any user familiar with the protocol's operations could leverage this discrepancy without significant technical complexity, making it an easily exploitable issue. Given the ease at which this issue can be exploited, the likelihood of its occurrence is high; the protocol's maximum leverage can be bypassed every time a user opens a new position.

## Proof of Concept
- Create a POC_Test.t.sol file in foundry with the appropriate test setup
- Add the following code snippet to the test file

```solidity
    function test_userCanExceedMaxLeverage() public {
        uint256 amount0Principal = 1000e18;
        uint256 amount1Principal = 500e6;

        // User borrows up to 7x of principal
        uint256 amount0Borrow = 6000e18;
        uint256 amount1Borrow = 3000e6;

        // Compute debt ratio from amount
        uint256 debtRatioFromAmount =
            vault.getDebtRatioFromAmounts(amount0Principal, amount1Principal, amount0Borrow, amount1Borrow);

        // Get liquidation debt ratio
        uint256 liquidationDebtRatio = vault.liquidationDebtRatio();

        assert(debtRatioFromAmount < liquidationDebtRatio);
        console2.log("Debt ratio from amount is : ", debtRatioFromAmount);
        console2.log("Liquidation debt ratio is : ", liquidationDebtRatio);
    }
```
- Run forge test --mt test_userCanExceedMaxLeverage -vvv
- The result is a log of debtRatioFromAmount and liquidationDebtRatio, where:
    - debtRatioFromAmount = 8571
    - liquidationDebtRatio = 8600

## Recommendation
To mitigate this issue, the result of `getDebtRatioFromAmounts()` should be compared to a predefined `Loan-To-Value (LTV)` ratio of 80 percent (80%), which clearly enforces the max leverage limit of 5x as specified in the protocol documentation.

Consider making the following changes to the `ShadowRangeVault` contract:

```diff
+   uint256 public LTV_Ratio = 8000;
    function openPosition(OpenPositionParams calldata params)
        external
        payable
        nonReentrant
        checkDeadline(params.deadline)
    {
        ...
-        require(getDebtRatioFromAmounts (params.amount0Principal, params.amount1Principal, params.amount0Borrow, params, amount1Borrow) < liquidationDebtRatio, "Borrow value is too high" );

+        require(getDebtRatioFromAmounts (params.amount0Principal, params.amount1Principal, params.amount0Borrow, params,   amount1Borrow) <= LTV_Ratio, "Borrow value is too high" );
        ...
    }
```  
