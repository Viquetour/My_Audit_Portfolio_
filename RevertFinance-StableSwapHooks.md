# Revert Finance - StableSwap Hooks Audit findings.

### [Revert Finance - StableSwap Hooks](https://cantina.xyz/code/e55ee7b9-6c99-42f8-8338-39f3dd134ef3/overview)


## Missing slippage protection in addLiquidity allows sandwich attacks


- **Severity:** Medium
- **Likelihood:** High
- **Impact:** High
- **Created by:** ViquetourOnTop


### Summary
`StableSwapZapIn.sol` passes zero slippage parameters to the inner `addLiquidity` call, allowing an attacker to sandwich the victim's zapIn transaction, extracting 33% of their LP shares at no protocol-enforced cost.

### Finding Description
In `StableSwapZapIn._addLiquidityAndGetShares`, it calls :
```solidity
_hooks.addLiquidity(balances, minAmounts, minShares)
```
where `minAmounts` is hardcoded to an array of zeros and `minShares` is forwarded from the caller-supplied `_minShares` parameter, which the zapIn interface accepts but does not enforce meaningfully, as the inner liquidity addition has no floor on per-token amounts received.

For example:

1. Victim calls `quoteZapIn` off-chain, receiving a `SwapQuote[]` and an expected share count based on current pool state.
2. Attacker observes the pending zapIn transaction in the mempool and frontruns with a large directional swap (token0 - token1), imbalancing the pool and shifting the internal price.
3. Victim's transaction executes with the stale `SwapQuote[]`. The zap swaps `token0` for `token1` at the now-unfavorable rate, then calls `addLiquidity` with `minAmounts=[0,0]` and `minShares=0`.
4. Because no slippage floor is enforced at the liquidity addition step, the protocol accepts the imbalanced deposit silently, minting significantly fewer shares than quoted.
5. Attacker backruns to restore pool balance, capturing the spread.

The victim has no on-chain recourse. Even if they supply a nonzero `_minShares` to zapIn, the vulnerability lies in the unchecked inner `addLiquidity` call, which always passes zero per-token minimums.

### Recommendation
To mitigate this issue propagate caller-supplied slippage bounds all the way into the inner addLiquidity call.
```solidity
// @audit:  (vulnerable)
_hooks.addLiquidity(balances, new uint256[](balances.length), _minShares);

// Fix
uint256[] memory minAmounts = new uint256[](balances.length);
for (uint256 i = 0; i < balances.length; i++) {
    minAmounts[i] = quotedAmounts[i] * (BPS_BASE - maxSlippageBps) / BPS_BASE;
}
_hooks.addLiquidity(balances, minAmounts, _minShares);
```

