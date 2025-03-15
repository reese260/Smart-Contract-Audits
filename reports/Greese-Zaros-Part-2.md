# Part 2 - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Trader receives less credit due to deleveraging factor being applied twice](#H-01)
    - ### [H-02. User will lose staking rewards if they stake again without claiming initial rewards](#H-02)

- ## Low Risk Findings
    - ### [L-01. Users could potentially use their own referral code](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Zaros

### Dates: Jan 20th, 2025 - Feb 6th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-01-zaros-part-2)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 0
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Trader receives less credit due to deleveraging factor being applied twice            



## Summary

When an order is filled through [\_fillOrder](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/perpetuals/branches/SettlementBranch.sol#L356), if the trader has a positive pnl they receive credit and receive settlement tokens from the market making engine. The problem is the margin to add is deducted by the delevergaing factor twice leading to less settlement tokens minted to the trader than intended. Once through [getAdjustedProfitForMarketId](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/CreditDelegationBranch.sol#L129) and again through [withdrawUsdTokenFromMarket](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/CreditDelegationBranch.sol#L246)

## Vulnerability Details

A market order keeper intends to fill an order. When [\_fillOrder](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/perpetuals/branches/SettlementBranch.sol#L356) is called it checks if the trader's pnl is currently positive. If it is they are credited settlement tokens through the market making engine. The margin to add needs to be adjusted based on the deleveraging factor. That is done through the call to `getAdjustedProfitForMarketId`. The amount returned is then passed into `withdrawUsdTokenFromMarket` to mint the corresponding amount of settlement tokens.

```Solidity
// if trader's old position had positive pnl then credit that to the trader
        if (ctx.pnlUsdX18.gt(SD59x18_ZERO)) {
            IMarketMakingEngine marketMakingEngine = IMarketMakingEngine(perpsEngineConfiguration.marketMakingEngine);

            ctx.marginToAddX18 =
                marketMakingEngine.getAdjustedProfitForMarketId(marketId, ctx.pnlUsdX18.intoUD60x18().intoUint256());

            tradingAccount.deposit(perpsEngineConfiguration.usdToken, ctx.marginToAddX18);

            // mint settlement tokens credited to trader; tokens are minted to
            // address(this) since they have been credited to the trader's margin
            marketMakingEngine.withdrawUsdTokenFromMarket(marketId, ctx.marginToAddX18.intoUint256());
        }
```

The problem is that both of these functions are reducing the amount by the deleveraging factor.
<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/CreditDelegationBranch.sol#L154C9-L160C10>
<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/CreditDelegationBranch.sol#L283C9-L292C64>

```Solidity
// we don't need to add `profitUsd` as it's assumed to be part of the total debt
        // NOTE: If we don't return the adjusted profit in this if branch, we assume marketTotalDebtUsdX18 is positive
        if (market.isAutoDeleverageTriggered(delegatedCreditUsdX18, marketTotalDebtUsdX18)) {
            // if the market's auto deleverage system is triggered, it assumes marketTotalDebtUsdX18 > 0
            adjustedProfitUsdX18 =
                market.getAutoDeleverageFactor(delegatedCreditUsdX18, marketTotalDebtUsdX18).mul(adjustedProfitUsdX18);
        }
```

```Solidity
// now we realize the added usd debt of the market
        // note: USD Token is assumed to be 1:1 with the system's usd accounting
        if (market.isAutoDeleverageTriggered(delegatedCreditUsdX18, marketTotalDebtUsdX18)) {
            // if the market is in the ADL state, it reduces the requested USD
            // Token amount by multiplying it by the ADL factor, which must be < 1
            UD60x18 adjustedUsdTokenToMintX18 =
                market.getAutoDeleverageFactor(delegatedCreditUsdX18, marketTotalDebtUsdX18).mul(amountX18);

            amountToMint = adjustedUsdTokenToMintX18.intoUint256();
            market.updateNetUsdTokenIssuance(adjustedUsdTokenToMintX18.intoSD59x18());
```

This greatly reduces the intended amount of credit dedicated to this user.

## Impact
Loss of user credit
## Tools Used
Manual Review
## Recommendations
There is no need to check the deleveraging factor when we mint the tokens in `withdrawUsdTokenFromMarket` as it is already accounted for.
## <a id='H-02'></a>H-02. User will lose staking rewards if they stake again without claiming initial rewards            



## Summary

Users can stake their index tokens to earn a proportion of fees through the [stake](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L380) function in `VaultRouterBranch`. The amount of shares staked is set in the `Distribution` logic when [setActorShares](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Distribution.sol#L44) is called. It also keeps track of the current value per share at the time of staking. An issue occurs when a user attempts to stake again and this variable is updated without the user ever claiming their fees between the last and latest update.

## Vulnerability Details

A user stakes and their shares are set in `setActorShares` where it also updates the last value share through [\_updateLastValuePerShare](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Distribution.sol#L93). This is where the current value is set to `lastValuePerShare` for future calculations.

```Solidity
function _updateLastValuePerShare(
        Data storage self,
        Actor storage actor,
        UD60x18 newActorShares
    )
        private
        returns (SD59x18 valueChange)
    {
        valueChange = _getActorValueChange(self, actor);

        actor.lastValuePerShare = newActorShares.eq(UD60x18_ZERO) ? int256(0) : self.valuePerShare;
    }
```

This way if the value per share were to increase, the staker would be eligible for the difference. This delta value calculation is done through [\_getActorValueChange](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Distribution.sol#L106). A problem arises if a user stakes a second time without claiming their current rewards. `lastValuePerShare` will be updated again and when the delta calculation is made the next time a user were to claim fees, they would only receive the difference from the second time they staked.

```Solidity
function _getActorValueChange(
        Data storage self,
        Actor storage actor
    )
        private
        view
        returns (SD59x18 valueChange)
    {
        SD59x18 deltaValuePerShare = sd59x18(self.valuePerShare).sub(sd59x18(actor.lastValuePerShare));
        valueChange = deltaValuePerShare.mul(ud60x18(actor.shares).intoSD59x18());
    }
```
The accumulate rewards function doesnt actually accumulate the rewards for the user up to this point. They would need to call [claimFees](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/FeeDistributionBranch.sol#L284) to get rewards up to this point.
## Impact
User can lose fees
## Tools Used
Manual Review
## Recommendations
Add logic for the user to claim before staking again or require the user to claim like in `unstake`
    


# Low Risk Findings

## <a id='L-01'></a>L-01. Users could potentially use their own referral code            



## Summary

[registerReferral](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/39e33b2f6b3890573bb1affc41a7e520277ceb2c/src/referral/Referral.sol#L155) is called from two different places. In `VaultRouterBranch` when a user initially [deposits](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/39e33b2f6b3890573bb1affc41a7e520277ceb2c/src/market-making/branches/VaultRouterBranch.sol#L265) and in `TradingAccountBranch` when a user creates a new trading account through [createTradingAccount](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/39e33b2f6b3890573bb1affc41a7e520277ceb2c/src/perpetuals/branches/TradingAccountBranch.sol#L225). The problem is `registerReferral` always uses `referrerCode` to decode the user's address however that will not always be the case as seen in `createTradingAccount` which uses the account id. This allows a user to bypass that validation that prevents them from using their own referral code.

## Vulnerability Details

in `registerReferral` there is a check that the decoded referral code does not equal the referrers address. The problem is that the referral code may not always be the user's address.
<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/39e33b2f6b3890573bb1affc41a7e520277ceb2c/src/referral/Referral.sol#L189C16-L192C18>

```Solidity
// revert if the referral code decoded is the same as the referrer address
                if (referrerAddress == abi.decode(referralCode, (address))) {
                    revert InvalidReferralCode();
                }
```

In the case of `createTradingAccount` it passes in the user's trading account id. So the user would be able to encode what their referral id would be based on the current index and pass it into this function.
<https://github.com/Cyfrin/2025-01-zaros-part-2/blob/39e33b2f6b3890573bb1affc41a7e520277ceb2c/src/perpetuals/branches/TradingAccountBranch.sol#L260C9-L264C10>

```Solidity
 if (referralCode.length != 0) {
            referralModule.registerReferral(
                abi.encode(tradingAccountId), msg.sender, referralCode, isCustomReferralCode
            );
        }
```

## POC

Add this test to `createTradingAccount.t.sol` and run forge test --match-test test\_UserCanUseOwnReferralCode

```Solidity
function test_UserCanUseOwnReferralCode()
        external
        givenTheTradingAccountTokenIsSet
        whenTheUserHasAReferralCode
        whenTheReferralCodeIsNotCustom
    {
        changePrank({ msgSender: users.naruto.account });

        uint128 expectedTradingAccountId = 1;

        bytes memory referralCode = abi.encode(expectedTradingAccountId);

        address referralModule = perpsEngine.workaround_getReferralModule();
        vm.expectEmit({ emitter: referralModule });
        emit Referral.LogReferralSet(
            address(perpsEngine), abi.encode(expectedTradingAccountId), users.naruto.account, referralCode, false
        );
        perpsEngine.createTradingAccount(referralCode, false);
    }
```

## Impact

User's can use their own referral code

## Tools Used

Manual Review

## Recommendations

Add more robust checks to see if a user is using their own code. In the case of creating a trading account where a user could create multiple accounts it wouldn't be able to use msg.sender to encode the referral code.



