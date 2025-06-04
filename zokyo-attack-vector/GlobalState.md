## Forgetting to Update the Global State in Smart Contract

- Storage(global State on the Bloclchain) : When a variable is expected to persist across function calls 

- Memory (temporary data that only exists during execution of the function)

The issue arises when developers forget to transfer the changes made to the `local copy` back to the `global state.`

Example of the Mistake
```solidity
struct UserInfo {
    uint256 balance;
    uint256 rewards;
}

mapping(address => UserInfo) public userInfo;

function updateUserBalance(address user, uint256 newBalance) external {
    // The mistake: We load the userInfo into memeory ,but don't update the global state
    UserInfo memory userLocal = UserInfo[user];

    userLocal.balance = newBalance;// This only update the local copy in memory

    // Forget to write the updated data back to `userInfo[user]` in storage
}
```

Impact of Forgetting to update the Global State
1. Data Inconsistencies
2. Logic Errors : 
Many operations in DeFi (like token transfers, staking, and reward calculations) rely on accurate and up-to-date state variables. If the global state is not updated, these operations will be executed with incorrect or stale data, potentially leading to financial losses for users or incorrect reward distributions
3. Subtle bugs