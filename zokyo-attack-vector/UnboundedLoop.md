# Unbounded Loops

- Less seen in Today Implmentation (Security Increase as time goes)

- Unbounded gas Consumption

- Reverts within Loops

1. Gas Limit Vulnerability(Out of gas)

Example Of vulnerabilities
- A function within a contract iterates through an array or list.If this array becomes excessively large, iterating over it could consume gas past the block limit
- An attacker manipulates the contract, intentionally increasing the number of iterations a loop must perform, making certain contract functions unusable

Out of gas Vulnerability
- Denial of Service(DOS)
- Locked Funds 
- Manipulation by Attackers


2. Transaction Failures Within Loops

Understanding the Vulnerabilities 

When a transaction fails within a loop, Solidity automatically reverts all changes made by the transaction, ensuring atomicity. While this is a protective feature, in loops, it means that a failure at any iteration can undo all previous iterations, making the process inefficient and susceptible to disruption

Example : Failure Due to Blacklisted User:
Imagine a smart contract that processes withdrawals, iterating through a list of users. If any user is blacklisted (e.g., due to regulatory compliance such as anti-money laundering rules), a transfer to such a user will fail. This failure will cause the entire transaction to revert, affecting all users, not just the blacklisted one

```solidity
function processWithdrawals(address[] memory users, uint256[] memory amounts) public {
    for (uint i = 0; i < users.length; i++) {
        // Assuming USDC is the token being withdrawn
        USDC.safeTransfer(users[i], amounts[i]);
    }
}
```

Recommendation:

```solidity
function processWithdrawals(address[] memory users, uint256[]memory amounts) public {
    for (uint i = 0; i < users.length; i++) {
        // Check if user is not blacklisted before attempting transfer
        if (!isBlacklisted(users[i])) {
            // Attempt the transfer and handle potential failure
            try USDC.safeTransfer(users[i], amounts[i]) {
            } catch {
                // Handle the error, e.g., log it, and continue with the next iteration
            }
        }
    }
}
```