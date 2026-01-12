# Audit Contest Summary: [Brivault]

This document summarizes the results and key findings from my participation in the `Codehawk` audit contest, held for the `Brivault` codebase.

- [Audit Contest Summary: \[Brivault\]](#audit-contest-summary-brivault)
  - [1. Overview \& Project Details](#1-overview--project-details)
  - [2. Contest Performance Summary](#2-contest-performance-summary)
- [Detailed Audit Findings (Valid Submissions)](#detailed-audit-findings-valid-submissions)
  - [High](#high)
    - [\[H-1\] Faulty `stakedAsset` Tracking in `BriVault::deposit` function Locks Part of User Assets](#h-1-faulty-stakedasset-tracking-in-brivaultdeposit-function-locks-part-of-user-assets)
    - [\[H-2\] `BriVault::joinEvent` allows multiple joins; incorrect share calculation may prevent user withdrawals or cause withdrawal amount to be less than deposit](#h-2-brivaultjoinevent-allows-multiple-joins-incorrect-share-calculation-may-prevent-user-withdrawals-or-cause-withdrawal-amount-to-be-less-than-deposit)
    - [\[H-3\] `Brivault` overrides `deposit` instead of `_deposit` function, causing shares to be minted to `msg.sender` rather than the intended `receiver` and bypassing `ERC4626’s:: _convertToShares`, which leads to inaccurate share calculations.](#h-3-brivault-overrides-deposit-instead-of-_deposit-function-causing-shares-to-be-minted-to-msgsender-rather-than-the-intended-receiver-and-bypassing-erc4626s-_converttoshares-which-leads-to-inaccurate-share-calculations)
  - [Medium](#medium)
    - [\[M-1\] Iterating Over All Users in `BriVault::_getWinnerShares ()`  Allows Potential DoS](#m-1-iterating-over-all-users-in-brivault_getwinnershares---allows-potential-dos)



## 1. Overview & Project Details

| Field               | Detail                                                                                 |
| :------------------ | :------------------------------------------------------------------------------------- |
| **Project Audited** | **Brivault**                                                                           |
| **Contest Period**  | 2025/11/06 – 2025/11/13                                                                |
| **Codebase Link**   | [2025-11-brivault](https://github.com/CodeHawks-Contests/2025-11-brivault)             |
| **Final Report**    | [Final Report](https://codehawks.cyfrin.io/c/2025-11-brivault/results?page=1&t=report) |


---

## 2. Contest Performance Summary

This section provides a statistical breakdown of my submission compared to the final results.
| Category            | My Valid Findings |   Total in Contest   |
| :------------------ | :---------------: | :------------------: |
| **High Severity**   |       **3**       |          6           |
| **Medium Severity** |       **1**       |          2           |
| **Medium Severity** |       **0**       |          4           |
| **Overall Rank**    |     **29th**      | **115 Participants** |


---

# Detailed Audit Findings (Valid Submissions)

## High

### [H-1] Faulty `stakedAsset` Tracking in `BriVault::deposit` function Locks Part of User Assets

**Description:** In the `deposit` function, each time a user deposits, the contract updates `stakedAsset[receiver]` with the new `assets`, instead of accumulating it.As a result, the previous deposited funds are overwritten and no longer tracked.When a user makes a second deposit, the first deposit becomes effectively locked.If the user deposits a third time, both the first and second deposits are no longer accounted for.

**Impact:** Consequently, when the user attempts to withdraw, they can only withdraw the amount from their most recent deposit, while all previous deposits remain unrecoverable.

**Proof of Concept:**
1. The user approves 10 ETH for the BriVault contract.
2. The user makes two deposits of 5 ETH each.
3. The user calls cancelParticipation() to withdraw all previously deposited funds, excluding the participation fee.

Place the following code in briVault.t.sol.
<details> 

<summary>Proof Of Code</summary>


``` javascript

   function testCannotWithdrawAllTheDeposit() public {
        vm.startPrank(user1);
        // balance of user before deposit should be 20eth
        console.log("user balance before withdraw:", mockToken.balanceOf(user1));
        mockToken.approve(address(briVault), 10 ether);
        // deposite 2 times, each 5 ether 
        briVault.deposit(5 ether, user1);
        briVault.deposit(5 ether, user1);
        // calculate the participationFee for 1 time 5 ether
        uint256 participationFee = (5 ether * participationFeeBsp) / 10000;
        briVault.cancelParticipation();
        vm.stopPrank();
        
        uint256 BalanceExpected = 20 ether - (participationFee*2);
        uint256 BalanceAfterCancelParticipation = mockToken.balanceOf(user1);

        assert(BalanceExpected > BalanceAfterCancelParticipation);
        assertEq(BalanceAfterCancelParticipation, (15 ether-participationFee));
    }
```

</details>
After calling cancelParticipation(), the user’s balance is expected to be 20 ETH, excluding participation fees for both deposits, but the actual balance is only 15 ETH, excluding the fee for the second deposit, because the first deposit was overwritten.

**Recommended Mitigation:** Each user can either deposit once or update their existing deposit.
```diff
    function deposit(uint256 assets, address receiver) public override returns (uint256) {
        require(receiver != address(0));

        if (block.timestamp >= eventStartDate) {
            revert eventStarted();
        }

        uint256 fee = _getParticipationFee(assets);
        // charge on a percentage basis points
        if (minimumAmount + fee > assets) {
            revert lowFeeAndAmount();
        }

-       uint256 stakeAsset = assets - fee;
-       stakedAsset[receiver] = stakeAsset;
+       uint256 currentStakedAsset = stakedAsset[receiver];
+       uint256 stakeAsset = assets - fee;
+       stakedAsset[receiver] = currentStakedAsset +  stakeAsset;


```




### [H-2] `BriVault::joinEvent` allows multiple joins; incorrect share calculation may prevent user withdrawals or cause withdrawal amount to be less than deposit

**Description:** `BriVault::joinEvent` allows a user to join multiple times, and calculates `participantShares` as `balanceOf(msg.sender)`. As a result, when a user joins multiple events, their balance is repeatedly counted. In addition, `usersAddress.push(msg.sender)` causes the same user to be pushed into `usersAddress` multiple times, which leads to `totalParticipantShares` being recalculated each time and including assets from the user's previous joins.

Because `userToCountry[msg.sender]` is updated every time `joinEvent` is called, even if a user voted for the winning country, their record can be overwritten, preventing them from being able to withdraw.

```javascript
  function joinEvent(uint256 countryId) public {

      .....
        
@>      userToCountry[msg.sender] = teams[countryId];
       
@>      uint256 participantShares = balanceOf(msg.sender);
@>      userSharesToCountry[msg.sender][countryId] = participantShares;

@>      usersAddress.push(msg.sender);

        numberOfParticipants++;
@>      totalParticipantShares += participantShares;

        emit joinedEvent(msg.sender, countryId);
    }
```
The `BriVault::_getWinnerShares` function repeatedly counts the same user because the user is pushed into `usersAddress` multiple times. As a result, `totalWinnerShares` is calculated multiple times for the same user, causing it to exceed the actual amount.

```javascript
    function _getWinnerShares () internal returns (uint256) {
        for (uint256 i = 0; i < usersAddress.length; ++i){
@>          address user = usersAddress[i]; 
@>          totalWinnerShares += userSharesToCountry[user][winnerCountryId];
        }
        return totalWinnerShares;
    }

```

**Impact:** Because a user’s balance is counted multiple times, totalWinnerShares becomes inflated, causing winning users to receive less on withdrawal than their original deposit. If a user joins the event multiple times and selects the winning option, their participation may not be correctly recorded, which can prevent them from being able to withdraw.

**Proof of Concept:**

Place the following code in briVault.t.sol.

<details>
<summary>Proof Of Code</summary>

```javascript
    function testWithdrawNotWorking() public{
        vm.prank(owner);
        briVault.setCountry(countries);
        
        vm.startPrank(user1);
        // user 1 deposite and join the event choose Country 10
        mockToken.approve(address(briVault), 4 ether);
        uint256 user1sharesFirst = briVault.deposit(3 ether, user1);
        briVault.joinEvent(10);
        // user 2 join event second time choose country 14
        uint256 user1sharesSecond = briVault.deposit(1 ether, user1);
        briVault.joinEvent(14);
        vm.stopPrank();

        vm.startPrank(user2);
        mockToken.approve(address(briVault), 19 ether);
        uint256 user2Shares = briVault.deposit(19 ether, user2);
        briVault.joinEvent(10);
        uint256 balanceBeforuser2 = mockToken.balanceOf(user2);
        vm.stopPrank();

        vm.warp(eventEndDate + 1);
        vm.startPrank(owner);
        briVault.setWinner(10);
        console2.log("finalized vault balance",briVault.finalizedVaultAsset());
        vm.stopPrank();

        console2.log("total Winner Shares",briVault.totalWinnerShares());
        vm.startPrank(user1);
        vm.expectRevert(abi.encodeWithSignature("didNotWin()"));
        briVault.withdraw();
        vm.stopPrank();  

        vm.startPrank(user2);
        briVault.withdraw();
        console2.log("user2 balance",mockToken.balanceOf(user2));
        assert(mockToken.balanceOf(user2) < 19 ether);
        vm.stopPrank();    

    }
```
The result:

```javascript
   finalized vault balance 22655000000000000000
   total Winner Shares 24625000000000000000
   user2 balance 18217800000000000000
```
</details>

From the results, we can see that totalWinnerShares is greater than the finalized vault balance, and user2 wins but withdraws even less than their deposit.


**Recommended Mitigation:** 
To allow each user to join only once, use a mapping to track whether a user has already joined and revert if they try again. If multiple joins are allowed, calculate only the newly added shares each time instead of using the `msg.sender` balance, and avoid pushing the same user multiple times into usersAddress.

```diff
+    mapping(address => bool) public hasJoined;

     function joinEvent(uint256 countryId) public {
+       if(hasJoined[msg.sender]){revert alreadyjoined};

```



### [H-3] `Brivault` overrides `deposit` instead of `_deposit` function, causing shares to be minted to `msg.sender` rather than the intended `receiver` and bypassing `ERC4626’s:: _convertToShares`, which leads to inaccurate share calculations.

**Description:** The `Brivaul`t contract inherits from `ERC462`6 and overrides the `deposit` function. In `ERC4626`, overriding the internal `_deposit` function automatically affects both the `deposit` and `mint` mechanisms. The `_deposit` function is supposed to mint shares to the specified `receiver` using `_mint(receiver, shares)`. However, in the overridden deposit function, shares are instead minted to `msg.sender` via `_mint(msg.sender, participantShares)`.

Additionally, by overriding deposit directly, the contract bypasses the standard `ERC4626 ::_convertToShares` logic, resulting in imprecise share calculations that may not correctly reflect the proportional ownership based on the vault’s assets.

```javascript
    function deposit(uint256 assets, address receiver) public override returns (uint256) {
      
```

**Impact:** The result is that the `receiver` does not receive the expected minted tokens; instead, the `msg.sender` receives the tokens.

**Proof of Concept:**
User1 approves Brivault and deposits on behalf of User2.As a result, User2 did not receive the shares.

Place the following code in briVault.t.sol.

```solidity
    function testReceiverCanNotGetTheShared() public {
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 5 ether);
        briVault.deposit(5 ether, user2);
        vm.stopPrank();

        console2.log("user balance 1:", briVault.balanceOf(user1));
        console2.log("user balance 2:", briVault.balanceOf(user2));
        assertEq(briVault.balanceOf(user2), 0 );
        
    }

```
The Result: 

```
  user balance 1: 4925000000000000000
  user balance 2: 0
```

**Recommended Mitigation:**  The `_deposit` function should be overridden instead of `deposit`.
```diff
+       function deposit(uint256 assets, address receiver) public override returns (uint256) {
+           return super.deposit(assets,receiver);}
        
-       function deposit(uint256 assets, address receiver) public override returns (uint256) {...}
+       function _deposit(address caller, 
+            address receiver, uint256 assets, int256 shares)internal override {
              
```


## Medium

### [M-1] Iterating Over All Users in `BriVault::_getWinnerShares ()`  Allows Potential DoS

**Description:** The internal function `_getWinnerShares()`, which is called by `setWinner()`, contains an unbounded loop that iterates over all entries in the `usersAddress` array to calculate the `totalWinnerShares`. As the number of users increases, the gas cost of this operation grows linearly. When the number of participants becomes large, the function may consume excessive gas and exceed the block gas limit, causing `setWinner()` to revert. 

```javascript
    function _getWinnerShares () internal returns (uint256) {
        for (uint256 i = 0; i < usersAddress.length; ++i){
@>          address user = usersAddress[i]; 
@>          totalWinnerShares += userSharesToCountry[user][winnerCountryId];
        }
        return totalWinnerShares;
    }

```

**Impact:** This creates a potential Denial of Service (DoS) condition, where the contract owner is unable to finalize the winner or execute other dependent operations.

**Proof of Concept:**
- `testSetWinnerOnlyOneUser`: Calculates the gas cost when only one participant joins.
- `testSetWinnerMoreUsers`: Calculates the gas cost when three participants join.

Place the following code in briVault.t.sol.

<details>
<summary>Proof Of Code</summary>

```javascript
function testSetWinnerOnlyOneUser() public {
        // only 1 people join 
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 10 ether);
        briVault.deposit(5 ether, user1);
        briVault.joinEvent(10);
        vm.stopPrank();

        vm.warp(eventEndDate + 1);
        vm.startPrank(owner);
        uint256 gasbefore= gasleft();
        briVault.setWinner(10);
        uint256 gasAfter= gasleft();
        console2.log("gas used when only 1 user", gasbefore - gasAfter);
        vm.stopPrank();
        
    }

        function testSetWinnerMoreUsers() public {
        // 1st people join 
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 10 ether);
        briVault.deposit(5 ether, user1);
        briVault.joinEvent(10);
        vm.stopPrank();
        // 2nd 
        vm.startPrank(user2);
        mockToken.approve(address(briVault), 10 ether);
        briVault.deposit(10 ether, user2);
        briVault.joinEvent(7);
        vm.stopPrank();

        // 3rd
        vm.startPrank(user3);
        mockToken.approve(address(briVault), 10 ether);
        briVault.deposit(10 ether, user3);
        briVault.joinEvent(4);
        vm.stopPrank();

        vm.warp(eventEndDate + 1);
        vm.startPrank(owner);
        uint256 gasbefore= gasleft();
        briVault.setWinner(10);
        uint256 gasAfter= gasleft();
        console2.log("gas used when 3 users ", gasbefore - gasAfter);
        vm.stopPrank();
    }
```

```
gas used when only 1 user 103929
gas used when 3 users  110447

```
</details>
When only two additional participants join, the gas cost increases by 6,518.

**Recommended Mitigation:** Use an accumulated total approach instead of looping through all users. Maintain a running total of shares per country that updates whenever users join event. This removes the need for an unbounded loop and keeps gas usage constant.

```diff
+  mapping(uint256 => uint256) public CountryToTotalshares;

function joinEvent(uint256 countryId) public {
    ...
    userToCountry[msg.sender] = teams[countryId];
    uint256 participantShares = balanceOf(msg.sender);
    userSharesToCountry[msg.sender][countryId] = participantShares;

    // update each country's total shares
+   CountryToTotalshares[countryId] += participantShares;

-   usersAddress.push(msg.sender);
    numberOfParticipants++;
    totalParticipantShares += participantShares;
    emit joinedEvent(msg.sender, countryId);
}

```
