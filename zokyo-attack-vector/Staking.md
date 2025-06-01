## Staking Mechanism

1. First Depositor inflation Attack in Staking Contract

The first Depositer Inflation Attack is sophisticated vulnerability that allows the first depositer in staking contract to manipulate the distribution of staking shares

This attack leverages the fact that the first depositor holds a significant advantage when no prior assets exist in the contract (i.e., totalSupply is 0).

By strategically inflating the contract’s assets and exploiting share calculations, the attacker can drain future depositors' assets. A key challenge with this attack is its potential for fromt-running , which makes difficult for regular users to avoid

Example of the Attack:

- Alice deposits 10 wei to mint 10 shares.

- Alice transfers 100e18 of a staking token to the staking contract, inflating the total assets.

- Bob attempts to deposit 19 ETH, but Alice front-runs Bob's transaction and ensures she benefits from the manipulated share-to-asset ratio.

- Bob receives only 1 wei of shares due to the rounding errors, and Alice withdraws 108 ETH, resulting in an 8 ETH loss for Bob.

Example of Vulnerable Staking Contract that could be exploited by the first depositer Inflation Attack

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract VulnerableStakingContract {
    mapping(address => uint256) public balanceOf;
    mapping(address => uint256) public shares;
    uint256 public totalShares;
    uint256 public totalAssets;

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    // Vulnerable deposit function
    function deposit() public payable {
        uint256 sharesToMint;

        // Step 1: Check if this is the first deposit (no assets, no shares)
        if (totalShares == 0) {
            // First depositor receives shares equal to their deposit amount
            sharesToMint = msg.value; 
        } else {
            // Step 2: Convert the deposit to shares based on the current ratio
            sharesToMint = (msg.value * totalShares) / totalAssets;
        }

        // Step 3: Mint shares to the depositor
        shares[msg.sender] += sharesToMint;
        totalShares += sharesToMint;

        // Step 4: Add the deposit to total assets
        totalAssets += msg.value;

        emit Deposited(msg.sender, msg.value);
    }

    // Vulnerable withdrawal function
    function withdraw(uint256 _shares) public {
        require(_shares <= shares[msg.sender], "Insufficient shares");

        // Step 1: Calculate how much ETH the user can withdraw based on their shares
        uint256 ethAmount = (_shares * totalAssets) / totalShares;

        // Step 2: Reduce the user's shares and the total shares
        shares[msg.sender] -= _shares;
        totalShares -= _shares;

        // Step 3: Reduce the total assets and send the ETH back to the user
        totalAssets -= ethAmount;
        payable(msg.sender).transfer(ethAmount);

        emit Withdrawn(msg.sender, ethAmount);
    }
}
```

### How the Attack Works:
- Alice deposits first: If Alice is the first depositor, she deposits a small amount (e.g., 10 wei of ETH), receiving an equivalent number of shares.

- Alice inflates the assets: After minting her shares, Alice sends a large amount of another asset (e.g., 100e18 of a staking token) directly to the contract, inflating the totalAssets without creating new shares.

- Bob deposits second: When Bob deposits 19 ETH, the inflated totalAssets causes the share calculation to round down. Bob only receives 1 share instead of a fair amount, while Alice can later withdraw a disproportionately large share of the total assets, effectively stealing Bob’s funds.


2. Front-Running Rebase attack (Stepwise Jump in Rewards)
The front-running rebase attack targets staking contract where rewards are distributed in a batch or rebse event. Attackers time their deposit to take advantage of predictable rewards distribution and withdraw a disproportionate amount of rewards at expense of ligitimate stakers.

like we learn in owen trum tutorial setwise increasing rewards , attacker can front run just before the rewards distribution and take the rewards in less time of staking 

the shares should be predicated on the bases of the contribution not the amount they staked in !

3. Rugability of a poorly Implemented recoverERC20 function in staking Contract

In staking Contract , a common feature is the ability for the Contract owner to recover accidentlly sent ERC20 tokens.This is usually done through a function like `recoverERC20` , which allows the contract owner to withdraw tokens that are not meant to be part of the staking or rewards mechanism.

- If the Function is not carefully implemented it opens the door to rug pulls where the owner can maliciously or unitentionally drain the rewards token from the Contract.

Vulnerability: `recoverERC20` Misuse
- The `recoverERC20` function is designed to allow the contract owner to withdraw tokens that might have been sent to staking contract by mistake 

- no safeguards are put in place , this function can also be used to withdraw the rewards tokens that have accumulated in the Contract , effectively draining the rewards pool and rugging the legitimate stakers.

```solidity
// Example of a vulnerable recoverERC20 function
function recoverERC20(address tokenAddress, uint256 tokenAmount) external onlyOwner {
    // Vulnerable: No check if the token being recovered is the rewardsToken
    IERC20(tokenAddress).transfer(owner(), tokenAmount);
}

```

The primary impact of this vulnerability is the complete loss of rewards for stakers, as the owner has the ability to drain the rewards pool at any time. This not only results in financial loss for users but also damages the reputation of the staking platform, as it creates a clear rug pull scenario where users’ expectations of receiving rewards are undermined by the contract owner’s actions.

4. General Consideration for ERC777 Reentrancy Vulnerabilities

- ERC777 tokens introduce new features compared to the more common ERC20 standard , one of which is `hooks mechanism`
- This feature allows a recipient to register a function ("hook") that will be automatically called when token are transferred to them.
- This will introduce the `reentrancy risk` occurs when an attacker uses the hook mechanism to `re-enter` the contract during a transfer potentially exploiting vulnerabilities such as draining rewards or other assets 

- Use `check-effects-interaction` pattern - attacker can manipulate the contract's logic , draining rewards or other assets the execution of rewards distribution function

Example of Vulnerable Function

```solidity
function claimRewards(address user, IERC20[] memory _rewardTokens) external {
    for (uint8 i = 0; i < _rewardTokens.length; i++) {
        uint256 rewardAmount = accruedRewards[user][_rewardTokens[i]];

        // Check for zero rewards
        if (rewardAmount == 0) revert("Zero rewards");

        // Vulnerability: Transferring rewards before clearing accruedRewards
        _rewardTokens[i].transfer(user, rewardAmount);

        // Clear the user's rewards after the transfer (Too late! Vulnerable to reentrancy)
        accruedRewards[user][_rewardTokens[i]] = 0;

        emit RewardsClaimed(user, _rewardTokens[i], rewardAmount);
    }
}    
```

4. Vulnerability: _lpToken and Reward Token Confusion in Staking Contract

This vulnerability arises in staking Contract where both the staking token (`_lpToken`) and reward token are treated similarly .If the staking token used in a new pool is the same as the reward token , the reward calculation becomes flawed . This is because the balance of the staking token is often used to calculate rewards and if the staking token is the same as the reward token , the contract rewards balance will incorrectly inflate the staking token balance , leading to improper reward distribution

This flaws allowed for rewards to be under-allocated to legitmate stakers as the balance used for reward calculation will be artifically  high due to rewards being add to same balance of the staking tokens. The incorrect calculation reduces the rewards that should have been distributed to stakers, leading to stakers receiving less than they are entitled to.

Impact:
If the staking token `_lpToken` is the same as the reward token

- The staking pool’s balance is calculated using the balance of `_lpToken`. However, if the rewards (which are also in the form of the reward token) are deposited into the contract, they will inflate the total supply of `_lpToken`. Then this inflated balance will be reward-per-share for stakers

- Since the balance used to calculate rewards (i.e., `lpSupply`) is inflated by the reward token itself, each staker’s rewards will be reduced

```solidity
function add(
    uint256 _allocPoint,
    IERC20 _lpToken,
    IRewarder _rewarder,
    bool _withUpdate
) public onlyOwner {
    if(_withUpdate) {
        massUpdatePools();
    }

    uint256 lastRewardBlock = block.number > startBlock ? block.numner : startBlock;

    totalAllocPoint = totalAllocPoint.add(_allocPoint);
    poolInfo.push(
        poolInfo({
            lpToken: _lpToken,
            allocPoint: _allocPoint,
            lastRewardBlock : lastRewardBlock,
            accCvxPerShare: 0,
            rewarder: _rewarder
        })
    );
}
```

### flawed
- No check is made to ensure that `_lpToken` is not the same as the reward token (e.g., cvx in the case of Convex)

- If `_lpToken` is the same as `cvx`, the balance of `cvx` in the contract will be inflated by the rewards being added, leading to under-allocation of rewards for stakers.

```solidity
function updatePool(uint256 _pid) public {
    PoolInfo storage pool = poolInfo[_pid];
    if (block.number <= pool.lastRewardBlock) {
        return;
    }
    uint256 lpSupply = pool.lpToken.balanceOf(address(this)); // Vulnerable: Includes reward tokens in balance
    if (lpSupply == 0) {
        pool.lastRewardBlock = block.number;
        return;
    }
    uint256 multiplier = getMultiplier(pool.lastRewardBlock, block.number);
    uint256 cvxReward = multiplier.mul(rewardPerBlock).mul(pool.allocPoint).div(totalAllocPoint);
    pool.accCvxPerShare = pool.accCvxPerShare.add(cvxReward.mul(1e12).div(lpSupply)); // lpSupply is inflated
    pool.lastRewardBlock = block.number;
}
```

Recommendation:
- The lack of ensure the staking token `_lpToken` is not same as `reward` token introduces a significant vulnerability in staking contract

5. Slippage Checks

- MEV Front -running Exploit During Withdrawals 
- ERC4626 vault, MEV front-running exploits can occur when a user’s withdrawal involves swaps or price-sensitive interactions.
- An MEV bot can profit by manipulating the swap prices or creating slippage that negatively impacts the user’s withdrawal amount
- Protection strategies like slippage controls, TWAPs, and using Flashbots/private transactions can help mitigate these risks and provide users with more predictable withdrawal outcomes.

6. The harvest Functionality in Vaults: Issue and Best Practices

- The vaults, the harvest function is a criticial features responsible for collecting rewards or yields generated by the vaults underlying strategies . The Proper implemenatation of this function generated by the vaults underlying strategies.
- The Proper implementation of this function ensures that users can maximize the returns from the deposited assets . however forgetting to implement the harvest function , calling it incorrectly and invoking it at the wring time can lead to significant issues such as `loss of yields`, unclaimed rewards or inequitable distribution among user


Issue 1- Calling Harvest After Deposits and Withdrawals
One common problem is when the harvest function is called after a user deposits or withdraws from the vault, which can lead to diluted rewards for existing users or loss of yield for withdrawing users.

```Solidity
// Vulnerable harvest logic: called after share calculation
function deposit(uint256 amount) external {
    uint256 sharesToMint = (totalShares == 0)
        ? amount
        : (amount * totalShares) / totalAssets;

    shares[msg.sender] += sharesToMint;
    totalShares += sharesToMint;
    totalAssets += amount;

    // Harvest is called AFTER share calculation, causing dilution
    harvest();  

    asset.transferFrom(msg.sender, address(this), amount);
}

function withdraw(uint256 shareAmount) external {
    uint256 amountToWithdraw = (shareAmount * totalAssets) / totalShares;

    shares[msg.sender] -= shareAmount;
    totalShares -= shareAmount;
    totalAssets -= amountToWithdraw;

    // Harvest is called AFTER withdrawal, user loses yield
    harvest(); 

    asset.transfer(msg.sender, amountToWithdraw);
}
```

Mitigation:

To avoid this issue , the harvest function should be called before calculating the users shares during deposit and withdrawal operations. This ensures that the vault total asset balance is updated `before` minting new shares or withdrawing assets

```solidity
function deposit(uint256 amount) external {
    // Call harvest before share calculation to prevent dilution
    harvest();

    uint256 sharesToMint = (totalShares == 0)
        ? amount
        : (amount * totalShares) / totalAssets;

    shares[msg.sender] += sharesToMint;
    totalShares += sharesToMint;
    totalAssets += amount;

    asset.transferFrom(msg.sender, address(this), amount);
}

function withdraw(uint256 shareAmount) external {
    // Call harvest before withdrawal to update user's share of the yield
    harvest();

    uint256 amountToWithdraw = (shareAmount * totalAssets) / totalShares;

    shares[msg.sender] -= shareAmount;
    totalShares -= shareAmount;
    totalAssets -= amountToWithdraw;

    asset.transfer(msg.sender, amountToWithdraw);
}
```

Issue 2 : Forgetting To Harvest During Adapter Changes
Another problem can occur when changing the vault's adapter (i.e., the module responsible for connecting the vault to a specific strategy). If the harvest function is not called during the adapter change process, any unharvested rewards stored in the previous strategy may be lost, as the vault switches to the new adapter without claiming these rewards

### How It Happens:

1. The vault’s adapter is changed by the owner or governance.

2. If the harvest() function is on cooldown or fails to execute, the vault switches adapters without claiming the unharvested rewards from the previous strategy.

3. Users lose their share of the unclaimed rewards.


```solidity
// Vulnerable adapter change logic: forgets to harvest
function changeAdapter(address newAdapter) external onlyOwner {
    require(newAdapter != address(0), "Invalid adapter");

    // Adapter switch without harvesting the unclaimed rewards
    adapter.redeem(); 
    adapter = newAdapter;
}
```

Impact:
- User lose the yield or rewards stroed in the previous adapter strategy

Mitigation:
Always ensure that the harvest() function is called before switching adapters to collect the outstanding rewards from the old strategy. This ensures that no rewards are left unclaimed.

Issue 3: Timing of Harvest and Cooldown issues
The timing of calling the harvest function can also lead to issues if it is not properly handled. Some strategies might have a cooldown period for harvest, meaning the harvest function can only be called once every certain period. If the harvest is not executed within the allowed timeframe or if it is called too early, it can fail or revert, causing unclaimed rewards to be lost.

### How It Happens:

1. Cooldown Period: Some vaults implement a cooldown on the harvest function to avoid excessive gas fees from calling it too frequently. If the cooldown isn’t respected, the harvest function will revert, and rewards may remain unclaimed.

2. Early Harvest: If the harvest is called too early (before enough rewards have accrued), it can result in wasted gas and inefficient reward collection.