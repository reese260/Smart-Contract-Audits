# Dria - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect variance calculation in `Statistics` completely breaks protocol due to underflow](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Users can submit malicious responses and validations due to unrestricted registry, lack of POW difficulty, and lack of punishment for dishonest behavior](#M-01)
    - ### [M-02. `withdrawPlatformFees` withdraws the entire contract balance which can include fees transferred into the contract](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Swan

### Dates: Oct 25th, 2024 - Nov 1st, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-10-swan-dria)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 2
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect variance calculation in `Statistics` completely breaks protocol due to underflow            



## Summary

The standard deviation calculation in `Statistics` will almost always revert from underflow due to incorrect variance calculation.

## Vulnerability Details

Once all validations are complete in `LLMOracleCoordinator` for a given task, the validations are finalized  in [finalizeValidation](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L323C3-L323C58) and a standard deviation of all scores is calculated. There is an issue however with the way variance is calculated. The [variance](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/libraries/Statistics.sol#L16C5-L27C1) calculation only account for scores above the mean and not below the mean. Anytime a score is below the mean, the following line will revert due to underflow.

```Solidity
uint256 diff = data[i] - mean;
```

## POC

You can modify any test to include different score values and you will receive an error. Currently all tests use the same score for all scores in the array. This is the only instance this error wouldn't occur.

```Solidity
describe("with validation of different scores", function () {
    const [numGenerations, numValidations] = [2, 2];
    // Make array with two different scores
    const scores = [parseEther("0.2"), parseEther("0.3")];
    let generatorAllowancesBefore: bigint[];
    let validatorAllowancesBefore: bigint[];

    this.beforeAll(async () => {
      taskId++;

      generatorAllowancesBefore = await Promise.all(
        generators.map((g) => token.allowance(coordinatorAddress, g.address))
      );
      validatorAllowancesBefore = await Promise.all(
        validators.map((v) => token.allowance(coordinatorAddress, v.address))
      );
    });

    it("should validate with different scores", async function () {
      await safeRequest(coordinator, token, requester, taskId, input, models, {
        difficulty,
        numGenerations,
        numValidations,
      });

      for (let i = 0; i < numGenerations; i++) {
        await safeRespond(coordinator, generators[i], output, metadata, taskId, BigInt(i));
      }

      // First validator submits different scores
      await safeValidate(coordinator, validators[0], scores, metadata, taskId, 0n);

      // Second validator does same validation
      await safeValidate(coordinator, validators[1], scores, metadata, taskId, 1n);

    });
  });
```

## Impact

Critical- this is completely protocol breaking. All tests suites currently use the same value for all scores. That is the only instance this will not fail. It will almost always be the case there will be some variance in the scores

## Tools Used

Manual Review

## Recommendations

Account for both sides of variance

```Solidity
function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
    mean = avg(data);
    uint256 sum = 0;
    for (uint256 i = 0; i < data.length; i++) {
        // Safe difference calculation
        uint256 diff;
        if (data[i] >= mean) {
            diff = data[i] - mean;
        } else {
            diff = mean - data[i];
        }
        sum += diff * diff;
    }
    ans = sum / data.length;
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Users can submit malicious responses and validations due to unrestricted registry, lack of POW difficulty, and lack of punishment for dishonest behavior            



## Summary

Any EOA could sign up to be a validator or generator in  the registry. This means that malicious users can submit spam validations and responses just to claim rewards. There is currently no system set in place to discourage dishonest behavior in the protocol as well. The difficulty for generating valid nonces for POW is also very low allowing a larger number of users to potentially place this attack.

## Vulnerability Details

The issue starts in the [LLMOracleRegistry](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleRegistry.sol#L91C5-L111C6) contract due to the fact that any user can sign up to be any kind of oracle.

```Solidity
 function register(LLMOracleKind kind) public {
        uint256 amount = getStakeAmount(kind);

        // ensure the user is not already registered
        if (isRegistered(msg.sender, kind)) {
            revert AlreadyRegistered(msg.sender);
        }

        // ensure the user has enough allowance to stake
        if (token.allowance(msg.sender, address(this)) < amount) {
            revert InsufficientFunds();
        }
        token.transferFrom(msg.sender, address(this), amount);

        // register the user
        registrations[msg.sender][kind] = amount;
        emit Registered(msg.sender, kind);
    }
```

Once they are register, they can call either [respond](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L207C5-L207C100) or [validate](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L260) without performing any of the necessary generations or validations needed for the protocol to work. This will take up space in the validation and responses array for actual oracle nodes who will provide honest information. It can also potentially reward these malicious actors for not performing any of the necessary work.

The current maximum difficulty for computing the correct nonce is also set as 10 in the market parameters. When we look at the POW validation we see it uses the following check:

```Solidity
 function assertValidNonce(uint256 taskId, TaskRequest storage task, uint256 nonce) internal view {
        bytes memory message = abi.encodePacked(taskId, task.input, task.requester, msg.sender, nonce);
        if (uint256(keccak256(message)) > type(uint256).max >> uint256(task.parameters.difficulty)) {
            revert InvalidNonce(taskId, nonce);
        }
    }
```

Lets take the maximum of 10 for example. To create a message less than max uint256 shifted by 10 bits, the probability of computing the correct nonce would be 1/2^10 which would be 1/1024. This is very easy to compute and would take most computers milliseconds to calculate. Below is an example using the existing computation in the `index.ts` file that includes helper functions for the tests

```Solidity
it("mine nonce", async function () {
      const taskRequest = await coordinator.requests(taskId);
      const validator = generators[0];
      const nonce = await mineNonce(BigInt(10), taskRequest.requester, validator.address, taskRequest.input, taskId)

      console.log(nonce);
  })
```

It only takes milliseconds to complete

```Solidity
✔ mine nonce (38ms)
```

## Impact

Malicious users could potentially steal rewards from honest validators and generators. It would be especially be easy for generators as no upper bound is enforced for the standard deviation of scores.

```Solidity
(uint256 stddev, uint256 mean) = Statistics.stddev(generationScores);
        for (uint256 g_i = 0; g_i < task.parameters.numGenerations; g_i++) {
            // ignore lower outliers
            if (generationScores[g_i] >= mean - generationDeviationFactor * stddev) {
                _increaseAllowance(responses[taskId][g_i].responder, task.generatorFee);
            }
        }
```

It could also render requests useless and buyer agents would not receive valid responses of assets to buy.

## Tools Used

Manual Review

## Recommendations

* Add whitelist for node oracles
* Increase maximum difficulty and minimum difficulty for POW
* Incentivize honest behavior and punish dishonest behaviot (ex. slashing staking amount)

## <a id='M-02'></a>M-02. `withdrawPlatformFees` withdraws the entire contract balance which can include fees transferred into the contract            



## Summary

The [withdrawPlatformFees](https://github.com/Cyfrin/2024-10-swan-dria/blob/c3f6f027ed51dd31f60b224506de2bc847243eb7/contracts/llm/LLMOracleCoordinator.sol#L374C5-L377C6) function will withdraw all of the fee tokens in the contract. This can include fees for generators and validators.

## Vulnerability Details

When the owner of the contract calls `withdrawPlatformFees`, it withdraws the entire balance of the contract. The problem with this is that there could be fees still in the contract for generators and validators.

```Solidity
/// @notice Withdraw the platform fees & along with remaining fees within the contract.
    function withdrawPlatformFees() public onlyOwner {
        feeToken.transfer(owner(), feeToken.balanceOf(address(this)));
    }
```

Rewards for generators and validators are granted through an allowance. Even if they were to automate the process and auto withdrawal on token approval, there is still the possibility the withdraw function could contain rewards.\\

```Solidity
/// Increases the allowance by setting the approval to the sum of the current allowance and the additional amount.
    /// @param spender spender address
    /// @param amount additional amount of allowance
    function _increaseAllowance(address spender, uint256 amount) internal {
        feeToken.approve(spender, feeToken.allowance(address(this), spender) + amount);
    }
```

If this function were to be called while there are still fee tokens, generators and validators may have allowances for an amount not available in the contract.

## Impact

Generators and validators can lose fees

## Tools Used

Manual Review

## Recommendations

There needs to be a separate variable to track how much protocol fees have been accrued that are eligible to claim.

```Solidity
function withdrawPlatformFees() public onlyOwner {
        feeToken.transfer(owner(), feeToken.balanceOf(address(this) - totalFeesToClaim);
}
```





