## abiDecoder v2

The ABI Decoder used to decode input data in ethereum smart Contract 

Input data encoding and decoding to manage interactions with tokens,pools and user balances

A critical Vulnerability pattern can emerge when a contract mistakenly assumes that the token address passed in the function is authentic , without adequently verifying the origin of that token address. ABI decoding vulnerabilities enable attackers to manipulate data inputs - such as token address perform unintended actions , unauthorizd withdrawals or balance mismanagement

### Attacker approach
Attackers can manipulate token addresses in calldata through `extra data` appended after the regular function arguments. 

When a contract does not explicitly validate the token address or rilies on asumptions about the input data structure , the attacker can trick the contract into interacting with a different token than intended

Using the abicoder v2, attackers can pass manipulated calldata that appends arbitrary token addresses. This may cause the contract to incorrectly map token balances or transfer amounts for the wrong token. 

#### Mitigations 
To prevent these type of vulnerabilities , developers must implement several key strategies to `safeguard against manipulated token interactions` 
- Explicit Token Validation
Always verify the authenticity of the token address before performing operations on it. Rather than relying solely on the token address passed by the user, contracts should compare it against a known whitelist of supported tokens.