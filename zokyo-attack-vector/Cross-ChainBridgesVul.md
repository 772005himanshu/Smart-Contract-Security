## Cross Chain Bridges Vulnerabilities

Cross Chain Bridges are often used to facilate the movement of tokens between different layers , such as Layer (L1) and Layer (L2) blockchain. In some Cases these bridges handle complex tokens like those compliant with the ERC-777 standard (introduce hooks) for token transfer --> to Reentrancy Vulnerability

### Vulnerability Explanation
In the Token Bridge Contract , users can transfer tokens between L1 and L2 by calling function like `bridgeToken`. This function usually calculates the number of tokens to bridge by measuring the difference between the contract `pre-transfer` and `post-transfer` balances. This calculation is essential to support tokens with `unusual transfer logic` such as those that deduct a fee on each transfer

Attacke Scenario:

1. Setup
The attacker registers a contract as an `ERC777TokensSender` via the `ERC-1820 registry`. This contract will receive the `tokensToSend` callback whenever an `ERC-777` transfer is initiated by the TokenBridge contract.

2. Initial Bridges Call
The attacker calls `bridgeToken` with 500 tokens. At this point:

- The contract stores its initial balance (let’s call it balanceBefore), which is `0`.

- The contract then initiates a `safeTransferFrom` call to transfer 500 tokens from the attacker’s address to the bridge contract.

- During this transfer, the `tokensToSend` callback in the attacker’s contract is triggered

3. Reentrancy Attack
- Inside the `tokensToSend` callback, the attacker `re-enters` the `bridgeToken` function by calling it again, this time transferring an additional 500 tokens. Since the contract logic hasn’t updated its state yet:

- `balanceBefore` in the second bridgeToken call is still 0, and the safeTransferFrom function is executed again.

- After the second transfer, `balanceAfter` becomes 500.

4. Original Call Completes
After the second call completes, the original `bridgeToken` call continues. The `balanceAfter` in this original call is now 1000 (due to both the original and re-entrant transfers). The contract incorrectly credits the attacker with 1500 tokens on L2, despite only sending `1000 tokens`.

#### Example Code: Vulnerable `bridgeToken` function

```solidity
function bridgeToken(address token, uint256 amount) external {
    uint256 balanceBefore = IERC20(token).balanceOf(address(this));

    // Transfer tokens from msg.sender to the bridge
    IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

    uint256 balanceAfter = IERC20(token).balanceOf(address(this));
    uint256 bridgedAmount = balanceAfter - balanceBefore;

    // Logic to credit bridgedAmount on L2...
}
```

Impact:

- Inflation of Token Balance
- Depletion of Liquidity(Collapse of the Bridge)

### Vulnerability: Withdrawals can be Locked Forever If Recipient Is a Contract

In Some Ethereum smart Contracts , withdrawals of assets like ETH or ERC20 are handles through a function teh tokens abck to the users after a cooldown period. 

One Common method for sending ETH is the low-level `transfer()` function , using the `transfer()` can inadvertently lock funds forver when the recipient is a contract, particularly if the recipient contract has a receive() and fallback() function requires more gas than what is forwarded by `transfer()`(2300 gas) , it is good for EoA but not enough to cover more complex logic that might exist in contract account(e.g., a multisig wallet or a contract implementing more complex functionality).

In scenarios where a user initiates a withdrawal from a contract account that requires more than `2300` gas to execute the `receive()` or `fallback()` function, the `transfer()` call will fail, causing the transaction to `revert`.

```solidity
payable(msg.sender).transfer(_withdrawRequest.amountToRedeem);
```

- Asset get permanently locked in the contract , since request is tied to multisig wallet , smart Contract Wallet or defi Protocol

Many multisig wallets, smart contract wallets, or DeFi protocols have a receive() or fallback() function that may require significantly more gas to perform necessary actions, such as emitting events, updating internal state, or interacting with other contracts. When transfer() is used, these contracts cannot process the transaction, causing it to fail.

### The Dangers of Not using SafeERC20 for Token Transfers

#### why SafeERC20 is Critical for Secure Token Transfer
The standard ERC-20 functions do not perform critical safety checks that could prevent unexpected behaviors or reentrancy attacks. Specifically, token transfers using functions like `transfer()` or `transferFrom()` can silently fail or interact with contracts in unintended ways if proper checks are not implemented.

1. Function Call Verification:
SafeERC20 uses OpenZeppelin's `functionCall()` method to verify that the target address contains valid contract code before attempting a transfer

2. Return Value Check
Standard ERC-20 functions may not return a boolean value indicating whether the transfer was successful. SafeERC20 checks the return value and ensures that the transfer succeeded.

3. Prevents Reentrancy Attacks
SafeERC20 helps protect contracts from certain types of reentrancy attacks by performing proper validation before and after the token transfer.

### Without SafeERC20 
- Tokens being transferred to invalid addresses (eg. zero address)
- Silent Failure , where the contract believes the transfer was successful but no tokens were actually moved
- Opening the door to reentrancy vulnerabilities in more complex contract , such as bridges


### Not Using the SafeERC20 in token Bridges(QBridge Hack using fake ETH or address zero with safeERC20 library)
- Invalid Target Address
- Reentrancy Attacks


### Uninitialized Variable Vulnerability in Uphradeable Smart Contract

An uninitialized variable refers to a variable that hasn't been explicitly set during contract deployment or upgrade, leaving it with a default value (typically `0`or the `zero` address). This default value can lead to unintended behavior, particularly in smart contracts where certain variables are critical for maintaining security, such as access control, voting mechanisms, or cross-chain verifications. In upgradeable contracts, the risk is amplified because developers may assume variables carry over automatically from one contract version to the next, which is not always the case.

Real life Vulnerabilities:
1. Ronin Network Hack($12 million)(_totalOperatorWeight == 0 (unitialize) MEV bot to exploit)
2. Nomad Bridge Hack ($190 million)(trusted root == zero address (call the contract process function without providing valid Proofs ))

### Unsafe External Calls and Their Vulnerabilities

An external call occurs when a smart contract interacts with another contract or an external address. External calls are fundamental to blockchain protocols, but they come with risks


Poly Network hack
```Solidity
function verifyHeaderAndExecuteTx(bytes memory proof, bytes memory rawHeader, bytes memory headerProof, bytes memory curRawHeader,bytes memory headerSig) whenNotPaused public returns (bool){
        ECCUtils.Header memory header = ECCUtils.deserializeHeader(rawHeader);
        // Load ehereum cross chain data contract
        IEthCrossChainData eccd = IEthCrossChainData(EthCrossChainDataAddress);
        
        // Get stored consensus public key bytes of current poly chain epoch and deserialize Poly chain consensus public key bytes to address[]
        address[] memory polyChainBKs = ECCUtils.deserializeKeepers(eccd.getCurEpochConPubKeyBytes());

        uint256 curEpochStartHeight = eccd.getCurEpochStartHeight();

        uint n = polyChainBKs.length;
        if (header.height >= curEpochStartHeight) {
            // It's enough to verify rawHeader signature
            require(ECCUtils.verifySig(rawHeader, headerSig, polyChainBKs, n - ( n - 1) / 3), "Verify poly chain header signature failed!");
        } else {
            // We need to verify the signature of curHeader 
            require(ECCUtils.verifySig(curRawHeader, headerSig, polyChainBKs, n - ( n - 1) / 3), "Verify poly chain current epoch header signature failed!");

            // Then use curHeader.StateRoot and headerProof to verify rawHeader.CrossStateRoot
            ECCUtils.Header memory curHeader = ECCUtils.deserializeHeader(curRawHeader);
            bytes memory proveValue = ECCUtils.merkleProve(headerProof, curHeader.blockRoot);
            require(ECCUtils.getHeaderHash(rawHeader) == Utils.bytesToBytes32(proveValue), "verify header proof failed!");
        }
        
        // Through rawHeader.CrossStatesRoot, the toMerkleValue or cross chain msg can be verified and parsed from proof
        bytes memory toMerkleValueBs = ECCUtils.merkleProve(proof, header.crossStatesRoot);
        
        // Parse the toMerkleValue struct and make sure the tx has not been processed, then mark this tx as processed
        ECCUtils.ToMerkleValue memory toMerkleValue = ECCUtils.deserializeMerkleValue(toMerkleValueBs);
        require(!eccd.checkIfFromChainTxExist(toMerkleValue.fromChainID, Utils.bytesToBytes32(toMerkleValue.txHash)), "the transaction has been executed!");
        require(eccd.markFromChainTxExist(toMerkleValue.fromChainID, Utils.bytesToBytes32(toMerkleValue.txHash)), "Save crosschain tx exist failed!");
        
        // Ethereum ChainId is 2, we need to check the transaction is for Ethereum network
        require(toMerkleValue.makeTxParam.toChainId == uint64(2), "This Tx is not aiming at Ethereum network!");
        
        // Obtain the targeting contract, so that Ethereum cross chain manager contract can trigger the executation of cross chain tx on Ethereum side
        address toContract = Utils.bytesToAddress(toMerkleValue.makeTxParam.toContract);
        
        //TODO: check this part to make sure we commit the next line when doing local net UT test
@>        require(_executeCrossChainTx(toContract, toMerkleValue.makeTxParam.method, toMerkleValue.makeTxParam.args, toMerkleValue.makeTxParam.fromContract, toMerkleValue.fromChainID), "Execute CrossChain Tx failed!");  // Here Vulnerable External Call

        // Fire the cross chain event denoting the executation of cross chain tx is successful,
        // and this tx is coming from other public chains to current Ethereum network
        emit VerifyHeaderAndExecuteTxEvent(toMerkleValue.fromChainID, toMerkleValue.makeTxParam.toContract, toMerkleValue.txHash, toMerkleValue.makeTxParam.txHash);

        return true;
    }
    
    /* @notice                  Dynamically invoke the targeting contract, and trigger executation of cross chain tx on Ethereum side
    *  @param _toContract       The targeting contract that will be invoked by the Ethereum Cross Chain Manager contract
    *  @param _method           At which method will be invoked within the targeting contract
    *  @param _args             The parameter that will be passed into the targeting contract
    *  @param _fromContractAddr From chain smart contract address
    *  @param _fromChainId      Indicate from which chain current cross chain tx comes 
    *  @return                  true or false
    */
    function _executeCrossChainTx(address _toContract, bytes memory _method, bytes memory _args, bytes memory _fromContractAddr, uint64 _fromChainId) internal returns (bool){
        // Ensure the targeting contract gonna be invoked is indeed a contract rather than a normal account address
@>        require(Utils.isContract(_toContract), "The passed in address is not a contract!"); // check is contract but with this we call they forgot to prevent users from calling a very important target... the EthCrossChainData contract
        bytes memory returnData;
        bool success;
        
        // The returnData will be bytes32, the last byte must be 01;
        (success, returnData) = _toContract.call(abi.encodePacked(bytes4(keccak256(abi.encodePacked(_method, "(bytes,bytes,uint64)"))), abi.encode(_args, _fromContractAddr, _fromChainId)));
        
        // Ensure the executation is successful
        require(success == true, "EthCrossChain call business contract failed");
        
        // Ensure the returned value is true
        require(returnData.length != 0, "No return value from business contract!");
        (bool res,) = ZeroCopySource.NextBool(returnData, 31);
        require(res == true, "EthCrossChain call business contract return is not true");
        
        return true;
    }

```

The first four bytes of transaction input data is called the "signature hash" or "sighash" for short. It's a short piece of information that tells a Solidity contract what you're trying to do.

The sighash of a function is calculated by taking the first four bytes of the hash of "<function name>(<function input types>)". For example, the sighash of the ERC20 transfer function is the first four bytes of the hash of "transfer(address,uint256)".

Poly's contract was willing to call any contract. However, it would only call the contract function that corresponded to the following sighash:


Well... here's the actual sighash of the target function:
```
http://ethers.utils.id ('putCurEpochConPubKeyBytes(bytes)').slice(0, 10) '0x41973cd9'
```
And the sighash that the attacker crafted...
```
http://ethers.utils.id ('f1121318093(bytes,bytes,uint64)').slice(0, 10) '0x41973cd9'
```
Fantastic. No private key compromise required! Just craft the right data and boom... the contract will just hack itself!



#### Why external Call are Dangerous 

1. Loss of Control
2. Reentrancy Attack
3. Custom Calldata Exploit

#### Best Practise to Mitigate Unsafe External Call Vulnerabilities
1. Validate External Contract Address
2. Use safeERRC20 For token transfer
3. Limit user Provided Data(call data)(Be cautious when accepting user-controlled calldata or function selectors.)
4. Use Reentrancy Guards
5. Check Return Values(Always check the return values of external calls to ensure that the call was successful and executed as expected. If the call fails or returns false, revert the transaction.)


### Signature Replay Attack in Cross Chain Protocols

`Digital signatures` are used to verify the authenticity and integrity of a transaction. Each signature is created by a user using their private key and is intended to authorize a specific transaction on a particular blockchain. However, in cross-chain environments, a signature that is valid on one chain may also be valid on another if the system does not differentiate between chains.(chain id is used as differentiate b/w chains)

In a signature replay attack, an attacker reuses a valid signature intended for one blockchain to perform the same transaction on another blockchain.

#### Key Issues Leading to Signature Replay Attacks
1. Lack of Chain Context
The root cause of this vulnerability is that the bridge protocol failed to check whether a signature was already used on a different blockchain. Signatures are often valid across multiple EVM-compatible chains (Ethereum, BSC, Polygon, etc.), making it critical to validate the chain ID or network context when verifying signatures.

2. Reuse of Signature
If a signature generated for one blockchain is valid on another, attackers can reuse it to execute duplicate transactions across different networks, resulting in the transfer of assets multiple times.

#### Mitigation 
Use ChainID Verification : The ChainID is the uniquie identifier for each blockchain, Always include the chain ID as part of the transaction data when generating a signature. This ensures that signatures generated on one chain are not valid on another