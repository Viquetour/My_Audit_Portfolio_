## liquidity-book-vaults Audit-findings

##### [liquidity-book-vaults](https://cantina.xyz/code/076935b1-2706-48c6-bf0a-b3656aa24194/overview)

### [M-1]


###### Description
The protocol documentation advertises a fixed one-time creation fee of 400 $S(stableCoin) but the smart contract implements a hardcoded ETH prone to price volatility. This design choice results in an inconsistent fiat cost(USDC) for users creating vaults, as the ETH price fluctuates in the market. When the price of ETH rises, users must pay more in fiat terms to meet the required Ether amount, potentially overpaying compared to the intended documented one time creation fee. This introduces unpredictability and unfairness as the cost of vault creation varies depending on market creation at the time of the transaction.

Proof of Concept
```solidity
function initialize2(address owner) public initializer {
        if (owner == address(0)) revert VaultFactory__InvalidOwner();

        __Ownable2Step_init();
        _transferOwnership(owner);

        _setDefaultOperator(owner);
        _setFeeRecipient(owner);

     @>   _creationFee = 75 ether;
    }

    function initialize3(address owner) public initializer {
        if (owner == address(0)) revert VaultFactory__InvalidOwner();

        __Ownable2Step_init();
        _transferOwnership(owner);

        _setDefaultOperator(owner);
        _setFeeRecipient(owner);

     @>   _creationFee = 75 ether; //!@audit:- If ethprice rises users over payif it crashes they underpay
        _defaultMMAumFee = 0.1e4; // 10% fee
    }
```
Example:

If 1 ether equals 2,000, the fee of 75 ether amounts to 150,000 If the same ether rises to 3,000, the fee of 75 ether now costs 225,000, resulting in users overpaying in fiat terms

This highlights how the ether price fluctuation directly impacts the fiat cost of the creation fee leading to direct deviation of the intended value.

#### Recommendation
Modify the contract to accept creation fee in stable coins like USDC OR DAI. This ensures the creation fee remains consistent unaffected by crypto currency market swings.


______________________________________________________________________________________________________________________________

### [L-1]

#### Title: No Function to Retrieve Native Tokens

The protocol includes a recoverERC20 function that helps in recovering ERC20 tokens from a given vault. However, it lacks a corresponding function to retrieve native tokens from a given vault. This omission could lead to native tokens being permanently locked in a vault if sent accidentally, as there is no mechanism for the owner to retrieve them.

Proof of Concept
```solidity
function recoverERC20(IBaseVault vault, IERC20Upgradeable token, address recipient, uint256 amount)
        external
        override
        onlyOwner
    {
        vault.recoverERC20(token, recipient, amount);
    }
```
This function allows the owner to retrieve tokens from the vault by calling the vault's recoverERC20 function.

However, there is no equivalent function to recover native ETH from the vault. The absence of the native token recovery function is evident when going through the protocol's public, external, and internal functions, none of which address native token recovery.

Recommendation
To address this issue, implement a native token recovery function in the protocol contract to allow users to recover native tokens when accidentally sent.


____________________________________________________________________________

#### [L-2]
##### Description

The deposit function does not check if the vault is flagged for shutdown. Allowing users to make deposit even when the vault is flagged for shutdown, this violates the intended protocol design. According to the shutdown design, deposits should be paused when flagged for shutdown, as a means of protecting users' funds during the shutdown phase before the future emergency mode. However, the deposit function calls updatePool, which only checks if the vault is in emergency mode or doesn't exist (address(0)) but, fails to verify the vaults shutdown status. This allows users to deposit even during the shutdown phase, potentially leading to operational inconsistencies during the shutdown phase.

Proof of Concept
```solidity
/**
     * @notice This will flag the vault as shutdown candidate and possible emergency mode in the near future
     * The flag indicate users to withdraw from the vault and claim any rewards before vault is in emergency mode.
     *
     * @dev Flag the vault for shutdown, withdraws all funds from strategy
     */
    function submitShutdown() external override onlyOperators {
        if (_flaggedForShutdown) revert BaseVault__AlreadyFlaggedForShutdown();

        _flaggedForShutdown = true;

        emit ShutdownSubmitted();
    }
```    
```solidity    
function deposit(uint256 amountX, uint256 amountY, uint256 minShares)
    
        public
        virtual
        override
        nonReentrant
        returns (uint256 shares, uint256 effectiveX, uint256 effectiveY)
    {
        _updatePool();

        // Calculate the shares and effective amounts, also returns the strategy to save gas.
        IStrategy strategy;
        (strategy, shares, effectiveX, effectiveY) = _deposit(amountX, amountY);
```       
The _updatePool function contains the following check
```solidity
function _updatePool() internal override {
   @>     if (address(getStrategy()) == address(0)) {
            return;
        }
```        
This check only verifies if the vault is in emergency mode or inactiveaddress(0) but does not check for the _flaggedForShutdown boolean, which indicates whether the vault is flagged for shutdown. As a result, deposit proceeds even after _flaggedForShutdown = true bypassing the intended restriction.

Example:

An Operator flagged the vault for shutdown, setting __flaggedForShutdown = true
A user calls deposit with a valid amountX, amountY and minShares
_updatePool checks if (address(getStrategy()) == address(0)) which passes because the strategy exist.
The deposit proceeds with minting shares and locking user's funds in a vault that is flagged for shutdown.
Recommendation
Modify the updatePool function to check for the flagged shutdown status and revert if the vault is flagged for shutdown.
```solidity
function _updatePool() internal override {
  -     if (address(getStrategy()) == address(0)) {
            return;
  +    if(_flaggedForShutdown && address(getStrategy() == address(0))
        }
```  

__________________________________________________________


### User Funds Locked During Vault ShutDown Process

- **Severity:** Low
- **Likelihood:** Medium
- **Impact:** Medium
- **Created by:** ViquetourOnTop

## Description

## Description
The  `emergencyWithdraw` function is intended to allow users to immediately withdraw their funds when system is flagged for shutdown and during emergency mode. However, due to an incorrect condition check, users can't use the `emergencyWithdraw` function during the flagged shutdown phase, which violates the intended behaviour of the function as described in the natspec.
 
## Proof of Concept

```
/**
     * @notice This will flag the vault as shutdown candidate and possible emergency mode in the near future
     * The flag indicate users to withdraw from the vault and claim any rewards before vault is in emergency mode.
     *
     * @dev Flag the vault for shutdown, withdraws all funds from strategy
     */
    function submitShutdown() external override onlyOperators {
        if (_flaggedForShutdown) revert BaseVault__AlreadyFlaggedForShutdown();

        _flaggedForShutdown = true;

        emit ShutdownSubmitted();
    }

```
When in shutdown mode the strategy remains active `(_strategy != address(0))`
```
/**
     * @notice Emergency withdraws from the vault and sends the tokens to the sender according to its share.
     * If the user had queued withdrawals, they will be claimable using the `redeemQueuedWithdrawal` and
     * `redeemQueuedWithdrawalNative` functions as usual. This function is only for users that didn't queue
     * any withdrawals and still have shares in the vault.
     * @dev This will only work if the vault is in emergency mode or flagged for shutdown
     */
    function emergencyWithdraw() public virtual override nonReentrant {
        // Check that the vault is in emergency mode.
     @>   if (address(_strategy) != address(0)) revert BaseVault__NotInEmergencyMode();
```

During flagged shutdown phase, `_flaggedForShutdown = true` but `_strategy) != address(0)`. This check blocks users from calling emergencyWithdraw.


## Recommendation
Update the check to allow withdrawals during both flagged shutdown and emergency mode.

```
 function emergencyWithdraw() public virtual override nonReentrant {
        // Check that the vault is in emergency mode.
   -       if (address(_strategy) != address(0)) revert BaseVault__NotInEmergencyMode();
  +        if (!_flaggedForShutdown) && address(_strategy) != address(0)) revert BaseVault__NotInEmergencyMode();


        _updatePool();

        // Get the amount of shares the user has. If the user has no shares, it will revert.
        uint256 shares = balanceOf(msg.sender);
        if (shares == 0) revert BaseVault__ZeroShares();

 if (!_flaggedForShutdown) && address(_strategy) != address(0)) revert BaseVault__NotInEmergencyMode();
```

