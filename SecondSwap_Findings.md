# Code4rena
### [SecondSwap](https://code4rena.com/audits/2024-12-secondswap)


Front Running attack on `transferVesting ()` function in `SecondSwap_StepVesting` contract
Avatar for Viquetour
Viquetour

- **Severity:** Medium


Finding description
The `transferVesting()` function in the `SecondSwap_StepVesting` contract is vulnerable to front-running attacks. This vulnerability exists because transactions are visible in the mempool before being mined, paving the way for malicious actors to observe and react to pending transactions. An attacker submits a transaction with a higher gas price, prioritizing its execution over legitimate transactions. This can disrupt the vesting process.


Proof of Concept
garanto has a total vesting amount of 1000 tokens with 20 claimed `amountClaimed = 200`
A legitimate user submits a transaction to transfer 50 tokens to `_beneficiary`
Exploit Scenerio:
1. An attacker or bot observes the legitimate transaction in the mempool, and submits his own transferVesting transaction first, transferring 700 tokens to themselves. which will be `(_beneficiary == maliciousAddress)` reducing grantors uncaimed balance to 100.

The legitimate transaction fails due to the require statement `require( grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount, "SS_StepVesting: insufficient balance" )` The truth of this invariant will no longer hold.

The attacker successfully front-runs the legitimate transfer and gets away with a significant portion of the tokens.

Recommended mitigation steps
Ensure that, the updated `_amountClaimed` is never greater than the `_totalAmount`.

Add a unique nonce or timestamp to the function and require it to match or exceed a previous value for the `_grantor` vesting schedule. This will make it harder to manipulate the order of transactions.

```solidity
require(block.timestamp >= grantorVesting.lastUpdatedTim)
```