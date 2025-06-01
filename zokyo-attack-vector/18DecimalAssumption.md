# 18 Decimal Assumption

The 'decimals' property determines how the token can be subdivided, with 18 decimals being a common configuration for many tokens. However, this isn't a one-size-fits-all scenario.

The Critical issue arises when a smart Contract interact with ERC20 tokens and make a faulty assumption that all token have 18 decimals , which is not universally true.

Token decimals ranging from 0 to 18 and smart contract that interact with these token must account for this varaibility to function correctly 

Incorrect Decimals assumption canlead to calculation errors, undervaluted transactions and other mafunctions 

## UnderStanding ERC20 Tokens

ERC20 is standard used for creating fungible Token on Ethereum blockchain 

Decimal Property: A Close Look
One of the critical properties of ERC20 tokens is the `decimals` attribute.
The `decimals` property is vital for representing token values accurately, allowing for transactions and computations involving fractions of tokens.

- Common Standard: 18 Decimals (1 Ether = 10 ^ 18 Wei) Smallest Unit it is easy to represnt a token in smallest unit , transfer with the help of this because we donot use float value

- Varaibility in Decimal - We can use any decimal between (0 - 18)

Importance of Decimals Awareness

- Calculation Error
- Transaction Failure - Smart contracts that enforce specific value constraints might malfunction if the decimal count is not what the contract assumes, leading to failed transactions.
- Asset Misrepresentation

## Example of Vulnerabilities To Do with Assuming 18 Decimals

- TWAP Oracle doesn't calculate VADER:USDV exchange rate correctly - Assumption of 18 decimals

In Twaporacle.sol the code perform the following calculations

```solidity 
coderesult = ((sumUSD * IERC20Metadata(token).decimals()) / sumNative);
```

Breaking down the vulnerability 

For an ERC20 token for 18 decimals , the calculation effectively becomes:

```solidity
codeResult = ((sumUSD * 18 )/ sumNative);
```

This calculation is problematic and counterintuitive, as it arbitrarily multiplies the sumUSD by the number of decimals (which in this case is assumed to be 18), rather than accurately scaling the value based on the token's actual decimal count. This approach results in inaccurate and misleading values

Correct Contract Should look like this 

```solidity
uint256 scalingFactor = 10 ** IERC20Metadata(token).decimals();
result = (sumUSD * scalingFactor) / SumNative
```

- Wrong Token alloaction computation for token decimals != 18 if floor price not reached

Impact:
The vulnerability arises within the LaunchEvent.createPair function, particularly when the set floor price is not attained — identified by the condition `floorPrice > wavaxReserve * 1e18 / tokenAllocated`. Under this circumstance, the number of tokens dispatched to the pool is adjusted downwards to align with the raised WAVAX at the designated floor price

While the validation check is implemented correctly , floorPrice > (wavaxReserve * 1e18 ) / tokenAllocated os coorectly implemented , the subsequent computation for `tokenAllocated` introduces a vulnerability due to its handling token decimals

```solidity
tokenAllocated = (wavaxReserve * 10**token.decimals()) / floorPrice;
```

Recommendation:

```solidity
tokenAllocated = (wavaxReserve * 1e18 ) / floorPrice;
```

- `makerPrice` assumes oracle prices is always in 18 decimals

Impact:

The EIP1271Wallet._validateOrder function is responsible for calculating a makerPrice. This calculated value inherently possesses a decimal count equivalent to that of takerAmount, which consistently stands at 18 decimals because the takerToken is invariably WETH. The makerPrice is then juxtaposed with the priceFloor value returned from a Chainlink oracle. For the comparison to be coherent and accurate, the priceFloor value must also be expressed in 18 decimals.

However, a significant risk is lurking here: while the old, now deprecated, Chainlink API used to return values in 18 decimals, there is no guarantee that future or updated APIs from Chainlink will maintain this format. Relying on the assumption of 18 decimals without verification or adjustment introduces a potential vulnerability and might lead to miscalculations and misinterpretations in the EIP1271Wallet._validateOrder function

- Tokens with `decimals` larger than `18` are not supported

Vulnerability Details
Tokens that posses a decimals count exceeding 18 decimals introduce a serious vulnerability across various function in the codebase . these function are prone to reverting due to underflow issue when interacting with such token

```solidity
function getPriceFromDex(address _tokenAddress) public view returns (uint256) { 
...
uint256 tokenDecimalDelta = 18 - uint256(IERC20Extended(priceInfo.token).decimals());
...
uint256 baseTokenDecimalDelta = 18 - uint256(IERC20Extended(priceInfo.baseToken).decimals());
...
```

```solidity
precisionMultipliers[i] = 10**uint256(SwapUtils.POOL_PRECISION_DECIMALS - decimals[i]);
```

```solidity
function getPriceFromChainlink(address _tokenAddress) public view returns (uint256) {
...
uint256 price = retVal.mul(10**(18 - uint256(aggregator.decimals())));
...
```


Each of these functions performs calculations where an 18-decimal standard is presumed. Consequently, tokens with decimal counts that deviate from this assumption, especially those with more than 18 decimals, are not supported and lead to calculation errors or function reverts

Recommendatioon:

- Decimals Count verifications

Introduce conditional checks to ascertain the decimal count of the tokens involved in calculations. If a token’s decimal count exceeds 18, implement normalization procedures to align the decimal count with the expected 18-decimal format.

- Normalization Procedure 
For tokens with decimal counts surpassing 18, normalize their values by dividing them by the difference in their decimal count and the 18-decimal standard. This normalization should be conducted carefully to avoid loss of precision and to ensure that subsequent calculations within the functions remain accurate and reliable.

- Wrong Price Scale For `GasOracle` - handling with two chainlink prices (ETH , USDC) - donot Forget to scale upto Recommended factor 

- ERC20Rewards breaks when setting a different token

The setRewards function is designed with the capability to designate a different token for rewards. However, this flexibility inadvertently introduces a critical vulnerability: holders from a previous reward cycle who haven't claimed their rewards are subject to receiving their dues in a new token once it's set. This transition is problematic, particularly when the newly assigned token has a different value or operates with a different decimal structure than its predecessor.


Recommendation : Calcuate the price of USDC using cahinlink then calculate the rewards amount

