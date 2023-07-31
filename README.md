# Audit Report:

# [Decentralized Stable Coin - Foundry Course](https://www.youtube.com/watch?v=wUjYK5gwNZs&t=11630s) by [Patrick Collins](https://github.com/PatrickAlphaC)

## author: [CarlosAlegreUr](https://github.com/CarlosAlegreUr)

## last update: July 23, 2023

# DISCLAIMER & CONTEXT , PLEASE READ BEFORE ANYTHING🧑‍💼⚠️

This audit is not profesional! It was **created to gain experience and for educational purposes**. 😸

The audit could be more complete, it **has analysis errors and mistaken conclusions**. It was made quickly and with a **time limit** to help me **simulate the audting process** and gain experience on the steps needed to audit a code.

Specifically I did it to get more familiariazed with this code:

[🔗 foundry-defi-stablecoin-f23](https://github.com/Cyfrin/foundry-defi-stablecoin-f23/tree/main#deployment-to-a-testnet-or-mainnet)

The time limit aimed to simulate a more realistic working enviroment. The deadline date was set to the beginning of the first auditing contest from [🦅CodeHawks🦅](https://www.codehawks.com/).

By the way `auditExercise.md` is a "false positive error" I thought the code had but it actually didnt. It's an exercise for you if you want to try to figure out why the things written there are actually wrong. Answers to this are at the end of the same file though.

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Audit Details](#audit-details)
  - [Summary of Findings](#summary-of-findings)
  - [Scope](#scope)
  - [Severity Criteria](#severity-criteria)
  - [Tools Used](#tools-used)
- [High](#high)
- [Medium](#medium)
- [Low](#low)
- [Informational](#informational)
- [Gas](#gas)
- [Known Issues Ignored](#known-issues-ignored)
- [Hypothetical or Unlikely Scenarios](#hypothetical-or-unlikely-scenarios)

# Protocol Summary

According to the README and comments provided by the client:

This project is meant to be a stablecoin where users can deposit WETH and WBTC in exchange for a token that will be pegged to the USD.

# Audit Details

## Summary of Findings

- A high-risk vulnerability has been identified.
- 1 medium risk, more like annoying bug.
- 2 potential attack vectors with low risk.
- Gas optimization opportunities have been identified.
- Recommendations for improving test coverage and improving code readability.

## Scope

The whole repo: including contracts, scripts and tests.

| Symbol | Description      |
| ------ | ---------------- |
| ✅     | File revised     |
| 🟡     | Only part of it  |
| 🔴     | File not revised |

Due to deadlines this is the whole coverage of the audit:

#### Contracts

```
src/DSCEngine.sol ✅
src/DecentralizedStableCoin.sol ✅
src/libraries/OracleLib.sol ✅
```

#### Scripts

```
script/HelperConfig.s.sol ✅
script/DeployDSC.s.sol ✅
```

#### Tests

```
test/unit/DSCEngineTest.t.sol 🟡 (until line 173)
test/unit/DecentralizedStablecoinTest.t.sol ✅
test/unit/OracleLibTest.t.sol ✅

test/fuzz/failOnRevert/StopOnRevertHandler.t.sol 🔴
test/fuzz/failOnRevert/StopOnRevertInvariants.t.sol ✅
test/fuzz/continueOnRevert/ContinueOnRevertHandler.t.sol 🔴
test/fuzz/continueOnRevert/ContinueOnRevertInvariants.t.sol 🟡 (just revised 1 invariant lines:56-71)

test/mocks/MockV3Aggregator.sol ✅
test/mocks/MockFailedTransferFrom.sol ✅
test/mocks/MockMoreDebtDSC.sol 🔴
test/mocks/MockFailedMintDSC.sol 🔴
test/mocks/MockFailedTransfer.sol 🔴
```

## Severity Criteria

- **High**: Critical vulnerabilities that can lead to system breakdowns or exploitable loopholes.
- **Medium**: Significant issues that cause inconvenience or minor disruptions, but aren't critical.
- **Low**: Minor concerns, often related to things like compiler versions or similar technicalities.
- **Informational**: Recommendations aimed at improving documentation and enhancing code readability.

## Tools Used

- **Manual Audit**: Individual scrutiny of the code.
- **Slyther**: Static analysis tool.
- **ChatGPT 4**: Assisted in code understanding and analysis.
- **Surya**: Function calls dependacy graph creator trhough _Solidity Visual Developer_ in it's VSCode edition.

> Note: **Potential Enhancements**: Given more time, I would have tested new invariants and expanded fuzz testing.

---

> 🚨 **Disclaimer**: For genuine audits, it's essential to consult with the client regarding the use of third-party AIs for analyzing documentation or code. While smart contracts become open-source once deployed, there may be scenarios where a contract is audited pre-launch.

# High

## 1.- Potential Solvency Manipulation 🚨

Any address with adequate funds can artificially manipulate the perceived solvency state of the `DSCEngine` contract.

### Cause:

The issue comes from this two functions in `DSCEngine.sol`:

```
function depositCollateral(address tokenCollateralAddress, uint256 amountCollateral) public

function redeemCollateral(address tokenCollateralAddress, uint256 amountCollateral) external
```

By using the `depositCollateral()` function, addresses can deposit collateral which isn't actively backing any `DSC` debt. Furthermore, the address that deposited this "collateral" can effortlessly retrieve it anytime by calling `redeemCollateral()`, without facing any restrictions or penalties.

### Implication:

Such actions can create scenarios where a significant number of user positions might look solvant due to the collateral being "owned" by the engine. But actually not due to the lack of restrctions on the `redeemCollateral()` function, which makes those tokens effectively owned by the original address who sent them. The system could appear healthy when checked using the `invariant_protocolMustHaveMoreValueThatTotalSupplyDollars` invariant test. However, this can change instantly if addresses with "fake collateral" (collateral that isn't supporting any debt) decide to retrieve it.

This vulnerability allows a single address or a group of addresses to control the solvency of the system, thus contradicting the objective of creating a decentralized and trustworthy engine.

### Proposed Solutions:

0. **The invariant needs to be changed**: The current approach fetches the balance using `ERC20Mock(weth).balanceOf(address(dsce));`. As highlighted, this does not truly reflect the actual balance under the engine's control.

1. **Restrictive Collateral Entry**: Ensure that any collateral introduced into the system is directly associated with a corresponding debt. This can prevent the system from accepting "fake collateral" that can be withdrawn at will.
2. **Collateral Retrieval With Debt Condition**: Implement a mechanism where users can only retrieve their collateral if they have any debt.

> ⚠️ **Note**: Both last solutions can be implemented independently or together, depending on the desired system complexity and objectives.

# Medium

## 1.- Inaccurate Health Factor Calculation for Small Values ⚠️

In `DSCEngine.sol`, in the method `_calculateHealthFactor` an issue arises when the collateral value in USD is smallet than 2.

For instance, if the `collateralValueInUsd` is $1.5 and `totalDscMinted` is 1, the calculated health factor should be valid. However, due to the way integer division is handled in Solidity, the health factor truncates to 0, which is not valid.

This discrepancy can lead to user confusion and dissatisfaction, as they may not be able to redeem small quantities of collateral even when it should be technically possible based on their position's health.

The relevant code is in `DSCEngine.sol`, line 320:

```
function _calculateHealthFactor(uint256 totalDscMinted, uint256 collateralValueInUsd)
    internal
    pure
    returns (uint256)
{
    if (totalDscMinted == 0) return type(uint256).max;
    uint256 collateralAdjustedForThreshold = (collateralValueInUsd * LIQUIDATION_THRESHOLD) / 100;
    return (collateralAdjustedForThreshold * 1e18) / totalDscMinted;
}
```

To prevent truncation and ensure accurate health factor calculation, we can reorder the operations for better precision (didnt test it):

```
function _calculateHealthFactor(uint256 totalDscMinted, uint256 collateralValueInUsd)
    internal
    pure
    returns (uint256)
{
    if (totalDscMinted == 0) return type(uint256).max;

    // Multiply first to prevent truncation, then divide
    return (collateralValueInUsd * LIQUIDATION_THRESHOLD * 1e18) / (totalDscMinted * 100);
}
```

> Warning ⚠️: The constants and mathematical computations used elsewhere in the contract haven't been fully reviewed, so the proposed changes assume other values and calculations are correct.

> Warning 2 ⚠️ Given the potential for totalDscMinted to be an odd number and the result of the division possibly having decimal places, I'm not sure whether the calculation will be precise. This hasn't been fully verified.

# Low

## Potential for Reentrancy

- **DSCENGINE - Line 289**:
  - Ensure that `i_dsc` will always be valid. Given that it's set in the constructor and is immutable, this is a reasonable assumption.
  - However, if there's ever a concern, consider adding a non-reentrant modifier here as an added security measure.

## Solidity Version Recommendations

- Slither recommends using Solidity version `0.8.18` for deployments.
  - Although using `0.8.19` should be okay if all tests are passing. It's a safer approach to use a version that has been exposed more to real-life enviroments.s

# Informational

## Documentation Improvements

### NatSpec Comments

- Ensure to use the correct NatSpec format.
  - Correct format: `/** --Comments-- */`
  - Found in provided contracts: `/* --Comments-- */`

### Orthographic Corrections

- File: `./src/DecentralizedStableCoin.sol`
  - Line 3: Change "volitility" to "volatility".

## Code Style Recommendations

### DSCEngine.sol

- **Line 244**: Consider using `!minted` directly for better code readability.

### Naming Conventions

- Your project uses clear naming conventions, which are easy to follow. However, if your project is intended to be used beyond your team, it's advisable to adhere to standard Solidity naming conventions.

### OracleLibTests.t.sol

- **Line 26**: It appears there's a redundant comment. If the code accomplishes what the comment describes, consider removing the comment for cleaner code.

# External Dependencies

## WBTC Security Recommendations

Friendly reminder that wBTC could rely on trusted bridges. If the priority is true decentralization, consider utilizing a more decentralized solution, such as The Ren Protocol. Though it might not be yet a decentralized assurance method, it offers a better alignment with decentralization.

# Scripts & Tests

## Script Analysis

### `DeployDSC.s.sol`

- **Line 29**:
  - When creating staging tests, it's advisable to wait for some blocks before further interactions. This ensures all your transactions are successfully processed.
  - Ensure the deployer's address in staging environments is properly funded.

## Test Analysis

### `OracleLibTest.t.sol`

- Improve the testing for the `staleCheckLatestRoundData()` function:
  - Implement fuzz testing on it.
  - Design different unit tests where the `if` conditions on lines 28 and 32 (from `OracleLib.sol`) experience different boolean states.

### `DSCEngineTest.t.sol`

- **Line 50, answer to the question in a comment: Should we put our integration tests here**:

  - For improved code organization, it's recommended to create a separate folder at the same directory level as unit tests, named `integration tests`. Within unit tests, focus solely on evaluating individual smart contract functions in isolation.

- **Lines 63, 64**:

  - ERC20 MOCKS might not be deployed in Sepolia. If this is the case, these lines should be included inside the Anvil chain ID conditional.

- **Line 174**:
  - Another mock is being used here. I assume that unit tests are not be designed to run on Sepolia. If they are intended to, ensure that the necessary mocks are deployed prior to test execution.

## New Invariant sugestions

- Create some solvent addresses and avoind using them during fuzzing. After the runs, as invariant, check that their accounts remain the same.

## Deployment Safety Recommendations

For `DSCEgine.sol`:

### Constructor Validations:

- Currently, there's no validation for a valid `DecentralizedStableCoin.sol` address.

  While we can the assume that the deployer has no reason to provide an incorrect address, implementing a validation could potentially save gas if an incorrect address is accidentally passed during deployment.

### Implementation Approaches:

1. **Hardcode Address (Recommended by it's simplicity)**:

   - Deploy the `DecentralizedStableCoin.sol` first.
   - Then, hardcode its address as a constant in the `DSCEgine.sol` contract. This approach stands out for its simplicity and straightforwardness.

2. **Contract Hash Verification**:
   - Use a contract hash to verify the correct contract address:
     ```
     bytes32 expectedCodeHash = ...; // previously computed value
     bytes32 actualCodeHash;
     assembly {
         actualCodeHash := extcodehash(dscAddress)
     }
     ```
   - Compare `expectedCodeHash` and `actualCodeHash` to ensure they match.

Choose the method that you prefer.

## Test Coverage

### Should be created if you have time

- Add some tests for your scripts for a more robust project.
  script/DeployDSC.s.sol  
  script/HelperConfig.s.sol

- If adding extra checkings as recomended for DSCEngine constructor you must implement test for them.

### Must increse

> Note ⚠️: Test your mocks for a more robust test system. You might as well add a new directory for them called `mocksTest` for example.

test/mocks/MockFailedMintDSC.sol  
test/mocks/MockFailedTransfer.sol  
test/mocks/MockFailedTransferFrom.sol

### Should increase

- Improve coverage in:
  test/mocks/MockMoreDebtDSC.sol

# Refactors:

- DecentralizedStableCoin.sol: Rvert Error DecentralizedStableCoin\_\_MustBeMoreThanZero could me made modifier as it's used multiple times.

- DSCEngine.sol line 379: You could modularizes this part of the code with a getTokenPriceFromAggregator() for example. Found duplicated in lines 314,315.

## Tests refactor

I noticed a little refactor it would save some lines of code:

- Create a modifier or private func: approveBeforeTranasfer(address token, address aprovingFor, uint256 quantity);

> Note ⚠️: Always remember to check the tests after every change to the code.

# Gas

> Warning ⚠️: These changes haven't been implemented. If applied, some tests might fail. The failures could be due to how the tests are written rather than core functionality issues. Regardless, always ensure tests are passing after making any modifications.

- DSCEgine.sol (lines: 119-122):

constructor: For better gas usage create a collateralTokens array in memory, push objects there and then make it equal to the one in sotrage:

Optimized code:

```
   constructor(address[] memory tokenAddresses, address[] memory priceFeedAddresses, address dscAddress) {
        if (tokenAddresses.length != priceFeedAddresses.length) {
            revert DSCEngine__TokenAddressesAndPriceFeedAddressesAmountsDontMatch();
        }
        // These feeds will be the USD pairs
        // For example ETH / USD or MKR / USD
        address[] memory collateralTokens = new address[](tokenAddresses.length); // <--- Here
        for (uint256 i = 0; i < tokenAddresses.length; i++) {
            s_priceFeeds[tokenAddresses[i]] = priceFeedAddresses[i];
            collateralTokens[i] = tokenAddresses[i]; // <--- Here
        }
        s_collateralTokens = collateralTokens; // <--- Here
        i_dsc = DecentralizedStableCoin(dscAddress); // No checkings for valid address, we can assume the deployer will never want to mess up with the constructor though.
    }
```

- DSCEngine.sol (lines 373-382):
  Dave .length in memory before loop execution. (slither recommenation, I tested it on Remix IDE and it didn't save gas, other way around it made it more expensive)

Also delcare but not initialize token and amount too. (this does save gas, specially adds up if DSC supports large num of tokens)

Optimized code:

```
  function getAccountCollateralValue(address user) public view returns (uint256 totalCollateralValueInUsd) {
      address token; // <-- Here
      uint256 amount  // <-- Here
      for (uint256 index = 0; index < s_collateralTokens.length; index++) {
          token = s_collateralTokens[index];
          amount = s_collateralDeposited[user][token];
          totalCollateralValueInUsd += _getUsdValue(token, amount);
      }
      return totalCollateralValueInUsd;
  }
```

#### Unnecessary checks

The following changes are unnecessary checks that can be deleted to save gas, deleting them will make your code "less readable" as it wouldnt set the statements in the direct contract, keeping them is a choice on gas saved and how easily the code can be read. Furthermore maybe changing some might lead to test needing some rewriting:

- DecentralizedStableCoin.sol

  - Lines 57, 70: do in uint256 `variable` == 0 instead of `variable` <= 0. In the Solidity's version used the less than 0 is built in for uint256 variables.

  - Lines 60-62:
    The following code is unnecessary as `super.burn(_amount)` eventually triggers
    ERC20.\_burn(address, amount) from OpenZeppelin. This function already includes
    a revert for this condition. As a result, the error message defined on line 43 can be removed.

  - Lines 67-72: Those checks are already made or by the `\_mint` function
    or by the built in uint256 < 0 solidity.

    If deleted all errors line 42-44 are not needed.

    If keeped, to save a bit of gas, do in uint256 `variable` == 0 instead of `variable` <= 0, it's redundant and less straightforward.

- DSCEngine.sol

  - Line 84: in `_burnDsc` when calling i_dsc, > 0 is already checked
  - Line 242: Not real need for modifier checking, mint in i_dsc checks for == 0.

# Known Issues Ignored

- We use storage variables instead of immutables for storing the addresses of the collateral. You can ignore this.

- If the protocol ever becomes insolvent, there is almost no way to recover. This is a known issue.

# Hypothetical or Unlikely Scenarios 🌪️

While this scenarios are highly unlikely, it's essential to consider every possible edge case when auditing a system. Take them more as a mental expermient. In a real audit this extremely unlikely scenarios migth not be worth it to analyze due to time limitations.

### 1.- Block Verifier Manipulation if Chainlink Fails

While Chainlink's robustness and resilience make this scenario highly unlikely, here we're exploring a hypothetical scenario where Chainlink fails temporarily and how block validators could potentially exploit the system during this window.

The potential issue arises from `OracleLib.sol` in the `staleCheckLatestRoundData()` function, specifically at line 32:

```
  function staleCheckLatestRoundData(AggregatorV3Interface chainlinkFeed)
        public
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
        // code...
        uint256 secondsSince = block.timestamp - updatedAt;
        if (secondsSince > TIMEOUT) revert OracleLib__StalePrice();
        // code...
    }

```

In a scenario where Chainlink becomes non-operational for the TIMEOUT period (3 hours) and then recovers, block validators can use their little freedom in setting the `block.timestamp` value to their advantage. By adjusting this timestamp, validators can force a transaction to fail, especially if it's a liquidation call. They can then revert this transaction intentionally and reissue the liquidation under their address, effectively pocketing the collateral.

They would make the if statement false making with a slight value in the timestamp the secondsSince value be greater than 3 hours.

The `staleCheckLatestRoundData()` function is invoked from 15 different functions in the DSCEngine.SOL contract, indicating multiple potential vectors for exploitation in this hypothetical scenario:

```
liquidate(address , address , uint256 )external

redeemCollateralForDsc(address, uint256, uint256) external

redeemCollateral(address, uint256)external

burnDsc(uint256) external

depositCollateralAndMintDsc( address, uint256, uint256) external

mintDsc(uint256) public


revertIfHealthFactorIsBroken(address ) internal view

_healthFactor(address ) private view returns (uint256)

getHealthFactor(address ) external view returns (uint256)

getAccountInformation(address ) external view returns (uint256 , uint256 )

_getAccountInformation(address ) private view returns (uint256, uint256 )

getAccountCollateralValue(address ) public view returns (uint256 )

 getTokenAmountFromUsd(address , uint256 ) public view returns (uint256)

getUsdValue( address, uint256 ) external view returns (uint256)

 _getUsdValue(address , uint256 ) private view returns (uint256)
```

> Note ⚠️: Due to the time limitations and the very unlikely nature of this scenarios, I didn't spend time finding solutions.
