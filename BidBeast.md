# Bid Beasts - Findings Report

# Table of contents

- ## High Risk Findings
    - ### [H-01. Unrestricted access to burn() function in BidBeast_NFT_ERC721.sol](#H-01)
- ## Medium Risk Findings
    - ### [M-01.  Bid-Based Time Extensions Create Never-Ending Auctions](#M-01)
- ## Low Risk Findings
    - ### [L-01.  Incorrect Event Emission ](#L-01)


# Summary

### Sponsor: First Flight #49

### Dates: Sep 25th, 2025 - Oct 2nd, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-09-bid-beasts)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Unrestricted access to burn() function in BidBeast_NFT_ERC721.sol            



# Burn functin has no restrictions on who can burn NFT

## Description

* In the `BidBeast_NFT_ERC721.sol` contract, there are two main functions: the `mint()` and the `burn()`. To be able to to mint and burn, the user must be the owner which is shown in the `mint()` function but not in the `burn()` function.Explain the specific issue or problem in one or more sentences

 

* The burn() function allows any user to burn any NFT token without ownership verification. This violates the fundamental principle of NFT ownership where only the token owner or approved addresses should be able to burn tokens.

```Solidity

  @> function burn(uint256 _tokenId) public {
        _burn(_tokenId);
        emit BidBeastsBurn(msg.sender, _tokenId);
    }
```

## Risk

**Likelihood**:

* This can occurs anytime a user or seller mint an NFT 

**Impact**:

*  Complete loss of digital assets for legitimate owners

*  Any user can permanently destroy any NFT in the collection Impact 2

## Proof of Concept

```markdown
Attack Scenario:
1. User A legitimately owns tokenID 5 
2. User B (Attacker) calls burns(5) without any restrictions
3. tokenID 5 is permanently burned and removed from User A's wallet
4. User A is in complete loss of the NFT
```

```Solidity
    address public constant MALICIOUS_USER = address(0x5);
    function test_anyoneCanBurnNFT() public {
            //Legitimate SELLER owns NFT ID = 0
            _mintNFT();
            assertEq(nft.ownerOf(0), SELLER)

            //Malicious USER decide to burn SELLER Assets...
            vm.prank(MALICIOUS_USER);
            nft.burn(0);

            //Token has been burnt ...
            vm.expectRevert();
            nft.ownerOf(0);
        }
```

## Recommended Mitigation

```markdown
 I recommand adding a Custom ownership check
```

```diff
function burn(uint256 _tokenId) public {
+       address owner = ownerOf(_tokenId);
+       require(owner == msg.sender, "Not owner or approved");
        _burn(_tokenId);
        emit BidBeastsBurn(msg.sender, _tokenId);
    }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01.  Bid-Based Time Extensions Create Never-Ending Auctions            



# Time extension causing a never ending of the auction

## Description

* The auction time extension logic incorrectly adds fixed duration to the current auction end time instead of setting a minimum end time from the current block timestamp. This creates a scenario where auctions can be extended indefinitely, preventing them from ever ending and making the `settleAuction` function unusable in placeBid() function.

* Each bid extends the auction by fixed duration from the current end time, not from current time. This creates compounding extensions that push the end time infinitely into the future.

```Solidity

require(msg.sender != previousBidder, "Already highest bidder"); // @Not_a_bug Isn't the requirement coming at the wrong time ??
        emit AuctionSettled(tokenId, msg.sender, listing.seller, msg.value); // @solve Wrong amount specified

        // --- Regular Bidding Logic ---
        //Code here ...

            uint256 timeLeft = 0;
            if (listing.auctionEnd > block.timestamp) {
                timeLeft = listing.auctionEnd - block.timestamp;
            }
    @>        if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
                listing.auctionEnd =
                    listing.auctionEnd +
    @>                S_AUCTION_EXTENSION_DURATION; 
                emit AuctionExtended(tokenId, listing.auctionEnd);
            }
        }
```

**Likelihood**:

* Every bid placed within the final 15 minutes of an auction extends the duration by a full 15 minutes from the current end time

**Impact**:

* settleAunction function is unusable

* infinite bidding wars causing wasted gas on unwinnable auctionsImpact 2

## PoC

Add this to the test file and run it:

```Solidity
    function test_ExcessiveAuctionDuration() public {
        _mintNFT();
        _listNFT();

        uint256 extensionDuration = market.S_AUCTION_EXTENSION_DURATION();

        console.log("=== EXCESSIVE AUCTION DURATION BUG ===");
        console.log(
            "Auction extension duration:",
            extensionDuration,
            "seconds (15 minutes)"
        );

        // First bid - starts the auction
        vm.prank(BIDDER_1);
        market.placeBid{value: 2 ether}(TOKEN_ID);

        BidBeastsNFTMarket.Listing memory listing = market.getListing(TOKEN_ID);
        uint256 firstAuctionEnd = listing.auctionEnd;
        console.log("First bid - Auction ends at:", firstAuctionEnd);
        console.log(
            "Time until end:",
            firstAuctionEnd - block.timestamp,
            "seconds"
        );

        // Second bid: 1 minute before end → SHOULD extend
        vm.warp(firstAuctionEnd - 60); // 60 seconds before end
        vm.prank(BIDDER_2);
        market.placeBid{value: 3 ether}(TOKEN_ID);

        listing = market.getListing(TOKEN_ID);
        uint256 secondAuctionEnd = listing.auctionEnd;
        console.log(
            "Second bid (1min before end) - New end:",
            secondAuctionEnd
        );
        console.log(
            "Time added:",
            secondAuctionEnd - firstAuctionEnd,
            "seconds"
        );
        console.log(
            "Total auction duration so far:",
            secondAuctionEnd - block.timestamp,
            "seconds"
        );

        // BUG DEMONSTRATION: Third bid with 10 MINUTES remaining still extends!
        vm.warp(secondAuctionEnd - 600); // 600 seconds = 10 MINUTES before end
        uint256 timeBeforeBid = block.timestamp;
        vm.prank(BIDDER_1);
        market.placeBid{value: 4 ether}(TOKEN_ID);

        listing = market.getListing(TOKEN_ID);
        uint256 thirdAuctionEnd = listing.auctionEnd;

        // Verify auction is still ongoing (as expected with bad time logic)
        vm.expectRevert("Auction has not ended");
        market.settleAuction(TOKEN_ID);

        console.log(
            "settleAuction reverts - auction still active as long as there will be a new bidder... no limit"
        );
    }
```



## Recommended Mitigation


```markdown
Implement a maximum auction deadline that prevents any auction from extending beyond 24 hours from its start time, ensuring all auctions eventually end.

```

```markdown
// Add maximum auction duration constant
+ uint256 public constant S_MAX_AUCTION_DURATION = 24 hours; //Or more 

// In placeBid function, modify the time extension logic:
} else {
  // code here ....
    if (timeLeft < S_AUCTION_EXTENSION_DURATION) {
        // Calculate new end time with maximum duration limit
+       uint256 newEndTime = listing.auctionEnd + S_AUCTION_EXTENSION_DURATION;
+       uint256 maxAllowedEnd = listing.startTime + S_MAX_AUCTION_DURATION;
        
        // Enforce maximum auction duration
+       if (newEndTime > maxAllowedEnd) {
            listing.auctionEnd = maxAllowedEnd;
+       } else {
            listing.auctionEnd = newEndTime;
+       }
        
        emit AuctionExtended(tokenId, listing.auctionEnd);
    }
}
```


# Low Risk Findings

## <a id='L-01'></a>L-01.  Incorrect Event Emission             



# Event is called at wrong time 

## Description

* The `AuctionSettled` event is emitted in the wrong conditional branch. It fires during regular auction bidding instead of when buyNow purchases actually settle the auction. This creates inverted event semantics where actual auction settlements are silent while ongoing auctions incorrectly appear settled.
  Explain the specific issue or problem in one or more sentences

```Solidity
        if (listing.buyNowPrice > 0 && msg.value >= listing.buyNowPrice) {
            uint256 salePrice = listing.buyNowPrice;
            uint256 overpay = msg.value - salePrice;

            // EFFECT: set winner bid to exact sale price (keep consistent)
            bids[tokenId] = Bid(msg.sender, salePrice);
            listing.listed = false;
            if (previousBidder != address(0)) {
                _payout(previousBidder, previousBidAmount);
            }

            // NOTE: using internal finalize to do transfer/payouts. _executeSale will assume bids[tokenId] is the final winner.
            _executeSale(tokenId);

            // Refund overpay (if any) to buyer
            if (overpay > 0) {
                _payout(msg.sender, overpay);
            }

            return;
        }

        require(msg.sender != previousBidder, "Already highest bidder"); 
    @>  emit AuctionSettled(tokenId, msg.sender, listing.seller, msg.value);
// The rest ....
```

## Risk

**Likelihood**:

*  The `AuctionSettled` event emits during every regular bid placement, occurring 100% of the time for non-buyNow bids

*  External systems and users receive false "auction ended" notifications on every single bid, creating consistent misinformation

**Impact**:

* BuyNow settlements are invisible to external systems

* Regular bids falsely appear as settlements

* Buyers: Confused about auction statesProof of Concept


## Proof Of Code

```markdown
Scenario1: Buy Now Logic 
    1. User pays buyNowPrice (5 ETH)
    2. _executeSale() transfers NFT to buyer
    3. Auction is ACTUALLY settled
    4. NO AuctionSettled event emitted
    5. External systems don't know auction ended

Scenario2: Regular Bid
    1. User places normal bid (1 ETH)
    2. emit AuctionSettled() → "Auction ended!"  //While it's still on 
    3. But auction continues accepting more bids
    4. Time gets extended
    5. External systems see impossible state
```

## Recommended Mitigation

```markdown
Delete `AuctionSettled` emitted at it's actaul location
```

```diff
            // --- Buy Now Logic ---

        if (listing.buyNowPrice > 0 && msg.value >= listing.buyNowPrice) {
            uint256 salePrice = listing.buyNowPrice;
            uint256 overpay = msg.value - salePrice;

            // EFFECT: set winner bid to exact sale price (keep consistent)
            bids[tokenId] = Bid(msg.sender, salePrice);
            listing.listed = false;
            if (previousBidder != address(0)) {
                _payout(previousBidder, previousBidAmount);
            }

            // NOTE: using internal finalize to do transfer/payouts. _executeSale will assume bids[tokenId] is the final winner.
            _executeSale(tokenId);

            // Refund overpay (if any) to buyer
            if (overpay > 0) {
                _payout(msg.sender, overpay);
            }
            
            return;
        }

        require(msg.sender != previousBidder, "Already highest bidder");
-       emit AuctionSettled(tokenId, msg.sender, listing.seller, msg.value);
```



