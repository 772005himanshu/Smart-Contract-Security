## Unsafe Casting
- Refers to the act of converting one data type to another in a way that can lead to unexpected or incorrect outcomes. 

- This issue usually surfaces when variables are downcasted or converted into a smaller data type, without the necessary checks for possible overflows or underflows.

## Understanding Data Type in Solidity
- It occurs when we are converting the larger type to set in Smaller data types
- Integer overflow happens when an arithmetic operation attempts to create a numeric value that is outside of the range that can be represented with a given number of bits, resulting in a value wrapping around to the opposite extreme.

#### Understanding Type casting
- Type casting is the process of converting one data type to another. In Solidity, type casting can be done explicitly by the developer using the syntax `type(variable)`. For instance, `uint(x)` casts an integer x to an unsigned integer
- casting a negative int to a uint can yield an extraordinarily high value, because uints in Solidity cannot represent negative numbers. Similarly, casting a large uint to a smaller type (like uint256 to uint128) may cause an overflow, with the high order bits of the larger number simply being discarded.
-  Solidity's type casting is `unchecked` by default, meaning that it will silently produce a potentially incorrect result without any warnings or errors

### Example Of the type casting bugs
1. Unsafe Cast on oracle retunr Value

```solidity
newProtocolEquity = oracle.pcvStats(); // if this give the return in int256 value 
newProtocolEquity = uint256(newProtocolEquity);  // Unsafe cast, no check for negative values
```

2. Unsafe cast in vesting

The line of code of interest here is:
```solidity
vester.amount -= uint192(vestedAmount);
```
This statement does two things:

- It subtracts `vestedAmount` from `vester.amount` (which reduces the amount of tokens still to be vested for the user).

- It casts the result to a `uint192` type.

- In Solidity does not revert or throw an error. Instead, it will silently overflow, i.e., lose the data exceeding its maximum storage capacity, which may lead to incorrect computations or even allow for potential exploits.

- To mitiagte this issue use the openzeppelin library, safeCast function will revert the transaction if an overflow occurs, thus protecting against potential exploits or bugs.

```solidity
vester.amount = SafeCast.toUint192(SafeMath.sub(vester.amount, vestedAmount));// Some code - first subtract it  and then covert and safeCast function
```

3. Unsafe uint128 casting may overflow

```solidity
rewardIntegral = uint128(rewardIntegral) + uint128(((bal - rewardRemaining) * 1e20) / _supply);
reward.reward_integral = uint128(rewardIntegral);
```
Converting the `uint256` to `uint128` should be an issue of unsafe casting

So use the safecast library from the openzeppelin
```solidity
import "@openzeppelin/contracts/utils/math/SafeCast.sol";

// In your contract
using SafeCast for uint256;

// Later in your function
rewardIntegral = rewardIntegral.toUint128() + (((bal - rewardRemaining) * 1e20) / _supply).toUint128();
reward.reward_integral = rewardIntegral.toUint128();
```

4. Implicit Under-flows - High Vulnerability

- In Solidity `0.8.x`, underflows and overflows are automatically checked and will cause a revert.

- The original problems occur when a subtraction operation is performed between `uint` values or a `uint` value is negated, and the result is then cast to `int256`. This may lead to incorrect values due to underflow

- The mitigation steps in a more generalized form are:

1. For subtraction operations, ensure that each operand is cast to `int256` before performing the operation. So replace `int256(a-b)` with `int256(a)-int256(b)`.

2. For negation operations, first cast the uint value to int256 before applying the negation. So replace `int256(-x) `with `-int256(x)`.


### Overall Mitigation Of Unsafe Casting:

- Use the OpenZeppelin SafeMath library for basic arithmetic operations: SafeMath helps handle integer overflows and underflows automatically by reverting the transaction.

- Use the OpenZeppelin SafeCast library for type casting: SafeCast provides functions for safely casting between different integer types.

- For subtraction operations, ensure each operand is cast to the appropriate type before performing the operation. Instead of int256(a-b), use int256(a)-int256(b).

- For negation operations, first cast the uint value to int256 before applying the negation. Instead of int256(-x), use -int256(x).