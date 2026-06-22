# Contest Findings

A curated collection of smart contract security audit contest findings discovered across multiple competitive audit platforms. Each report documents vulnerabilities identified during public audit contests, including severity classifications, proof-of-concept tests, and recommended mitigations.

## Repository Structure

| File | Platform | Contract(s) Audited | Severities |
|------|----------|-------------------|------------|
| [`Jigsaw_Findings.md`](./Jigsaw_Findings.md) | Cantina | Jigsaw Contracts | H-1 |
| [`mighty-contracts findings.md`](./mighty-contracts%20findings.md) | Cantina | Mighty Contracts | High, Medium |
| [`RevertFinance Audit-findings.md`](./RevertFinance%20Audit-findings.md) | Cantina | Revert Finance (AutoRange, Lending) | Low |
| [`RevertFinance-StableSwapHooks.md`](./RevertFinance-StableSwapHooks.md) | Cantina | Revert Finance - StableSwap Hooks (ZapIn) | Medium |
| [`SecondSwap_Findings.md`](./SecondSwap_Findings.md) | Code4rena | SecondSwap (StepVesting) | Medium |
| [`liquity_book_vault_Audit_findings.md`](./liquity_book_vault_Audit_findings.md) | Cantina | Liquity Book Vaults | M-1 |

## Findings Overview

### High Severity

| Finding | Contract | Summary |
|---------|----------|---------|
| Partial Liquidation Without Minimum Amount | Jigsaw | Multiple partial liquidations can leave positions with unbacked debt due to cumulative liquidation bonuses depleting collateral disproportionately. |
| Missing Stale Price Threshold and Expo Handling | Mighty Contracts | `PrimaryPriceOracle` lacks proper staleness checks and expo normalization, returning raw Pyth prices without decimal scaling. |
| No Partial Liquidation as Documented | Mighty Contracts | The liquidation system cannot partially repay debt when the liquidator's collateral is insufficient for full repayment, contradicting documented behavior. |

### Medium Severity

| Finding | Contract | Summary |
|---------|----------|---------|
| Front-Running on `transferVesting()` | SecondSwap | Attacker can front-run vesting transfers by observing pending transactions in the mempool and executing with higher gas priority. |
| Missing Slippage Protection in `addLiquidity` | Revert Finance - StableSwap Hooks | ZapIn passes zero slippage parameters to inner `addLiquidity`, enabling sandwich attacks that can extract 33%+ of LP shares. |

### Low Severity

| Finding | Contract | Summary |
|---------|----------|---------|
| `autoCompound()` Ignores `maxRewardX64` Limit | Revert Finance | AutoRange's `autoCompound()` charges the global protocol fee (2%) against every position, bypassing the per-position `maxRewardX64` ceiling configured by the owner. |
| Inconsistent Vault Creation Fee | Liquity Book Vaults | Protocol documents a fixed 400 $S creation fee but implements a hardcoded 75 ETH fee, causing unpredictable fiat costs as ETH price fluctuates. |

## Audit Platforms

- **[Cantina](https://cantina.xyz/)** — Competitive audit platform for smart contracts (Revert Finance, Jigsaw, Mighty Contracts, Liquity Book Vaults)
- **[Code4rena](https://code4rena.com/)** — Decentralized audit competition platform (SecondSwap)

## Key Vulnerability Themes

The findings in this repository span several recurring vulnerability categories:

- **Price Oracle Manipulation / Misconfiguration** — Stale price checks, missing expo normalization, ETH-denominated fees
- **Front-Running & MEV** — Mempool-visible transactions enabling transfer and sandwich attacks
- **Inconsistent Parameter Enforcement** — Global settings overriding per-user configurations (`maxRewardX64`, slippage bounds)
- **Liquidation Logic Flaws** — Partial liquidation without minimums, missing pre-liquidation solvency checks

## Purpose

This repository serves as a personal knowledge base and portfolio of security research conducted during competitive audit contests. Each finding includes:

- A detailed description of the vulnerability
- The attack scenario or proof of concept
- The root cause and affected code paths
- A concrete recommendation for remediation

## License

This project is for educational and portfolio purposes. Vulnerability reports are the original work of the author, submitted as part of competitive audit contests on their respective platforms. All code snippets referenced are from publicly available, contest-scoped codebases.