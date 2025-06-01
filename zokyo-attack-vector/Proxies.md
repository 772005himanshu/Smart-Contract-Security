## Proxies

Proxy Contracts in Ethereum Allow for the Seperation of logic and data , enabling upgradability and modularity in smart Contracts. 

If not properly managed , they can inadvertently introduce point of failure or misuse .This is often due to complex delegate calls nad adminitrative right mismanagement , Storage layout inconsisyencies or flawed upgrade mechanisms

### Storage: The Permanent Record Keeper

Every Smart Contract in Ethereum comes with its own set of long-term memeory This is what we refer to as `storage`.It where data needs to presist or remain unchanged 

- Persistent-Data: The Term 'Perssitent' simply means the data doen't go away 
- Structure & Costs: There is a gas cost of each slot for keeping data
- Differentiating Storage: Ethereum Also has short term data space like the stack and memory 

## Memory: The Brief Note Taker
`Memory` - Ethereum transient Data Holder, existing only for for a Contract short lived operation

- Life Cycle - When a contract is executed , memory as a scratcpad. It begins empty, captures interim data during execution, and then clears out once the operation concludes.

- Cost Dynamics - Uses Less gas Costs

- Role in Proxy Contract - But understanding it aids in appreciating the distinction between short-term and persistent data, a crucial concept when diving deeper into proxy contracts.

## Ethereum Callsa and Delegate Calls
1. External Calls
2. Internal Calls

Delegate Calls in Ethereum : A Deep Dive into Proxy Contract BackBone

- Due to capabilties of Delegate call lets Contract invoke a function from another contract while retaining its own state used for development of upgradable Smart Contract

- Even though the function's logic is borrowed from Contract B, any changes made during the function's execution will be applied to Contract A's storage. Contract B remains unaffected.Delegate call also cost gas , the cost might be offset by the advantages it provides , especially in system that value upgradability  and modularity.

Safety Check of using Delegate Call and Proxy
- Storage Layout between a proxy and its logic Contract doesn't match , it might leads to unintended storage modification , also known as storage collision

## Upgradability Patterns In Ethereum: Enhancing Smart Contract Over Time

1. Basic Structure :

- Proxy Contract(User Interaction , the state(data) and routes calls to logic contract)

- Logic Contract(it contains the logic of the Contract)

2. Harnessing Delegate Calls:
Interaction flow :
    User -> Proxy Contract -> Implementation Contract
Upgrading Contract Flow 
    User -> Proxy Contract -> logic (0x...1) (*)
                     |
                     --------> Logic(0x...2) (New Logic)     

Read More About Upgradable Contract(Openzeppelin upgrades)

- Storage Collision in Proxies : The Unstructured Storage Approach

- Flawed Storage Mapping

- Donot implement same function name/ Function Indentifier(4 Bytes based on its name and arity) in proxies nad logic contract(Transparent proxies and Potential Function Overlaps)

## Exploring the Landscape of Ethereum Proxies
1. Transparent Proxies 
At its core the transparent Proxy pattern is designed to circumvent the risk of the function signature clashes. It Differentiates between the admin(owner) of the proxy and Regular users to determine how calls are processed

Mechanism:
- Admin Calls -  If the caller is the admin of the proxy, it doesn't delegate any calls. It responds directly to messages it understands, especially those related to upgrades or admin-specific functionalities.

- User Calls - If the caller is anyone other than the admin, the proxy will delegate the call to the implementation contract, even if the function signature matches one of the proxy's functions

Benefits of Transparent Proxies
- Clear Admin / User Demarcations
- Safety Against Clashes

### Mechanics of function Clashes
Each function in a smart contract's public ABI is identified by a unique 4-byte signature. Given the limited size, different functions can inadvertently share the same signature. While Solidity warns of clashes within a contract, it doesn't do so across contracts. In a proxy setup, clashes between the proxy and its implementation can be problematic. The transparent proxy pattern adeptly handles this challenge.

## Consideration and Potenatial Trade-offs 
- Gas Costs - The additional logic beteen admina and user calls may result in higher gas costs


2. UUPS Proxies (Universal Upgradeable Proxy standard)(EIP-1822)
The key distinguishing feature of UUPS is that the upgrade logic is held in the implementation contract rather than the proxy itself. This differentiates it from other patterns like the Transparent Proxy.

- The magic occurs with the use of the EVM’s `DELEGATECALL` opcode. When a function is invoked on the proxy, it uses `DELEGATECALL` to delegate the function’s execution to its current logic contract, while retaining its own context, like `msg.sender`.

The Upgrade Mechanism: In UUPS, the proxy itself doesn't know how to upgrade. Instead, the logic of how and where to upgrade is stored in the implementation contract. When an upgrade is needed, the logic contract, which knows the proxy’s storage layout, is responsible for transferring the state and updating the proxy to point to the new logic contract. This offloads the upgrading complexity from the proxy to the logic contract.

- While the upgrade logic is offloaded, it still presents a potential vulnerability. Proper access controls, rigorous testing, and constraints (like time locks or multisigs) are essential

3. Beacon Proxies

A Beacon Proxy is an upgradeable smart contract pattern where the proxy contract, rather than pointing to a fixed logic contract, points to a beacon. This beacon, in turn, holds a reference to the logic contract. The distinguishing feature here is the introduction of this intermediary - the beacon - which determines which logic contract the proxy should delegate calls to.

- If multiple proxy contracts use the same beacon, upgrading the logic contract for all these proxies can be achieved by simply updating the beacon. This streamlines mass upgrades.

- The beacon can be designed to have advanced logic, like time-locked upgrades or multi-signature requirements, enhancing security

- As the lighthouse of logic contracts, the beacon becomes a critical security point. Rigorous testing, access controls, and safeguards are essential to prevent malicious upgrades.

4. Diamond Proxies(EIP-2535)

- In Ethereum, there's a maximum limit to how much bytecode a single contract can have, which is roughly 24 KB.
- As projects grow and become more complex, their smart contracts can easily surpass this limit, which makes it difficult to add new functionalities or update existing ones without splitting the logic across multiple contracts.
- The Diamond pattern introduces a more structured and extensible way to split contract logic across different contracts while still being viewed and interacted with as if they were a single contract

BreakDown of the Diamond Pattern:
1. Facets (Break smart Contract in parts)
2. Diamond Storage(Shares Storage is used for all facets)
3. Dispatcher(used for direct calling made to the Diamond to facet) by delegatecall 
4. Diamond Laupe(A set of standard functions to inspect facets and their functions)
