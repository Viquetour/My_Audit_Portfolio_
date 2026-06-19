## Mighty-Contracts Audit Findings.

## [mighty-contracts](https://cantina.xyz/code/616d8bb4-16ce-4ca9-9ce9-5b99d6e146ef/overview)


# Missing Stale Price Threshold and Expo Handling in `PrimaryPriceOracle.sol` Contract

- **Severity:** High
- **Likelihood:** High
- **Impact:** High
- **Created by:** ViquetourOnTop

## Description

## Description

The `PrimaryPriceOracle` contract, designed to fetch price data from the Pyth network, has a critical issue related to the handling of price staleness and expo. The contract calls the `getPriceUnsafe` function from the Pyth SDK to retrieve price data and implements a basic staleness check using the `maxPriceAge` parameter. However, it lacks a dedicated stale price hold mechanism and doesn't account for the price expo provided by the Pyth price feed. These omissions can lead to unreliable price data.

The `maxPriceAge` could serve a similar purpose, like setting the stale price threshold, but it isn't named or documented as a stale price threshold, which could lead to confusion or oversight. Additionally, the lack of a requirement that `maxPriceAge` is greater than 0 (maxPriceAge > 0 ) in the `setMaxPriceAge` function risks setting it to zero, which will disable the staleness check entirely, allowing outdated prices to be used.

The `PythStruct.price` includes an expo field, which represents the exponents to scale the price(e.g if expo= -8, a price of 100000000 represents 1.0 in an asset-based unit). The contract adjusts the price to `uint256(uint64(PythStruct.price))`, but does not adjust to expo. This means the returned price is in its raw form without accounting for decimal calling provided by expo. For example, a price of `100000000` with expo `= -8` will be treated as `1.0`, but the contract returns `100000000`, which could be misleading. This is a significant issue where accurate price calling is critical. 
 

## Recommendation
To resolve this issue, implement a stale price threshold for clarity and add a validation to make sure it is always greater than 0.

Modify the `getPythPrice` function to adjust the price based on the expo field. This ensures the returned price is normalised to a consistent decimal format.


__________________________________________________________________________________________________________________________________________________________________
__________________________________________________________________________________________________________________________________________________________________


# No Partial Liquidation in Contract as Documented

- **Severity:** High
- **Likelihood:** High
- **Impact:** High
- **Created by:** ViquetourOnTop

## Description

## Description
The Liquidate function does not implement a partial liquidation as documented. The documentation specifies a staged liquidation process for low liquidity pools and large positions as well, where:

     (I) Initial Liquidation covers 50% of the position
     (ii) If the position remains above the liquidation threshold, an additional 25% is liquidated.
     (iii) This process repeats until the position stabilizes or is fully closed.

However, the current implementation of the `liquidatePosition` performs a full liquidation in a single step by calling `IShadowRangePositionImpl(positionInfo.positionAddress).liquidatePosition(msg.sender)`. which doesn't account for the staged percentages described in the documentation.

This discrepancy could lead to excessive liquidation in low-liquidity pools, potentially causing losses for position owners and destabilizing the protocol.

## Recommendation
To align the contract with the documented partial liquidation mechanism, the liquidate position function should be modified to implement the staged liquidation process.

__________________________________________________________________________________________________________________________________________________________________
__________________________________________________________________________________________________________________________________________________________________


# No LimitOrder Fee in Smart Contract


- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** Medium
- **Created by:** ViquetourOnTop


## Description

## Description
The protocol fails to collect the documented 0.05% Limit order execution fee, allowing users to call the `fullfillLimitOrder` function and close via `_closePosition` without the deduction of the specified fee. This makes users avoid paying fees entirely, violating the intended documented rules of the protocol. This flaw makes the protocol lose guaranteed revenue from its primary fee mechanism. 

## Recommendation
Implement fee collection in `fullfillLimitOrder` function.
__________________________________________________________________________________________________________________________________________________________________
__________________________________________________________________________________________________________________________________________________________________


# Discrepancies in Liquidation Fees and Liquidation Debt Ratio  Between Docs and Contract


- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** Low
- **Created by:** ViquetourOnTop

## Description

## Description
The smart contract specifies `LiquidationDebtRatio = 8600` which is 86% and `LiquidationFee = 500%` (5%) these values conflict with the provided documentation, which states a Liquidation fee of 7.5% and Liquidation threshold of 80% for stablecoin pairs, 70% for general Asset pairs and governance defined thresholds for high risk assets. The discrepancy in the liquidation fee will result in Liquidity Providers(LPs) getting a lower fee than expected during the liquidation process, potentially affecting their incentives and trust in the protocol. Also, the `LiquidationDebtRatio` of 86% doesn't correspond with the documented thresholds, which could lead to unexpected liquidation for users, as liquidation may occur at a higher debt ratio than advertised. This misalignment between contract and documentation may cause financial discrepancies and reputational damage.

## Recommendation
Kindly update `LiquidationFee` and `LiquidationDebtRatio` in the smart contract to align with the documentation, or the documentation to align with the smart contract.

__________________________________________________________________________________________________________________________________________________________________
__________________________________________________________________________________________________________________________________________________________________



# Insufficient Token Balances Can Cause Liquidation to Revert After Fees Are Collected. 


- **Severity:** High
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** High
- **Created by:** ViquetourOnTop

## Description

## Description
The Liquidate position function may revert after collecting fees due to insufficient tokens to cover the full debt repayment.
Specifically, when attempting to repay `currentDebt0` or `currentDebt1`, the function checks if the current debt is greater than the reducedToken amount `currentDebt0 > token0Reduced`. It then checks for excess amounts in token1 to swap to token0 to cover the shortfall using `swapTokenEaxctInput` function. The same process applies if current debt1 is greater than token1 reduced `currentDebt1 > token1Reduced`, it swaps token0 to token1. However, the function fails to verify if the `token1` or `token0` has sufficient amount to perform the swap to cover for the shortfall. If token1 is insufficient to cover the debt of token0 (or vice versa), the swap will fail, causing the liquidation process to revert.

The current issue is that the function always attempts to pay all debt, without first verifying that the available tokens after fee collection, plus any potential swap, can cover the full debt amount.

## Recommendation

1. Consider implementing partial liquidation when full debt repayment isn't possible and emit an event indicating partial repayment occurred.

2. Implement pre-liquidation checks to ensure available tokens, after accounting for the fee, plus swap tokens, can cover the debt.  


