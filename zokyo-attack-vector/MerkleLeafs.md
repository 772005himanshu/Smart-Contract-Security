## Merkle Leafs

A typical Merkle tree consists of leaf nodes (which hold the original data) and intermediate nodes (which are combinations of two child nodes' hashes). The root node, or the Merkle root,

### Misuse of Merkle Leaf Nodes

The Vulnerability arises when the system does not strictly enforce the difference between leaf nodes(original data) and intermediate nodes( hashes of combined child nodes)

Attacker can submit intermediate nodes in place of leaf nodes . This allows them to pass invalid or incorrect data as if it were part of the set.

#### Visualizing the Vulnerability'

Image a Merkle tree Structure as follow

```
Root (hash of hash(Leaf A) and hash(Leaf B))
         /      \
    hash(Leaf A)  hash(Leaf B)
```

In this Structure Leaf A and Leaf B represent valid data , while the root node and intermediate nodes are hashes that represent the combination of child nodes

In a secure system, only Leaf A and Leaf B should be valid data points. However, if the system incorrectly treats an intermediate node, such as hash(Leaf A), as valid, an attacker can exploit this by submitting the intermediate node as if it were a valid piece of data.

#### Why this Vulnerability Happens
1. Lack of Validation Between Leaf and Intermediate Nodes
2. Misuse of Proof

#### Example: Token ID Validation via Merkle tree
The protocol allows users to specify several token IDs they are willing to accept. It builds a Merkle tree out of these token IDs, and the root hash is stored as the identifier for the item or trade criteria

An attacker submits an intermediate hash from the Merkle tree instead of a valid token ID. For instance, if the Merkle tree was generated from token IDs 1 and 2, the attacker might submit `hash(tokenID 1)` instead of `tokenID 1`. The system incorrectly validates the intermediate hash as a valid token, allowing the attacker to fulfill the transaction with an invalid token.

The protocol should accept only token ID 1 or 2. However, if the system does not differentiate between leaf nodes and intermediate nodes, an attacker can submit `hash(1)` as a valid token. This allows the attacker to bypass the intended validation and fulfill the trade without submitting a valid token ID.

#### Consequences
- Invalid Trade Execution
- Loss of Funds
- Protocol Vulnerability

### Why This Vulnerability is a Concern for ERC-721 Tokens
If a protocol uses a Merkle tree to validate token IDs, the system must carefully distinguish between valid token IDs (leaf nodes) and intermediate hashes. Failure to do so can allow attackers to fulfill trades with invalid tokens, leading to significant losses for users.

#### Mitigation Strategies
1. Hash Leaves Before Inserting into the Merkle tree(A recommended approach is to hash the token IDs themselves before placing them in the Merkle tree. This ensures that intermediate hashes are distinct from leaf node hashes, preventing attackers from submitting intermediate nodes as valid token IDs.)
2. Strict Leaf Node Validation(Implement rigorous validation checks to ensure that the submitted data corresponds to a valid leaf node in the Merkle tree. This can be done by hashing the submitted data (like a token ID) and comparing it against the Merkle proof to ensure it is a leaf node.)
3. Use of Type-Bytes to Differentiate Node Type(Another approach is to use type-bytes to differentiate between leaf nodes and intermediate nodes when constructing the Merkle tree. By including a type byte in the hashing process, you ensure that leaf nodes and intermediate nodes are not confused with each other.)