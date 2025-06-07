## Weird ERC20 Tokens

The ERC20 standard itself is loosely defined and in practice function more as an interface declaration>even the few requirement it imposes are frequently disregarded by token developer,

Creating a smart contract that directly interact with ERC20 tokens can be quite challenging.

To mitigate these risks , we develop these things in contract implementation
1. Implement an allowlist at the contract level to restrict interactions to trusted tokens

2. Use dedicated wrapper contracts for token interactions at the boundaries of your system. This approach ensures that the ocre logic assumes consistent and reliable behaviour from the external tokens

### Weird Token List

1. Reentrant Calls
Some tokens , such as ERC777 tokens , allows reentrant calls during transfer .this means that during a token transfer ,an external can be made back into smart contract before original transfer completes , which can lead to vulnerabilities.This behaviuor has been exploited for instance , in imBTC Uniswap Pool , where reentrant calls led to drained funds

2. Missing Rturn Values
Certain tokens donot return a boolen('bool') value form ERC20 methods as expected Tokens like USDT,BNB and OMG are examples of this. SOme tokens such as BNB , many return a boolean for some methods but fail to do so for other , which caused stuck tokens in Uniswap V1. In extreme Cases such Tether Gold, tokens may declare `bool` return type but always retunr `false` even when the transfer are successful


Example Tokens:
- `MissingReturns.sol`: Does not return a boolean for any ERC20 operation.

- `ReturnsFalse.sol`: Always returns false for all ERC20 operations.


3. Fee On transfer 
Some tokens charge a fee on transfer , such as STA And PAXG. Even tokens that donot currently charges fees ,like USDT or USDC , may indroduce it Example - Balancer pool (STA Transfer fee($500k))

4. Balance Modifiaction outside of transfer(Rebaing / Airdrop)
Certain token modify user balance without initiating a transfer . these include rebasing token like Ampleforth and airdrop models like compunds governance tokens

Such arbitrary balance changes can break system that cache baalnces , like Uniswap V2 or Balancer . To prevent this , some system ensure  their pools aare updated atomatically during rebasing

5. Upgradable tokens
Tokens like USDC and USDT can be upgraded, allowing the contract owner to change the token's behavior at any time. This poses a risk to smart contracts that depend on specific behaviors from tokens. To mitigate this, developers can introduce logic that freezes interactions with an upgradable token if an upgrade is detected, as MakerDAO did with the TUSD adapter

6. Flash Mintable Tokens
Tokens like DAI support `flash minting` allowing tokens to be minted for duration of a transaction, provided they are returned by the end of the transaction. This is similar to flash loan but without requiring pre-existing tokens. Such tokens could mint upto `max uint256` tokens within a single transaction

7. Token with Blocklists 
Tokens like USDC and USDT implement admin-controlled blocklists that prevent transfers to or from certain addresses. This could be used to trap funds, for example, by adding a contract’s address to the blocklist. This risk may arise due to regulatory action or even malicious intent

8. Pausable Tokens
Tokens such as BNB and ZIL allow an admin to pause the token, preventing all transfers. This can expose users to risk if the admin is compromised or acts maliciously.

9.Approval Race Protections
Some tokens, like USDT and KNC, do not allow increasing an approved amount (`M > 0`) if an existing amount (`N > 0`) is already approved

10. Revert on Approval to Zero Address
Tokens like those in the OpenZeppelin framework will revert if an attempt is made to approve the zero address to spend tokens.

11. Revert on Zero Value Approvals
Some tokens, such as BNB, revert when approving a zero-value amount. Integrators must account for this by adding special cases in their contract logic.

12. Reverts on Zero Value Transfer
Some tokens revert when a zero-value transfer is initiated.


13. Multiple Token Address
Some proxied tokens, particularly those with multiple addresses, pose unique risks. For example, a `rescueFunds` function could allow a contract owner to steal all tokens in a pool if the logic assumes a single token address per contract.

14. Low Decimals
Tokens with fewer than 18 decimals, like USDC (which has 6), may introduce precision loss in calculations. Even more extreme, tokens like Gemini USD only have 2 decimals.

15. High Decimals
Some tokens, such as YAM-V2, have more than 18 decimals (YAM-V2 has 24). This can cause overflows and introduce risks of unexpected reverts in smart contracts.

16. TransferFrom with src == msg.sender
Some tokesn like DSToken , do not attempt to decrease the callers allowance if the caller is also the sender , making `transferFrom` behaves like `transfer`.  In contrast, other tokens, like those using OpenZeppelin, always decrease the allowance.

17. Non-string Metadata
Tokens like MKR encode their metadata (e.g., name, symbol) as `bytes32` rather than `string`. This can lead to issues when attempting to read metadata

18. Revert on transfer to the Zero Address
Tokens, such as those based on OpenZeppelin’s implementation, revert when transferring to `address(0)`. This could disrupt systems relying on `address(0)` for burning tokens.

19. Revert on Large Approvals & transfer
Tokens like UNI and COMP revert if the value passed to approve or transfer exceeds `uint96`. They also implement special logic for approve that sets allowances to `type(uint96).max` if the approval amount equals `uint256(-1)`.

20. Code Injections vis Token Name
Some malicious tokens include JavaScript code in their `name` attribute, allowing attackers to steal private keys from users interacting with the token via vulnerable frontends. This tactic has been used in the wild, notably to exploit EtherDelta users.

21. Unusual Permit Functions
Certain tokens, such as DAI, RAI, GLM, and others, have a `permit()` function that does not follow `EIP2612`. Tokens without a proper `permit()` implementation may not `revert`, leading to unexpected execution of subsequent lines of code. `Uniswap’s Permit2` is a more compatible alternative for handling such tokens.

22. Transfer of Less Than Amount
Tokens like cUSDCv3 contain special handling for transfers when `amount == type(uint256).max`, transferring only the user’s balance. Systems that transfer user-supplied amounts without verifying the actual transferred value may encounter issues with these tokens