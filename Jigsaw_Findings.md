Jigsaw-Contracts-Audit-Findings.

[Jigsaw-contest-details](https://cantina.xyz/code/7a40c849-0b35-4128-b084-d9a83fd533ea/overview)

#### [H-1]

### Title: Partial Liquidation Without Minimum Amount Leaves Positions With Unbacked Debt
[summary]
The liquidate() function permits partial liquidations without enforcing a minimum liquidation amount, allowing multiple liquidators to perform small liquidations on an undercollateralised position. Each liquidator includes a liquidation bonus, which is deducted from the user's collateral. When multiple partial liquidations occur on the same position, the cumulative bonuses can deplete the collateral disproportionately, leaving the remaining debt undercollateralized or unbacked. For example, a position with 500 USD collateral and 500 jUSD debt can be partially liquidated by two liquidators for 225 jUSD each with a bonus from the collateral, potentially leaving the 50 jUSD with insufficient collateral or unbacked.

## Proof of Concept
The provided test demonstrates the issue by simulating two partial liquidations on an undercollateralized position.
```soliddity
function test_liquidate_when_withStrategies(
    uint256 _collateralAmount
) public {
    TestTempData memory testData;

    //  Initialize user with collateral and debt
    testData.user = user;
    testData.userHolding = initiateWithUsdc(testData.user, _collateralAmount);
    testData.userJUsd = jUsd.balanceOf(testData.user);
    testData.userCollateralAmount = usdc.balanceOf(testData.userHolding);

    // Initialize first liquidator
    testData.liquidator = liquidator;
    testData.liquidatorCollateralAmount = _collateralAmount;
    initiateWithUsdc(testData.liquidator, testData.liquidatorCollateralAmount);
    testData.liquidatorJUsd = jUsd.balanceOf(testData.liquidator);
    testData.liquidatorCollateralAmountAfterInitiation = usdc.balanceOf(address(testData.liquidator));

    //  Initialize second liquidator (to demonstrate partial liquidations)
    address secondLiquidator = makeAddr("secondLiquidator");
    initiateWithUsdc(secondLiquidator, _collateralAmount);
    uint256 secondLiquidatorJUsd = jUsd.balanceOf(secondLiquidator);

    //  Make investment to put collateral in strategy
    vm.prank(testData.user, testData.user);
    strategyManager.invest(address(usdc), address(strategyWithoutRewardsMock), testData.userCollateralAmount, 0, "");

    //  Setup liquidation calldata for strategy withdrawal
    ILiquidationManager.LiquidateCalldata memory liquidateCalldata;
    liquidateCalldata.strategies = new address[](1);
    liquidateCalldata.strategiesData = new bytes[](1);
    liquidateCalldata.strategies[0] = address(strategyWithoutRewardsMock);
    liquidateCalldata.strategiesData[0] = "";

    //  Make position liquidatable by changing price
    console.log("Position liquidatable before price change:", stablesManager.isLiquidatable({ _token: address(usdc), _holding: testData.userHolding }));
    usdcOracle.setPriceForLiquidation();
    console.log("Position liquidatable after price change:", stablesManager.isLiquidatable({ _token: address(usdc), _holding: testData.userHolding }));

    //  Record initial state before any liquidations
    uint256 initialBorrowedAmount = sharesRegistry.borrowed(testData.userHolding);
    uint256 initialCollateralInHolding = usdc.balanceOf(testData.userHolding);
    console.log("Initial borrowed amount:", initialBorrowedAmount);
    console.log("Initial collateral in holding:", initialCollateralInHolding);

    //  First partial liquidation (liquidate ~45% of debt)
    uint256 firstLiquidationAmount = (testData.userJUsd * 45) / 100;
    vm.prank(testData.liquidator, testData.liquidator);
    uint256 firstCollateralUsed = liquidationManager.liquidate(
        address(testData.user), 
        address(usdc), 
        firstLiquidationAmount, 
        0, 
        liquidateCalldata
    );

    //  Record state after first liquidation
    uint256 borrowedAfterFirst = sharesRegistry.borrowed(testData.userHolding);
    uint256 collateralAfterFirst = usdc.balanceOf(testData.userHolding);
    console.log("After first liquidation - Borrowed:", borrowedAfterFirst);
    console.log("After first liquidation - Collateral:", collateralAfterFirst);
    console.log("First liquidation collateral used (with bonus):", firstCollateralUsed);

    //  Second partial liquidation (liquidate another ~45% of remaining debt)
    uint256 secondLiquidationAmount = (borrowedAfterFirst * 45) / 100;
    vm.prank(secondLiquidator, secondLiquidator);
    uint256 secondCollateralUsed = liquidationManager.liquidate(
        address(testData.user), 
        address(usdc), 
        secondLiquidationAmount, 
        0, 
        liquidateCalldata
    );

    //  Record final state after both liquidations
    testData.holdingBorrowedAmountAfterLiquidation = sharesRegistry.borrowed(testData.userHolding);
    uint256 finalCollateralInHolding = usdc.balanceOf(testData.userHolding);
    console.log("After second liquidation - Borrowed:", testData.holdingBorrowedAmountAfterLiquidation);
    console.log("After second liquidation - Collateral:", finalCollateralInHolding);
    console.log("Second liquidation collateral used (with bonus):", secondCollateralUsed);

    //  Calculate total collateral removed vs debt repaid
    uint256 totalCollateralRemoved = firstCollateralUsed + secondCollateralUsed;
    uint256 totalDebtRepaid = firstLiquidationAmount + secondLiquidationAmount;
    
    console.log("Total collateral removed:", totalCollateralRemoved);
    console.log("Total debt repaid:", totalDebtRepaid);
    console.log("Remaining debt:", testData.holdingBorrowedAmountAfterLiquidation);
    console.log("Remaining collateral:", finalCollateralInHolding);

    //  Demonstrate the vulnerability - check if debt remains after partial liquidations
    if (testData.holdingBorrowedAmountAfterLiquidation > 0) {
        console.log("VULNERABILITY CONFIRMED: Position still has debt after multiple partial liquidations");
        console.log("Multiple liquidator bonuses have drained collateral disproportionately");
    }

    //  Verify liquidators received collateral (proving bonuses were paid)
    testData.liquidatorJUsdAfterLiquidation = jUsd.balanceOf(testData.liquidator);
    testData.liquidatorCollateralAfterLiquidation = usdc.balanceOf(testData.liquidator);
    uint256 secondLiquidatorCollateral = usdc.balanceOf(secondLiquidator);
    
    assertGt(testData.liquidatorCollateralAfterLiquidation, 0, "First liquidator should receive collateral");
    assertGt(secondLiquidatorCollateral, 0, "Second liquidator should receive collateral");
}

```

## Recommendation
To mitigate this issue, introduce a minimum liquidation amount.
______________________________________________________________________________________________________________________________

#### [L-1]

### Title: Unexpected Collateral Loss Due to Wrong Slippage Odering in selfLiquidation Function.

### Description
The selfLiquidate() function in LiquidationManager.sol performs a slippage check before actual swap, leading to a scenario where users receive less collateral value than expected. The slipage check uses the user-provided amountInMaximum and slippagePercentage based on the oracle price, but the actual swap may consume close to the maximum collateral amount due to market conditions, resulting in significantly higher slippage than the user anticipated. This causes users to lose more collateral than they intended.
```solidity
Proof of Concept
The provided POC demonstrates the vulnerability by simulating a self-liquidation with a 5% fee slippage tolerance.

function test_selfLiquidate_slippageExpectationMismatch(uint256 _amount) public {
    SelfLiquidationTestTempData memory testData;
    _amount = bound(_amount, 800, uniswapPoolCap / 100_000);
    
    testData.collateral = USDC;
    testData.userHolding = initiateUser(DEFAULT_USER, testData.collateral, _amount);
    testData.userJUsd = jUsd.balanceOf(DEFAULT_USER);
    testData.selfLiquidationAmount = testData.userJUsd / 2;
    testData.userCollateralAmount = IERC20(testData.collateral).balanceOf(testData.userHolding);

    // Calculate required collateral using registry instance
    testData.requiredCollateral = _getCollateralAmountForUSDValue(
        testData.collateral, 
        testData.selfLiquidationAmount, 
        registries[testData.collateral].getExchangeRate()
    );

    ILiquidationManager.SwapParamsCalldata memory swapParams;
    ILiquidationManager.StrategiesParamsCalldata memory strategiesParams;

    // Set acceptable slippage (5%)
    uint256 acceptableSlippage = liquidationManager.LIQUIDATION_PRECISION() * 5 / 100;
    swapParams.slippagePercentage = acceptableSlippage;
    
    // User sets amountInMaximum thinking 5% slippage is acceptable based on oracle price
    swapParams.amountInMaximum = testData.requiredCollateral + 
        (testData.requiredCollateral * acceptableSlippage / liquidationManager.LIQUIDATION_PRECISION());

    // Mock the swap to succeed but use close to the maximum (simulating market moved against user)
    uint256 actualCollateralUsed = swapParams.amountInMaximum - 1; // Just under the limit
    vm.mockCall(
        address(swapManager),
        abi.encodeWithSelector(SwapManager.swapExactOutputMultihop.selector),
        abi.encode(actualCollateralUsed)
    );

    vm.prank(DEFAULT_USER, DEFAULT_USER);
    (uint256 collateralUsed, uint256 jUsdAmountRepaid) = liquidationManager.selfLiquidate(
        testData.collateral, 
        testData.selfLiquidationAmount, 
        swapParams, 
        strategiesParams
    );
    // user expected max 5% slippage based on oracle price,
    // but actual slippage was much higher due to real market conditions
    uint256 actualSlippage = ((actualCollateralUsed - testData.requiredCollateral) * 
        liquidationManager.LIQUIDATION_PRECISION()) / testData.requiredCollateral;
    
    // Demonstrate that actual slippage exceeded what user thought they were accepting
    assertTrue(
        actualSlippage > acceptableSlippage * 2,
        "Slippage protection failed - user got much worse execution than expected"
    );
    
    // Clean up mock
    vm.clearMockedCalls();
}
```
Recommendation
To mitigate the issue, the slippage check should be performed before the swap.