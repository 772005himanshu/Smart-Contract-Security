## Handling Edge Cases of Banned Addresses in DeFi

Handling these banned addresses becomes crucial for ensuring that protocols operate securely and fairly. If edge cases related to banned addresses are not handled properly, they can create vulnerabilities, such as the ability for malicious actors to bypass security checks or the inability to recover funds from transactions involving banned addresses.

### Understanding Banned Address Edge Cases

1. Cross-Chain Bridges: In Cross Chain System a transaction might be place in a `retriable` state when an operation filas (eg due to insufficient gas). If the user becomes banned during this period, the system might still allow the transaction to retired , resulting in unexpected behaviour

2. Pending Transcation:  A user might submit a transaction before being banned, and if that transaction is delayed or retried later (for example, after a protocol upgrade), the protocol might still process the transaction even though the user is now banned.

3. BlackListing in ERC20 or Governance Contract: Some protocols have blacklists or ban lists for addresses that aren't allowed to transfer tokens or participate in governance. 

### Example 1: Cross Chain Bridges
In a cross chain Bridges , uers can transfer assets between two blockchain (e.g Ethereum and binance Smart Chain). If the transaction fails(e.g Due to a gas limit or network issue), it might enter a `retriable` state where the user or the protocol can retry the transaction later . where the user or the protocol can retry the transaction later.