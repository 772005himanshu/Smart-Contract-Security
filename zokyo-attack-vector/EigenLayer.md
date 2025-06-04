## Eigen Layer - Offer re-Staking services for Etherum

Allowing users to re-purpose their Staked Assets for Securing Additional Decentrailzed Services

### Denial of Service in NodeDelegator Due to EigenLayer's `maxPerDeposit` Check

The Vulnerability in the NodeDelegator contract's `depositAssetIntoStrategy` function introduce potentially Denial of Service(DoS) issue , arising From the interaction with `Eigenlayer's ` `maxPerDeposit` limit . When the NodeDelegator attempts to deposit assets into a strategy managed by EigenLayer,it Triggger a check in EigenLayers `StrategyBaseTVLimits` contract to ensure that the despoited amount doesnot exceed the maximum allowed per deposit (maxPerDeposit)

If the Deposit exceeds this limit, the transaction is reverted , leaving the token in the NodeDelegator Contract Without being Deposited.

This can cause a DoS condition for the NodeDelegator contract because subsequent attempts to deposit assets will fail unless manual intervention is taken to resolve the issue

#### Impact of the Vulnerability
- Denial of Service(DoS) 
- Operational Disruption 
- Blocked Transaction

### Incorrect Share Issuance Due to Strategy Updates in EigenLayer Integerations

The Process of Updating Strategies is Crucial to maintaing optima; performance and returns , If the Protocol does not Correctly account for all assets when switching , it can lead to the `incorrrect issuance of shares`, it lead to significant risk to the financial integrity of the Protocol

#### How the Vulnerability Ocuurs in EigenLayer Integeration
1. Asset Deposit into Original Strategy
2. Strategy Update(the issue arises when the protocol updates to a new strategy but fails to account for assets that remain staked in the previous strategy)
3. Incorrect Share Calculation
4. Financial Discrepancy(As a result, new depositors receive more shares than they are entitled to, diluting the value of shares held by previous users.)

#### Causes of the Vulnerability in EigenLAyer Integeration
1. Failure to Account for Old Strategy Assets
2. Incomplete Migration of Assets( During the strategy update, assets from the old strategy may not be fully migrated to the new one)
3. Outdated daat in Asset Pricing Feed(EigenLayer re-staking involves dynamic price feeds and asset valuation. If the protocol uses outdated or incorrect price data from the old strategy,)

#### Impacted of Existing Shares
- Dilution of Existing Shares
- Unfair Advantage for New Depositors
- Financial Instability

#### Mitigation Strategies For Eigenlayer Integration
1. Ensure Complete Accounting During Strategy Updates
it’s essential to ensure that all assets from both the old and new strategies are accounted for in the total asset balance

- Tracking the balance of both the old and new strategies during the update process.

- Ensuring that the total asset value reflects all staked assets, including those held in the old strategy, to maintain accurate share calculations.

2. Migratioon Assets Between Strategies
Implement a robust migration mechanism to move assets from the old strategy to the new strategy in EigenLayer.This will help prevent assets from being left behind in the old strategy, which can cause miscalculations during share issuance.


### NonReentrant Vulnerability in EigenLater Withdrawals

While nonReentrant modifier is used to protect against reentrancy attacks, there are scenario where reentrancy is neccesary for the proper functioning of a contract , especially in the context of cross-chain operation or complex workflows like EigenLayer Withdrawals

When a nonReentrant modifier is applied too broadly , preventing legitmate operations that require reentrancy. In case of EigenLayer , the OperatorDelegate contract uses the nonReentrant modifier in its `receive()` function, which blocks ETH withdrawal. This leads to withdrawals being Permanenetly stuck, as the contract prevents the reentrancy required for successful processing these transaction 

#### How the Vulnerability Occurs in EigenLayer Withdrawals
1. Initiating Withdrawals in EigenLayer
In EigenLayer, users can stake assets (including ETH) across multiple strategies. When a user initiates a withdrawal, the `OperatorDelegator` contract is responsible for processing the queued withdrawals and returning the assets to the user. This process involves multiple steps, and for ETH withdrawals, reentrancy is required to complete the transaction flow between different contracts (e.g., the `EigenPod` and `OperatorDelegator`)

2. Blocking Reentrancy in the Recieve Function
The `receive()` function in the `OperatorDelegator` contract is called during ETH withdrawals. However, because the `nonReentrant` modifier is applied to this function, it prevents the contract from handling the reentrant call that is needed to complete the ETH withdrawal. This leads to the withdrawal transaction being permanently stuck.

3. Permanent Stuck Withdrawals
Since ETH withdrawals rely on a reentrant call to complete the process, the nonReentrant modifier effectively blocks the withdrawal of any ETH from EigenLayer. As a result, users cannot withdraw their ETH, and the assets remain stuck in the contract indefinitely.