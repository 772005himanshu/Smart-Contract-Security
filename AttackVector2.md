## 1/64 Rule Attack Vector

## Report 1
### [M] Gas Manipulation by Malicious Winners in `claimPrizes` Function

**Description:** A malicious winner can exploit the `claimPrizes`
function in the `Claimer` contract by reverting the transaction through returning a huge data chunk. This Manipulation can cuase the transaction to ruun out of gas , preventing legitmate claims and allowing the malicious user to claim prizes without computing winners

**Vulnerability Details:** 
- The Climer contract allow users to claim prizes on behalf of others by calling the `claimPrizes` function
- A malicious Winner can exploit this function by returning a huge data chunk when called , causing the transaction gas to be too high and reverts
- Although the function catches the revert the remaining gas (63/64 of the original gas) is likely insufficient for the rest of the claims
- Then malicious winner can then replay the transaction to claim the fees from the first claimers computation without needing to compute the winner themseleves

**Summmary:**
(Then attacker could call claimsPrizes function returninf large data chunk , the transaction reverts , but transaction used the 1/64 of original gas , the remaining gas is not sufficient to rest of the claims -> malicious actor then replay the transaction fee -> winner)


**Impact:** 
- Legitmate claimers may loss gas fee due to transaction reverts caused by malicious winners

- Malicious winners can exploit this to claim prizes without computing winners, undermining the fairness of the prize distribution


**Recommended Mitigation:** 

- Implement a gas limit check to ensure that sufficient gas remains for the rest of the claims after catching a revert.

- Consider adding a mechanism to penalize or blacklist addresses that attempt to exploit this vulnerability.


## Report 2

### [M] Withdrawal queue can be forcibly activated to hinder bridge operations  - High Vulnerability Should not get the Explanation 

The RootERC20PredicateFlowRate contract implements a withdrawal queue to more easily detect and stop large withdrawals from passing through the bridge. This queue can be activated in four different ways: if a token's flow rate has not been configured by the rate control admin, if the withdrawal amount is larger than or equal to the large transfer threshold for that token, if the total withdrawals of that token are larger than the defined token capacity, or if the rate controller manually activates the queue. Once the queue is active, all withdrawals from the bridge must wait a specified time before the withdrawal can be finalized. This can be exploited by malicious actors to hinder the expected operation of the bridge.


