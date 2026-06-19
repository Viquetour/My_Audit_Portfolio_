## Revert Finance Audit Findings

#### [Revert Finance](https://cantina.xyz/code/efb6f308-f13b-4110-aff8-0d67181608dd/overview)

## AutoRange.autoCompound() Silently Overcharges Users by Ignoring maxRewardX64

- **Severity:** Low
- **Likelihood:** Low
- **Impact:** Low
- **Created by:** ViquetourOnTop

## Description

## Summary
The `AutoRange.autoCompound()` charges the global protocol fee rate against every position it compounds, completely ignoring the per-position maxRewardX64 limit that the owner explicitly configured. A user who set `maxRewardX64 = 0` expecting zero autocompound fees is silently charged up to 2%. A user who set `maxRewardX64 = 0.1%` as a ceiling is silently charged 20x that amount. The same contract's `execute()` function correctly enforces this limit , making the omission in `autoCompound()` a clear and asymmetric inconsistency.

## Finding Description
When a position owner calls `configToken()`, they set `maxRewardX64` as an explicit ceiling on fees the protocol can charge against their position.
```solidity
positionConfigs[tokenId] = config;

emit PositionConfigured(
    tokenId, ..., config.maxRewardX64  // emitted — user sees and expects this limit
);
execute() respects this limit unconditionally:
solidityif (params.rewardX64 > config.maxRewardX64) {
    revert ExceedsMaxReward();
}
autoCompound() loads the same config struct, but only checks config.autoCompound. It never reads config.maxRewardX64:
```
```solidity
PositionConfig memory config = positionConfigs[params.tokenId];
if (!config.autoCompound) {
    revert NotConfigured();
}

// fee calculation below never consults config.maxRewardX64

uint256 rewardX64 = totalRewardX64;   // global rate — always 2% at deployment

state.amount0Fees = state.compounded0 * rewardX64 / Q64;
state.amount1Fees = state.compounded1 * rewardX64 / Q64;
```
The global `totalRewardX64` starts at `MAX_REWARD_X64 = Q64 / 50`, exactly 2%, and can only be lowered by the contract owner via `setAutoCompoundReward()`. It cannot be set per-position. The per-position `maxRewardX64` field exists specifically to allow users to cap their individual exposure below the global rate, but `autoCompound()` never reads it.


_________________________________________________________________________

## FlashloanLiquidator Passes Swap Input Amounts as Liquidation Output Minimums, Causing Legitimate Liquidations to Revert

- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** ViquetourOnTop


### Summary
`FlashloanLiquidator` passes the planned swap input amounts as the minimum output thresholds for the vault's liquidation call. These are semantically different quantities. When a position yields marginally less collateral than the liquidator planned to swap due to rounding, partial liquidation coverage, or price movement between off-chain computation and on-chain execution, the liquidation reverts with SlippageError(). A position that should be liquidated survives, and bad debt accumulates in the vault while the window for profitable liquidation closes.

## Finding Description
Inside `uniswapV3FlashCallback`, after borrowing the flash loan, the liquidator calls `vault.liquidate()`.

```solidity
data.vault.liquidate(
    IVault.LiquidateParams(
        data.tokenId,
        data.swap0.amountIn,   // @audit planned swap INPUT for token0
        data.swap1.amountIn,   // @audit planned swap INPUT for token1
        address(this),
        "",
        data.deadline
    )
);
```
Inside `V3Vault.liquidate()`, these values are used as minimum output thresholds on the collateral tokens extracted from the position.

```solidity
(amount0, amount1) = _sendPositionValue(
    params.tokenId, state.liquidationValue, state.fullValue, state.feeValue, params.recipient, params.deadline
);


if (amount0 < params.amount0Min || amount1 < params.amount1Min) {
    revert SlippageError();
}
```
The semantic mismatch is direct. data.swap0.amountIn is "how much token0 I plan to swap back to the asset", a quantity the liquidator computed off-chain based on the expected position composition. amount0Min in LiquidateParams means "the minimum token0 I must receive out of the position", a slippage guard on the liquidation output.
In normal operation, these values happen to coincide because the liquidator sets swap amounts equal to what they expect to receive. But they are not the same thing, and they diverge in the exact scenarios where liquidation is most time-sensitive:
- Partial liquidations: 
- When the position value only partially covers the debt, `_sendPositionValue` returns a proportional subset of the position's tokens. The exact amounts depend on oracle-derived prices and rounding, and may come out slightly below the liquidator's off-chain estimate.

In this case, amount0 < data.swap0.amountIn ->SlippageError() -> the entire flash loan callback reverts ->the liquidation does not happen.
Crucially, the minReward check at the end of the callback already provides meaningful profit protection:
```solidity
balance = data.asset.balanceOf(address(this));
if (balance < data.minReward) {
    revert NotEnoughReward();
}
```
This ensures the liquidator walks away with at least minReward asset tokens regardless of exact token splits. The intermediate SlippageError check adds no protection that minReward does not already provide; it only adds fragility.


## Recommendation
To mitigate this issue, pass 0 for both minimums in the liquidation call and rely on the minReward check that already exists at the end of the callback for profit protection:
```solidity
data.vault.liquidate(
    IVault.LiquidateParams(
        data.tokenId,
        0,    // no minimum on token0 output — minReward provides overall protection
        0,    // no minimum on token1 output — minReward provides overall protection
        address(this),
        "",
        data.deadline
    )
);
```
The minReward parameter passed by the liquidator at call time already enforces that the entire operation is profitable; it checks the liquidator's final asset balance after all swaps complete. 

_________________________________________________________________________

## AutoRange Position Replacement Permanently Corrupts Owner Loan Enumeration With Ghost Entries.


- **Severity:** Low
- **Status:** Duplicate
- **Likelihood:** High
- **Impact:** Medium
- **Created by:** ViquetourOnTop

### Summary
Every time a position is replaced through `transform()`, as happens with every `AutoRange` execution, the old token ID is left behind as a permanent ghost entry in the owner's loan enumeration. The vault adds the new token to the owner's list but never removes the old one. Over time, `loanCount()` and `loanAtIndex()` drift further and further from reality, and any system that relies on these for accurate portfolio accounting receives permanently corrupted data.

### Finding Description
When a transformer sends a new NFT to the vault during an active transform() call, onERC721Received detects the replacement and runs the following sequence:

```solidity
if (tokenId != oldTokenId) {
    address owner = tokenOwner[oldTokenId];

    transformedTokenId = tokenId;

    uint256 debtShares = loans[oldTokenId].debtShares;
    loans[tokenId] = Loan(debtShares);

    _addTokenToOwner(owner, tokenId);        // pushes newTokenId into ownedTokens ✓
    emit Add(tokenId, owner, oldTokenId);

    _cleanupLoan(oldTokenId, debtExchangeRateX96, lendExchangeRateX96);  // ✗ no owner cleanup

    _updateAndCheckCollateral(tokenId, ...);
}
```
The new token is registered, the old loan is deleted but the old token ID is never removed from ownedTokens. Looking at `_cleanupLoan` itself makes the gap explicit:

```solidity
function _cleanupLoan(uint256 tokenId, uint256 debtExchangeRateX96, uint256 lendExchangeRateX96) internal {
    _updateAndCheckCollateral(tokenId, debtExchangeRateX96, lendExchangeRateX96, loans[tokenId].debtShares, 0);
    delete loans[tokenId];
    // _removeTokenFromOwner is never called
}
```

After every position replacement the owner's enumeration state looks like this:
```solidity
ownedTokens[owner]     = [oldTokenId, newTokenId]  // oldTokenId is a ghost
tokenOwner[oldTokenId] = owner        //never cleared
loans[oldTokenId]      = deleted            //no backing record
loanCount(owner)       = 2                       //should be 1
```
This is distinct from the liquidation path, where the same `_cleanupLoan` is called, but the ghost is intentional and temporary. The owner is expected to call `remove()` to retrieve their empty NFT, which clears the enumeration entry at that point. The transform-replacement path has no equivalent. The old NFT has no value to retrieve, no prompt exists to call remove(), and the UX contract of `AutoRange` is that the entire operation is self-contained. A user who never discovers they need to manually call remove(oldTokenId) accumulates one ghost entry per range change, permanently.

## Recommendation
The fix is a single additional call in onERC721Received before `_cleanupLoan` runs on the old token:
```solidity
if (tokenId != oldTokenId) {
    address owner = tokenOwner[oldTokenId];

    transformedTokenId = tokenId;

    uint256 debtShares = loans[oldTokenId].debtShares;
    loans[tokenId] = Loan(debtShares);

    _addTokenToOwner(owner, tokenId);
    emit Add(tokenId, owner, oldTokenId);

    _removeTokenFromOwner(owner, oldTokenId);  // ← add this line
    _cleanupLoan(oldTokenId, debtExchangeRateX96, lendExchangeRateX96);

    _updateAndCheckCollateral(tokenId, ...);
}
```
This mirrors exactly what `remove()` does when an owner manually cleans up after liquidation, and it keeps the enumeration state consistent without touching any other code path. The liquidation path correctly continues to defer cleanup to` remove()`; that behavior is unchanged.
