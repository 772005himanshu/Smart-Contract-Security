## Arithmetic 

- Simple calculation(Token transactioon, financial calculation and logic implmentation) Error leads to heavy loss

1. Arithmetic Pitfall 1: Division By 0

In solidity , if a division by zero occurs , the contract will automatically revert and the transaction will fail. This can be exploited by malicious actors to disrupt contract functionalities, cause denial of servies or hinder , such as token withdawals or rewrds distribution

```solidity
contract VulnerableContract {
    function vulnerableDivision(uint256 numerator , uint256 denominator) public pure retunrs(uint256) {
        return numerator / denominator;
        // No Proper Validatiom here 
    }
}

```

Mitigation:

```solidity
require(denominator != 0 ,"Denominator cannot be zero");
```
- use Safe Math Libraries 

2. Arithmetic Pitfalls : Precision Loss Due to Rounding
The Multifaceted Implications of Precision Loss:
- Token Value Erosion
- Inequitable Distribution(reward or asset distribution)
- Vulnerability to DDoS Attacks(overburdening the network, disrupting services, and causing accessibility issues, leading to a trust deficit in the system’s robustness.)
- Systemic Value Loss

Example illustrating Precision Loss:

```solidity
contract PrecisionLossExample {
    function divide(uint256 a, uint256 b) public pure returns(uint256) {
        return  a/b;
    }
    // If the function divide(3, 2) is called, it will return 1, instead of 1.5 or 2.

} // Consider same example in User / reward distribution lead to user getting less rewards
```

- Perform Multiplication Before Division
- Utilize Scaling Techniques
```solidity
result = (a * 1000) / b;
result = result / 1000; // Scaling down the result
```
- Adopt Fixed-Point Libraries - Consider employing fixed-point arithmetic libraries like ABDK Math 64.64, which facilitates operations involving real numbers in Solidity, helping mitigate precision loss.

3. Erroneous Calculations
- Reading the docs and Check they implemented the same logic or not ??
- Missing Check for input coming from the user end ??