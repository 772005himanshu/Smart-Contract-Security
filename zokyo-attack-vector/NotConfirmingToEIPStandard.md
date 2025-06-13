## Not Conforming To EIP standards

Ethereum Improvement Proposals (EIPs) such as  EIP-2981, EIP-165, EIP-2612, and others,

When developers fail to adhere to these established standards, various risks can be introduced into decentralized applications. Non-conformance can lead to bugs, incompatibilities with external systems, and security vulnerabilities that can undermine the integrity of the contract

The importance of standards like EIP-155 for replay attack protection and EIP-1967 for storage management

## Understanding EIP - 2981 NFT Royalty Standard
The EIP-2981 defines a standard way for NFT contracts to communicate royalty information to marketplaces and platforms. 

This allows for consistent enforcement of royalty payments whenever an NFT is sold on secondary markets. Implementing this standard ensures that the creator automatically receives a portion of the sales proceeds.

The Key function introduced by EIP-2981 is:
```solidity
function royaltyInfo(uint256 tokenId, uint256 salePrice) external view returns (address receiver, uint256 royaltyAmount);
```

This function returns the royalty recipient’s address and the royalty amount based on the sale price of the NFT.

In addition, `EIP-165` must also be implemented to enable marketplace contracts to detect whether an NFT supports the `EIP-2981` standard. Without implementing `EIP-165`, marketplaces may be unable to verify that an NFT includes royalties, leading to missed payments for the creator.

### Vulnerability: Missing EIP-2981 Implementation
If the NFT contract does not implement EIP-2981 properly or fails to include the necessary interface detection through EIP-165, the NFT will not signal its royalty information to marketplaces.

As a result, platforms that support EIP-2981-compliant NFTs may assume that royalties are not applicable, resulting in no royalties being paid to the creator when an NFT is resold.

### Impact 
1. Loss of Creator Royalities
2. MarketPlace Incompatability

### Implementing EIP-2981 Properly 
Step 1: Implement EIP-2981 Royality Logic(to NFT Contract)
Step 2: Implement EIP-165 for Interfcae Detection(to MarketPlace)


## Improper Implementation of EIP-2612 Permit Standard
`EIP-2612` introduces a significant improvement to ERC-20 tokens by adding a `Permit` function that allows `gasless` approvals. With this standard, users can `approve token` transfers through signed messages (`off-chain`), rather than having to submit `on-chain` approval transactions. This `reduces` the number of `transactions` required and helps users save on gas fees


### Understanding EIP-2612: Permit Function ans Gasless Approvals

`EIP-2612` allows for the Permit function, which enables users to `approve ERC-20` token transfers by `signing` a message that can be submitted by `another party`. This eliminates the need for an `on-chain` approval transaction, reducing the number of transactions required to approve and transfer tokens.

Key feature of EIP-2616 in the following function

```solidity
function permit (
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s,
) external;
```

Additionally, the standard requires the implementation of a DOMAIN_SEPARATOR() function, which ensures that signed messages can be verified as belonging to the correct contract instance. This is essential to prevent replay attacks across different chains or contract instances.

```solidity
function DOMAIN_SEPARATOR() external view returns (bytes32);
```

### Vulnerbility: Improper EIP-2612 Implementation
The Contract does not properly implemented the `DOMAIN_SEPARATOR` function or deviated from the required standard

1. Imcompatibility with services and Contract: Other protocols and services that strictly adhere to the EIP-2612 standard rely on the `DOMAIN_SEPARATOR()` function to verify signed messages. Without this function, the token contract becomes incompatible with services that use this mechanism, limiting the token's utility.

2. Lack of Gasless Approvals: Tools and exchanges that enable gasless approvals may not recognize the token as compatible with EIP-2612, preventing users from utilizing this functionality. As a result, users are forced to pay additional gas fees for approvals, which defeats the purpose of EIP-2612's optimization

3. Replay Attack Risks: Not Properly implemented Domain Separtor

## Vulnerabilities of missing EIP-155 Replay Attack Protections
The `EIP-155` standard was introduced to address this issue by including `chain ID` as part of `transaction signatures`, preventing signed transactions from being `replayed` on different chains.

### Understanding EIP-155: Protection Against Replay Attacks
`EIP-155` ensures that signed messages are valid only on the chain where they were intended to be used. The `chain ID `uniquely identifies the network where a transaction is broadcast, and transactions signed under `EIP-155` contain the `chain ID` as part of the `signature`.

### Vulnerability: Missing EIP-155 Replay Protection

In the example provided, functions such as publishProject, addMember, and escrow use `ecrecover` to verify `signed messages` but `do not` include `chain ID` in the `message`, making the protocol vulnerable to replay attacks if deployed across multiple networks.

### Impact of Missing Replay Attack Protection
1. Cross-Network Exploits
2. Hard Fork Vulnerabilities
3. Security Breakdown in Multichain Deployments

## Vulnerabilities Due to Missing EIP-1967 in Proxy Contract
EIP-1967 introduces a standard for handling storage slots in upgradeable proxy contracts, ensuring that important variables like the proxy’s implementation address and admin do not overlap with storage slots in the implementation contract

Failure to adopt EIP-1967 can lead to storage collisions, where storage variables in the proxy and implementation contracts are overwritten, causing unintended behavior or security vulnerabilities

### Understanding EIP-1967: Storage Collision Prevention in proxy Contract

EIP-1967 was introduced to solve this problem by defining fixed storage slots for key proxy variables, such as the implementation address and admin. This ensures that these variables do not conflict with storage in the implementation contract, preventing accidental overwriting

EIP-1967 assigns specific storage slots based on hashed values. For example, the storage slot for the proxy’s implementation address is defined as:

```solidity
bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

### Vulnerability: Missing EIP-1967 Lead to Storage Collisions
For example, if the proxy contract inherits from Ownable, the `_owner` variable may occupy the `first storage slot`. If the `implementation` contract also has variables that use the` same storage slots`, this can lead to storage `collisions`, where important data is overwritten or corrupted.

```solidity
contract CoreProxy is Ownable {
    address private immutable _implement;

    // Proxy logic here...
}
```

## Proof of Concept: Storage Collision Example
```solidity
contract CoreProxy is Ownable {
    address private immutable _implement;

    function _delegate(address implementation) internal {
        // Delegate call logic
    }
}
```

If the implementation contract uses the same storage slot (e.g., slot 0 for _owner), the address of the implementation contract could be overwritten by a variable in the implementation logic, leading to malfunction or loss of control.

### Use EIP-1967 to Avoid Storage Collisions


## Vulnerability of Design Preventing EIP-165 Extensibility
EIP-165 provides a standard method for contracts to signal support for interfaces, which allows for easy detection of a contract’s capabilities by external protocols

If a wallet’s `fallback` mechanism is not properly designed, it can `block` the extensibility of `EIP-165`, making it `impossible `to extend or add support for `new interfaces`.


`EIP-165` introduces the `supportsInterface` function, which allows a contract to declare the interfaces it supports, such as `ERC721` or `ERC1155`, making it easier for other contracts to detect what functionality a given contract has

```solidity
function supportsInterface(bytes4 interfaceID) external view returns (bool);

```

Smart contracts often use `fallback` handlers to extend functionality. These fallback mechanisms allow contracts to `delegate calls` that are not explicitly defined in the main contract to an `external` fallback handler. This pattern allows for `dynamic functionality`, enabling wallets to be flexible and extensible.

### Vulnerability: Wallet Design Prevents EIP-165 Extensibility
In a typical wallet design that uses fallback handlers, any function defined in the main wallet contract, such as `supportsInterface`, will always be handled by the main contract and not passed to the fallback

1. New Interface Cannot Be Added: Once the wallet is deployed, it will only support the interfaces defined at that point. Any future interfaces or standards that require EIP-165 detection will be unsupported unless the wallet is redeployed

2. Inflexibility for Upgrades:The wallet is locked into the functionality defined at deployment. Any attempt to extend or upgrade support for new interfaces through the fallback will fail because the calls will be caught by the main contract.

### Impact:
- limited functionality
- Incompatibility with future Standards
- Missed Extensibility

## The Dangers of Not Properly Implementing ERC-4626 in Yield Vaults
`ERC-4626` is a tokenized vault standard designed to optimize and standardize `yield-bearing vaults`. It provides a common framework for managing assets `deposited` into a vault and specifies how `shares and assets` are handled.

ERC-4626 can lead to significant vulnerabilities, such as incorrect accounting of assets, over-withdrawing of funds, and violations of the protocol,

### Understanding ERC-4626: The Tokenized Vault Standard
Key functions defined by ERC-4626 include:

- `deposit(uint256 assets, address receiver)`: Allows users to deposit assets and receive shares.

- `mint(uint256 shares, address receiver)`: Allows users to mint a specific number of shares by depositing the corresponding amount of assets.

- `withdraw(uint256 assets, address receiver, address owner)`: Allows users to redeem their shares for a specific amount of assets.

- `maxWithdraw(address owner)`: Returns the maximum amount of assets that can be withdrawn.

- `convertToShares(uint256 assets)`: Converts assets into their equivalent number of shares.

- `convertToAssets(uint256 shares)`: Converts shares into the equivalent number of assets.

### Vulnerability: Incorrect Implmentation of ERC-4626
mproper implementation of the `maxWithdraw` and `maxRedeem` functions in the `ERC-4626` vault standard can lead to serious vulnerabilities

For example, if the vault's convertToAssets() function is used incorrectly, it may return too many assets, leading to a violation of the ERC-4626 specification and the potential for users to withdraw more than they should.

### Impact
- Over-Withdrawal
- Violation of ERC-4626 Specifictions(maxWithdraw() and maxRedeem() )
- Inaccurate Accounting


### Proof of Concept: Vulnerable maxWithdraw Implementation
Consider the following example, where `_maxYieldVaultWithdraw()` uses `convertToAssets()` to calculate the maximum amount of assets that can be withdrawn from the vault:

```solidity
function _maxYieldVaultWithdraw() internal view returns (uint256) {
    return yieldVault.convertToAssets(yieldVault.maxRedeem(address(this)));
}
```
In this case, `convertToAssets()` is used to calculate the asset amount, but since `convertToAssets()` is an approximation, this can result in returning too many assets. Consequently, the `maxWithdraw()` function might allow users to withdraw more than they should:

```solidity
function maxWithdraw(address owner) public view returns (uint256) {
    return _maxYieldVaultWithdraw();
}
```


### Mitigating the Vulnerability: Correctly Implementing ERC-4626

1. Replace `convertToAssets()` with `preciewRedeem()`
2. Ensure Proper Validation in `deposit` and `mint`

## EIP-712 Implementation and Replay Attacks

