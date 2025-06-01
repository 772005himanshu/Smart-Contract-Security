## Chain Re-org Vulnerability

- Chain reorganizations (re-orgs), which can impact the behavior of contracts and the outcomes of transactions.

- A chain re-org occurs when a blockchain network experiences a temporary fork, where different versions of the blockchain coexist until one is chosen as the canonical chain

- This process can potentially invalidated previously confirmed transaction , leading to serious security issue for smart contract

- Chain re-org vulnerabilities can affect a variety of decentralized applications (dApps) and protocols, especially those that rely on time-sensitive or transaction-dependent operations


### Chain Reorganization (Re-org) Vulnerability

A chain re-org refers to situation where the blockchain network temporaraily forks into two competing chains, after a certain number of blocks , one chain is discarded in favor of the other 

This process effectively "reorganizes" the blockchain and can lead to transactions that were previously considered confirmed being reverted or lost.

While re-orgs are generally rare and occur mostly due to network conditions, they can be exploited by malicious actors in certain scenarios, especially when dealing with smart contracts.

When re-rog vulnerability and how it can be leveraged by attackers to front-run , back-run or manipulate the behavior of smart contract, leading to potential thefft of funds, loss of contract ownership or double spending attacks

### How the chain Re-org Vulnerability Works

In a Blockchain once a transaction is included in a block and several blocks are added on top, it is typically considered considered finalized . However during a re-org , if the chain is restructured to favour a competing fork, a transaction that was part of the discarded chain can become invalid.This can create oportunities for attackers to exploit smart contracts by manipulating transaction ordering and altering expected outcomes

#### Attack Flow
1. Frontrunning the Deployment
when a smart contract or vault is deployed using a standard `create()` function, its deployment address is derived from the sender's address and nonce .
If a re-org occurs an attacker can `frontrun` the deployment by quickly deploying a malicious contract at same address as intended contract
This can lead to the attacker taking control of the intended contracts address

2. Backrunning the Deposit
After  frontrunning the deployment , an attacker can wait for the user to deposit assets into compromised vault. Once the deposit is confirmed on the blockchain, the attacker can then backrun the transaction, pulling funds from the vault by exploiting the ownership they gained during the re-org
This leaves the victim believing their deposit was successful , but the funds are actually controlled by the attacker

3. Taking Advantage of pending transaction
During the re-org, attackers monitor pending transactions in the mempool. They can submit competing transactions that are more profitable to miners (via higher gas fees), causing miners to include their transactions in the newly organized chain, while the legitimate transactions are discarded

### Common Re-org Vulnerability Exploit
1. ownership theft  
2. Front-running Contract Creation
3. Vault Fund Drain

### Mitigation Strategies For Reorg Vulnerabilities

Use `create2()` Instead of `create()`: The `create2()` function uses a `salt` to deterministically compute the contract address , making it less susceptible to address collision during re-org

## Chain Re-org Vulnerability in Governanace Voting mechanism
In Governance system, voting mechanism are often designed with increamental proposal id to Track and manage votes on different Proposals

Attack Flow:
1. Proposal Creation With Increamental IDs
2. Re-org Vulnerability in Proposal Voting:
- During a chain re-org, if the proposal creation transaction is included in the discarded chain, an attacker can take advantage of the reorganization to manipulate the proposal creation process.
- The attacker monitors the mempool for a pending proposal creation transaction. When a re-org occurs, the attacker submits their own proposal transaction during the re-org and takes the incremental proposal ID that was meant for the original proposal.
3. Front-Running Proposal Creation: proposal IDs are incremental, the attacker can "steal" the proposal ID by submitting their proposal first, causing the legitimate proposal to be assigned a different, later ID
4. Back-running Voting Process: Once the re-org resolves, the attacker has effectively front-run the proposal submission and back-run the voting process, gaining control over the governance process

