# THIS IS AN ERROR I THOUGHT IT WAS REAL BUT ITS'S NOT
# THERE IS A LOGICAL FLAW HERE, TRY TO FIND IT 🕵️
## solution is at the end

## 2. Block Validators Potential Manipulation 🚨

### **Problem Description:**

The library `OracleLib.sol` uses the Solidity's `block.timestamp` to calculate the time since the last update from a Chainlink contract:

```
function staleCheckLatestRoundData(AggregatorV3Interface chainlinkFeed) public view returns (uint80, int256, uint256, uint256, uint80) {
    //...
    line 31: uint256 secondsSince = block.timestamp - updatedAt;
    //...
}
```

The risk is created when combining this factors: 

- `updatedAt` value is fetched from an external Chainlink aggregator contract. 
- Block validators have a degree of freedom in determining the `block.timestamp` value.

If the Chainlink aggregator updates its contract in the same block as someone activates a call to `staleCheckLatestRoundData()`, a block validator could maliciously adjust the `block.timestamp` to be a slightly smaller value than the `updatedAt` timestamp. This would trigger an underflow due to the result of the substraction being assigned to an `uint256` type, which will lead to the transaction reverted.

The gravity of this comes by the fact that the `staleCheckLatestRoundData()` is called from several functions within the `DSCEngine.sol` contract, including vital operations like liquidation. If a validator causes a valid liquidation to revert and then sends another transaction from their own address to claim the collateral, the system's fairness is compromised. This potentially disincentivizes competition and might result in severe consequences.

### **Affected Functions:**

The following 15 functions in `DSCEngine.sol` eventually call `staleCheckLatestRoundData()`:

```
liquidate(address, address, uint256) external
redeemCollateralForDsc(address, uint256, uint256) external
redeemCollateral(address, uint256) external
burnDsc(uint256) external
depositCollateralAndMintDsc(address, uint256, uint256) external
mintDsc(uint256) public
revertIfHealthFactorIsBroken(address) internal view
_healthFactor(address) private view returns (uint256)
getHealthFactor(address) external view returns (uint256)
getAccountInformation(address) external view returns (uint256, uint256)
_getAccountInformation(address) private view returns (uint256, uint256)
getAccountCollateralValue(address) public view returns (uint256)
getTokenAmountFromUsd(address, uint256) public view returns (uint256)
getUsdValue(address, uint256) external view returns (uint256)
_getUsdValue(address, uint256) private view returns (uint256)
```

### **Proposed Solution:**

To mitigate the potential underflow and ensure operation reliability, it's recommended to verify that the `block.timestamp` isn't earlier than `updatedAt` prior to the subtraction.

Or allow the substraction to have negative values and then set a threshold in those negative values to see if the price recieved is within a valid or not and it's only negative due to block validators causes.

# Why this is incorrect?

- Sequential Timestamp Progression: Block validators ensure that each new block's timestamp is always greater than its predecessor's. Therefore even if it's true block validators have some range of freedom in block.timestamp value, it's impossible for a new block's timestamp to be set before the timestamp of the last block.

- Consistent Block Timestamps: All transactions within a single block share the same timestamp. If the Chainlink price update and another transaction are included in the same block, the difference between their timestamps will always be zero or a positive number.

Given these constraints, the potential for an underflow due to malicious timestamp manipulation is negligible. Thus, the concern was unfounded.

Validators are humans and can have this kind of errors, so because of this and because audits are not silver bullets... always take your audits as reference and not as the the total truth. 😄
