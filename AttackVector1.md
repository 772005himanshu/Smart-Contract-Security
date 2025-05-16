## 0x Attack Vector 

### [M] `CidNFT`: Broken `tokenURI` function

**Description:** 
 
`tokenURI` function does not convert `uint256 _id` argument to a string before interpolating it in token URI:

```solidity
    /// @notice Get the token URI for the provided ID
    /// @param _id ID to retrieve the URI for
    /// @return tokenURI The URI of the queried token (path to a JSON file)
    function tokenURI(uint256 _id) public view override returns (string memory) {
        if (ownerOf[_id] == address(0))
            // According to ERC721, this revert for non-existing tokens is required
            revert TokenNotMinted(_id);
            /// NOTE - We donot Convert the _id to String
@>        return string(abi.encodePacked(baseURI, _id, ".json"));
    }

```
This means the raw bytes of the 32-byte ABI encoded integer `_id` will be interpolated into token URI, eg
`0x0000000000000000000000000000000000000000000000000000000000000001` for ID `#1`

**Impact:** `CidNFT` tokens will have invalid `tokenURI's` . OFFchain tools that read the `tokenURI` view may break or display malformed data

**Proof of Concept:**

Test

```Solidity
    function test_InvalidTokenURI() public {
        uint256 id1 = cidNFT.numMinted() + 1;
        uint256 id2 = cidNFT.numMinted() + 2;
        // mint id1
        cidNFT.mint(new bytes[](0));
        // mint id2
        cidNFT.mint(new bytes[](0));

        // These pass — the raw bytes '0000000000000000000000000000000000000000000000000000000000000001' are interpolated as _id.
        assertEq(string(bytes(hex"7462643a2f2f626173655f7572692f00000000000000000000000000000000000000000000000000000000000000012e6a736f6e")), cidNFT.tokenURI(id1));
        assertEq(string(bytes(hex"7462643a2f2f626173655f7572692f00000000000000000000000000000000000000000000000000000000000000022e6a736f6e")), cidNFT.tokenURI(id2));

        // These fail - the generated string on the right is not the expected string on the left. 
        assertEq("tbd://base_uri/1.json", cidNFT.tokenURI(id1));
        assertEq("tbd://base_uri/2.json", cidNFT.tokenURI(id2));
    }
```

**Recommended Mitigation:** 

Convert `_id` to string before calling `abi.encodePacked` . Latest Solmate includes a `LibString` helper library for this Purpose

```solidity
import {LibString} from "@solmate/utils/LibString.sol";

function tokenURI(uint256 _id) public view override returns (string memory) {
        if (ownerOf[_id] == address(0))
            // According to ERC721, this revert for non-existing tokens is required
            revert TokenNotMinted(_id);
            /// NOTE - We donot Convert the _id to String
        return string(abi.encodePacked(baseURI, LibString.toString(_id), ".json"));
    }

```

### [H] Bought/ Purchased Token Can be Sent to Attacker's Wallet Using 0x Adapter


**Description:** The lack of reciepien against the 0x Order within the 0x adapter (ZeroAdapter) allows the purchased/ouput tokens of the trade to be sent to the attacker's wallet

This is because the `from` of the `getExecutionData` is completely ignored, and the caller has the full flexibility of crafting an order that benefits the caller. This is not the case for Curve, Balancer V2, Uniswap V2 & V3 adaptors, as the caller is not able to specify the recipient of the output tokens to another address. Attackers can craft a 0x order that redirects the assets to their wallet, leading to loss of assets for the vaults and their users. To prevent this, it is recommended to implement validation against the submitted 0x trade order to ensure that the recipient of the bought tokens is set to the vault when using the 0x DEX

**Impact:** 
Attackers can craft a 0x order that redirects the assets to their wallet, leading to loss of assets for the vaults and their users.

**Proof of Concept:**

**Recommended Mitigation:** - Could Not Understand What is recommendation ??

### [H] First Depositor can break minting of shares

**Description:** The attack vector and impact is the same as [TOB-YEARN-003](https://github.com/yearn/yearn-security/blob/master/audits/20210719_ToB_yearn_vaultsv2/ToB_-_Yearn_Vault_v_2_Smart_Contracts_Audit_Report.pdf), where users may not receive shares in exchange for their deposits if the total asset amount has been manipulated through a large “donation”.


**Impact:** 

**Proof of Concept:**
* Attacker deposits 2 wei (so that it is greater than min fee) to mint 1 share

* Attacker transfers exorbitant amount to `_strategyController` to greatly inflate the share’s price. Note that the `_strategyController` deposits its entire balance to the strategy when its `deposit()` function is called.

* Subsequent depositors instead have to deposit an equivalent sum to avoid minting 0 shares. Otherwise, their deposits accrue to the attacker who holds the only share.


**Recommended Mitigation:** 
* [Uniswap V2 solved this problem by sending the first 1000 LP tokens to the zero address](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L124). The same can be done in this case i.e. when `totalSupply() == 0`, send the first min liquidity LP tokens to the zero address to enable share dilution.

* Ensure the number of shares to be minted is non-zero: `require(_shares != 0, "zero shares minted");`

* Create a periphery contract that contains a wrapper function that atomically calls `initialize()` and `deposit()`

* Call `deposit()` once in `initialize()` to achieve the same effect as the suggestion above.
