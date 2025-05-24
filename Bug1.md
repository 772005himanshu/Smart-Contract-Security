## Loss of funds for traders due to accounting error in royalty calculations

## Vulnerability Details
### Impact
The `PrivatePool.buy` and `PrivatePool.sell` function intended to distribute royalty amount whenever NFTs are traded . The implementation of buy and sell liks like this 

```solidity
    function buy(uint256[] calldata tokenIds, uint256[] calldata tokenWeights, MerkleMultiProof calldata proof)
        public
        payable
        returns (uint256 netInputAmount, uint256 feeAmount, uint256 protocolFeeAmount)
    {
        // ...

        // calculate the sale price (assume it's the same for each NFT even if weights differ)
        uint256 salePrice = (netInputAmount - feeAmount - protocolFeeAmount) / tokenIds.length;
        uint256 royaltyFeeAmount = 0;
        for (uint256 i = 0; i < tokenIds.length; i++) {
            // transfer the NFT to the caller
            ERC721(nft).safeTransferFrom(address(this), msg.sender, tokenIds[i]);

            if (payRoyalties) {
                // get the royalty fee for the NFT
                (uint256 royaltyFee,) = _getRoyalty(tokenIds[i], salePrice);

                // @audit - here is vulnerability - first check the royalty Fee and recipient != address(0) then add royaltyFee to royaltyFeeAmount

                // add the royalty fee to the total royalty fee amount
                royaltyFeeAmount += royaltyFee;
            }
        }

        // add the royalty fee amount to the net input aount
        netInputAmount += royaltyFeeAmount;

        // ...

        if (payRoyalties) {
            for (uint256 i = 0; i < tokenIds.length; i++) {
                // get the royalty fee for the NFT
                (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);

                // transfer the royalty fee to the recipient if it's greater than 0
                if (royaltyFee > 0 && recipient != address(0)) {
                    if (baseToken != address(0)) {
                        ERC20(baseToken).safeTransfer(recipient, royaltyFee);
                    } else {
                        recipient.safeTransferETH(royaltyFee);
                    }
                }
            }
        }

        // emit the buy event
        emit Buy(tokenIds, tokenWeights, netInputAmount, feeAmount, protocolFeeAmount, royaltyFeeAmount);
    }

    function sell(
        ...
    ) public returns (...) {
        // ...

        uint256 royaltyFeeAmount = 0;
        for (uint256 i = 0; i < tokenIds.length; i++) {
            // transfer each nft from the caller
            ERC721(nft).safeTransferFrom(msg.sender, address(this), tokenIds[i]);

            if (payRoyalties) {
                // calculate the sale price (assume it's the same for each NFT even if weights differ)
                uint256 salePrice = (netOutputAmount + feeAmount + protocolFeeAmount) / tokenIds.length;

                // get the royalty fee for the NFT
                (uint256 royaltyFee, address recipient) = _getRoyalty(tokenIds[i], salePrice);

                // @audit - here is vulnerability - first check the royalty Fee and recipient != address(0) then add royaltyFee to royaltyFeeAmount

                // tally the royalty fee amount
                royaltyFeeAmount += royaltyFee;

                // transfer the royalty fee to the recipient if it's greater than 0
                if (royaltyFee > 0 && recipient != address(0)) {
                    if (baseToken != address(0)) {
                        ERC20(baseToken).safeTransfer(recipient, royaltyFee);
                    } else {
                        recipient.safeTransferETH(royaltyFee);
                    }
                }
            }
        }

        // subtract the royalty fee amount from the net output amount
        netOutputAmount -= royaltyFeeAmount;

        if (baseToken == address(0)) {
            // transfer ETH to the caller
            msg.sender.safeTransferETH(netOutputAmount);

            // if the protocol fee is set then pay the protocol fee
            if (protocolFeeAmount > 0) factory.safeTransferETH(protocolFeeAmount);
        } else {
            // transfer base tokens to the caller
            ERC20(baseToken).transfer(msg.sender, netOutputAmount);

            // if the protocol fee is set then pay the protocol fee
            if (protocolFeeAmount > 0) ERC20(baseToken).safeTransfer(address(factory), protocolFeeAmount);
        }

        // ...
    }

```

### Key Concepts

- Sell and Buy NFT - Selling and buying of NFT and Distribute the royalty fee for trader
- Inconsitency in royalty collection and Buy distribution can loss of funds for trader
- Fee will be collected from traders but won't be distributed to royalty recipient

## Potential Impact
- Trader/Buyer loss 
- Less reliable Protocol

## How to Identify This Sort of Vulnerability
- Check the state changes and Where the amount add between the asymmetry function like sell/buy deposit/ withdraw etc
- Check for `address(0)` so We cannot loss the amount

## Proof of concept
- A buyer initiates the buy call for an NFT.
- The PrivatePool.buy function queries the _getRoyalty function which returns 10 WETH as the royaltyFee and 0x00 address as the royalty recipient.
- This 10 WETH value will be added to the royaltyFeeAmount amount and will be collected from the buyer.
- But since the recipient address is 0x00, the 10 WETH royalty amount will not be distributed.
- The 10 WETH amount won't be returned to the buyer either. It just simply stays inside the pool contract.
- The buyer here suffered loss of 10 WETH.
- A similar scenario is possible for the NFT sell flow.

## Recomended Mitigation steps
Consider collecting royalty amount from traders only when the royalty recipient is non-zero

```solidity
    if (royaltyFee > 0 && recipient != address(0)) {
        royaltyFeeAmount += royaltyFee;
    }

```