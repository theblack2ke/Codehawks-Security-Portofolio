# Eggstravaganza - Findings Report

# Table of contents

- ## High Risk Findings
    - ### [H-01. Weak randomness in `EggHuntGame::searchForEgg()` allows users to influence or predict the random number or if they are going to win. ](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Unsafe `ERC721::_mint()`](#M-01)


# Contest Summary

### Sponsor: First Flight #37

### Dates: Apr 3rd, 2025 - Apr 10th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-04-eggstravaganza)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 0


# High Risk Findings

## Weak randomness in `EggHuntGame::searchForEgg()` allows users to influence or predict the random number or if they are going to win.             



## Summary


Hashing `msg.sender`, `block.timestamp`, and `block.prevrandao` together creates a predictable number.A predictable number is not a good random number.
This implementation remains vulnerable to manipulation in production environments.



# Impact

Attackers can predict future "random" numbers by monitoring public blockchain data and time transactions to influence outcomes.


## Scenario


1. Validators can know ahead of time `block.timestamp` and `block.difficulty` and use that to predict when/how to participate. 
2. User can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner!
3. Users can revert their `searchForEgg()` transaction if they don't like the outcome or if they know they will fail.


## Recommendations


There are some better way to generate random numbers like:
 The chainlink VRF for randomness implementation. [See in https://docs.chain.link/vrf]

    
# Medium Risk Findings

## Unsafe `ERC721::_mint()`            



# Summary


Using `ERC721::_mint()` can mint ERC721 tokens to addresses which don't support ERC721 tokens. 


# Impact


Using `_mint()` instead of `_safeMint()` can permanently lock NFTs in non-ERC721-compatible contracts, leading to irreversible token loss.
This means if  the `EggstravaganzaNFT` doesn't approve before calling the `mint()` it will revert.




# Recommendations&#x20;


Use `_safeMint()` instead of `_mint()` for ERC721 tokens.

Made by 2ke.





