# Fjord Token Staking - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Auction tokens will be permanently locked in `FjordAuctionFactory` when an auction ends with no bidders](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Fjord

### Dates: Aug 20th, 2024 - Aug 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-fjord)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 0



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Auction tokens will be permanently locked in `FjordAuctionFactory` when an auction ends with no bidders            



## Summary
Auction tokens will be permenantly locked inside the `FjordAuctionFactory` contract when `FjordAuction::auctionEnd` is called for auctions with no bids.

## Vulnerability Details
After an auction ends, the `auctionEnd` function is called to either transfer the locked tokens to the contract owner or calculate the claimable amount for each participant of the auction. In the case there are no bidders, all tokens in the contract will be sent to the `FjordAuctionFactory` contract because it is set as the owner upon deployment.

When deploying a new `FjordAuction`, `FjordAuctionFactory::createAuction` is called with the following parameters.

https://github.com/Cyfrin/2024-08-fjord/blob/3a78c456b39c799f64c9c2510992584ada3516a0/src/FjordAuctionFactory.sol#L58C9-L60C11
```Solidity
 address auctionAddress = address(
            new FjordAuction{ salt: salt }(fjordPoints, auctionToken, biddingTime, totalTokens)
        );
```
In `FjordAuction`, msg.sender is set as the owner of the contract in the constructor. Because `FjordAuctionFactory` deployed the contract, it will be set as the owner.

https://github.com/Cyfrin/2024-08-fjord/blob/3a78c456b39c799f64c9c2510992584ada3516a0/src/FjordAuction.sol#L134
```Solidity
owner = msg.sender;
```
When `auctionEnd` is called for an auction with no bidders, all tokens are transfered to the owner. The owner here would be the `FjordAuctionFactory` contract.

https://github.com/Cyfrin/2024-08-fjord/blob/3a78c456b39c799f64c9c2510992584ada3516a0/src/FjordAuction.sol#L192C1-L195C10
```Solidity
       if (totalBids == 0) {
            auctionToken.transfer(owner, totalTokens);
            return;
        }
```
These tokens would then be permanentley locked inside the contract as there is no function inside the `FjordAuctionFactory` to retireve them.


## Impact
Tokens transfered into the `FjordAuctionFactory` will be locked permanently.

## Tools Used
Manual Review

## POC
Test to verify tokens are sent to `FjordAuctionFactory` for auctions with no bidders

I first created a `FjordAuctionFactory` contract instance in `auction.t.sol`
```Solidity
function setUp() public {
        fjordPoints = new ERC20BurnableMock("FjordPoints", "fjoPTS");
        auctionToken = new ERC20BurnableMock("AuctionToken", "AUCT");
        auction =
            new FjordAuction(address(fjordPoints), address(auctionToken), biddingTime, totalTokens);
        auctionFactory = new AuctionFactory(address(fjordPoints));

        deal(address(auctionToken), address(auction), totalTokens);
    }
```

Test function that uses a utility function to derive the address of the auction contract deployed by the factory. It then verifies funds are sent to the factory contract.
```Solidity
    function testFactoryIsOwnerOfAuction() public {
        IERC20(auctionToken).approve(address(auctionFactory), totalTokens);
        deal(address(auctionToken), address(auctionFactory.owner()), totalTokens);

        vm.prank(auctionFactory.owner());
        auctionFactory.createAuction(address(auctionToken), biddingTime, totalTokens, salt);

        // Calculate the address of the deployed auction contract
        address auctionAddress = computeAuctionAddress();

        FjordAuction auction = FjordAuction(auctionAddress);

        // Check that the factory contract is the owner of the auction
        assertEq(auction.owner(), address(auctionFactory), "Factory is not the owner of the auction");

        skip(biddingTime);

        uint256 balBefore = auctionToken.balanceOf(address(auctionFactory));
        auction.auctionEnd();
        uint256 balAfter = auctionToken.balanceOf(address(auctionFactory));

        assertEq(auction.ended(), true);
        assertEq(balAfter - balBefore, totalTokens);
    }

    // Utility function to compute the address of the deployed auction contract using create2
    function computeAuctionAddress() internal view returns (address) {
        bytes memory bytecode = abi.encodePacked(
            type(FjordAuction).creationCode,
            abi.encode(fjordPoints, auctionToken, biddingTime, totalTokens)
        );

        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(auctionFactory),
                salt,
                keccak256(bytecode)
            )
        );

        return address(uint160(uint256(hash)));
    }
```

## Recommendations
If it is the intention that these tokens be transfered to the factory contract then a withdraw function should be created that is only called by the owner. Otherwise consider adding an owner parameter in the auction constructor to specify who the owner should be, rather than just using msg.sender.




