## NFT JSON and XSS injections

In solidity based protocols , the `tokenURI` function is commonly used to return metadata for non-fungible tokens (NFTs).This metadata is often formatted as JSON containing important information such as the tokens name , description and image 

Improperly handling user input or failing to sanitize data can expose smart contract to JSON injection vulnerabilities, malicious actors manipulate the JSON data structure , potentially leading to security , identify as spoofing cross-site scripting (XSS) attacks or misleading data representation

### Vulnerability : JSON Injection in tokenURI Functions

Due to user input is not properly sanitized or encoded before it is incorporated into the token's metadata. this can allow an attacker to manipulate the JSON structure by injecting character (like `"` or `\`) that alter the intended structure of the JSON

NFT metadata setup . the `tokenURI` function may retunr something like

```javascript
{
  "name": "Bio #1",
  "description": "Test",
  "image": "data:image/svg+xml;base64,..."
}
```

If the user input for a field like "description" or "name" contains special characters and is not properly handled, the attacker can inject malicious JSON elements. For example:

Attcaker Input

```
Test", "name": "Bio #999999999
```

Resulting Manipulated JSON

```
{
  "name": "Bio #1",
  "description": "Test",
  "name": "Bio #999999999",
  "image": "data:image/svg+xml;base64,..."
}
```

#### Potential Attcak vector

1. Impersonation of other NFTs
2. Image Manipulation
3. Cross-Site Scripting(XSS) Attacks
4. Altered Metadata

#### Consequences 
- Confusion in Ownership 
- External Image Manipulation
- Cross Site Scripting

#### Mitigation Strategies
1. Santize User input : Always sanitize input provided by users before incorporating it into JSON metadata. This includes escaping special characters that could alter the structure of the JSON data, such as quotes (") or backslashes (\).

2. Properly Encode Data : Encode user data when embedding it in the tokenURI response to ensure that it does not introduce malformed JSON. 

3. Strict JSON Schema Validation : Implement validation to ensure that the resulting JSON conforms to a strict schema. By verifying that the output matches expected fields and data types, you can prevent unexpected values from being injected into the JSON.

4. Front-End Security: Ensure that any frontend applications interacting with the NFT metadata properly escape and sanitize user-generated content. This can help prevent XSS attacks from occurring when the data is rendered in a browser.


### Cross-Site Scripting (XSS) Vulnerability via SVG Construction in Smart Contracts: injecting Maliciuos Script

- This can be possible because of the SVG( Scalable Vector Graphics ) is an XML based file format used for rendering two-dimensional vector graphics, SVG allows Javascript to used in it 

- Stealing Session Cookies
- Malicious redirections
- Denial of Sevice (DOS)

SVGs, being XML-based, allow embedding of JavaScript using <script> tags or other inline event handlers (e.g., onload, onclick).

Example of Malicious XSS via SVG Contruction 
```solidity
contract MaliciousERC20 is ERC20 {
    constructor() ERC20("Malicious Token", "<svg><script>alert('XSS');</script></svg>") {}
}
```

Interaction With NFT Contract

```solidity
contract NFTTokenURIScaffold {
    function constructTokenURI(address asset) public view returns (string memory) {
        string memory svg = NFTSVG.constructSVG(ERC20(asset).symbol());
        return svg;
    }
}
```
#### Mitigation Strategies
1. Avoid Using User-Provided Input in SVGs: One of the simplest solutions is to avoid including user-provided input (like ERC20 symbols) in the generated SVGs. Instead, use hardcoded values or restrict the input sources to trusted entities.

2. Sanitize Inout Before Including it in SVG: Although Solidity lacks a native sanitization library, external systems (such as off-chain services) could sanitize the input before passing it to the smart contract

3. Validation Inputs Rigorously: Implement validation logic to ensure that symbols and other user-provided inputs do not contain malicious characters (e.g., <, >, or ")

4. Escape Dangerous Character in SVG Construction: 
If dynamic data must be included in SVGs, escape potentially dangerous characters before including them in the SVG construction.