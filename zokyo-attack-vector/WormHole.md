## WormHole

Wormhole is the cross -chain interoperability prorocol designed to facilate the seamless transfer of assets , data ,and message between different blockchain networks

### Proposal Execution failure Due to Guardian Set Change

These guardian sets are responsible for verfying the authenticity of cross-chain messages (VAAs) and ensuring the correct functioning of the Protocol

A vulnerability arises when a guardian set is updated between the time a proposal is queued and when it is executed, leading to failed execution due to outdated signature validation

#### How the Vulnerability Occurs
1. Proposal Queuing
2. Guardian Set Change During Delay
3. Failed Proposal Execution( When the proposalDelay passes and the protocol attempts to execute the proposal, the system performs a second verification against the Wormhole bridge contract it fails to exuction)
4. Delayed governance(The Proposal remains unexecuted and the Governance process is delayed)