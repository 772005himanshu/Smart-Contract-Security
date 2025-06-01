## isContract Vulnerability

To ensure the integrity and robustness of a contract's functionality. One such notable aspect is the distinction between Externally Owned Accounts(EOA) and Contract Accounts.

Developers sometimes use the `isContract()` function to discern between contract accounts and EOAs to enforce specific restriction or design patterns in the contracts

### Understanding the `isContract()` vulnerability

- What is the `isContract()` Function ?
The `isContract()` function is utilized within Solidity to ascertain whether a specific address to a contract or an exteranlly owned account (EOA) .

It uses the `extcodesize` opcode to check if the address has associated contract code

```solidity
function isContract() private view returns(bool) {
    uint32 size;
    assembly {
        size := extcodesize(_addr)
    }
    return (size > 0);
}
```

### The Vulnerability Explained

- Bypassing During Contract Creation
The Vulnerability emerges during the contract creation phase . When a contract is in its construction process ,the `extcodesize` for that contract address is zero 

If the contract interact with another contract in its constructor, the `isContract()` function will inaccurately return false, allowing it to bypass any restriction that should apply to contract accounts

Due to this vulnerability, a malicious actor could instantiate a contract that interacts with the target contract within its constructor, effectively bypassing the `isContract()`restriction and executing functions intended only for EOAs or specifically whitelisted addresses.

### Example

```solidity
modifier onlyEOA() {
    require(!isContract(msg.sender), "Contracts not allowed");
    _;
}

function sensitiveFunction() public onlyEOA {

}
```

A malicious actor exploit the vulnerability

```solidity
contract MaliciousContract {
    TargetContract target;

    constructor(TargetContract _target) {
        target = _target;
        target.sensitiveFunction();
    }
}
```