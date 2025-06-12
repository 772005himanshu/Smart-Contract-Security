## Multiple Token Addresses in Proxied Tokens

Tokens are often Deployed using proxy Pattern to enable upgrades without changing the token's address .A common vulnerability arises when proxied token have multiple addresses . Contract interacting with these tokens, particularly when handling transfer or safeguarding funds , may incorrectly assume the each token is associated with a single address.


One such vulnerability occurs in the context of `rescue `functions, which are designed to recover tokens mistakenly sent to the contract

If the `rescue` function assumes a single address per token, proxied tokens with multiple addresses could be incorrectly rescued or drained

### Understanding Vulnerabilities Arising From Token with Multiple Entry Point

Tokens implemented with proxy patterns, upgradeable mechanisms, or other special configurations may have more than one valid contract address.This leads to serious security flaws , including double withdrawal , token duplication and unexpected token interaction


#### How the Vulnerability Occurs
- Token A may be accessible via Address A1 and Address A2.

- Tokens implemented with proxy patterns, upgradeable mechanisms, or other special configurations may have more than one valid contract address. 
This can leads to :
1. Double withdrawals or double transfer 
2. Excessive Minting or rewards

- Exploiting Through Multiple Token Addresses

#### Mitigation 
By properly accounting for tokens that can be accessed through different addresses, developers can mitigate these risks and prevent attackers from exploiting this pattern.

Mitigation strategies such as token whitelists, balance snapshots, and token tracking help ensure that tokens are only processed once, regardless of how many addresses they have. Understanding and addressing this vulnerability pattern is critical for building secure and robust decentralized applications