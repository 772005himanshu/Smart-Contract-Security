## Integer Underflow and Overflow Vulnerabilities in Solidity(Before 0.8.0)

Arithmetic Operation are backbone of smart Contract functionalities being instructional in asset transaction , logical decision-making etc

Integer underflow and overflow are common vulneranilities in solidity especially in version before 0.8.0 

```solidity
pragma solidity ^0.7.0;

contract OverflowExample {
    uint8 public counter = 255;

    function overflow() public {
        counter += 1; // Here the counter overflows
    }
}
```

In this example, since counter is a uint8, it can hold values from 0 to 255. When we try to increment 255 by 1, it wraps around and becomes 0, an instance of integer overflow

Integer underflow

```solidity
pragma solidity ^0.7.0;

contract UnderflowExample {
    uint8 public counter = 0;

    function underflow() public {
        counter -= 1; // Here the counter underflows
    }
}
```
In this example, decrementing 0 by 1 results in 255 due to underflow since the variable `counter` is of type `uint8.`

After 0.8.0:
Solidity cause transaction to revert when undeflow and oerflow condition met.

For Contract Deployed with older version:

- Use Safe Math Libraries
- Manual Checks

