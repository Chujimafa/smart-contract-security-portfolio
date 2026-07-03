# L1_sendL1Bridge uses spotBalance.total instead of total - hold, causing silent L1 drop and permanent outstandingL1Principal corruption

## Description

According to the [Hyperliquid documentation](https://docs.chainstack.com/reference/hyperliquid-info-spotclearinghousestate), `hold` represents the amount locked in open orders or pending transactions, and the **available balance** is defined as `Total - Hold`. However, in `MonetrixVault._sendL1Bridge` ([L557](https://github.com/code-423n4/2026-04-monetrix/blob/main/src/core/MonetrixVault.sol#L557)), `l1Available` is computed using only `spotBalance.total`, without subtracting `hold`:

```solidity
function _sendL1Bridge(uint256 amount) internal {
    uint64 usdcToken = uint64(HyperCoreConstants.USDC_TOKEN_INDEX);
@>  uint256 l1Available = uint256(PrecompileReader.spotBalance(address(this), usdcToken).total);
    if (pmEnabled) {
        l1Available += uint256(PrecompileReader.suppliedBalance(address(this), usdcToken));
    }
    require(
@>      l1Available >= TokenMath.usdcEvmToL1Wei(amount),
        "L1 USDC insufficient (unwind hedge or wait for settlement)"
    );
```

This means the guard passes even when the actual free balance (`total - hold`) is insufficient to cover the requested bridge amount.

On HyperEVM, CoreWriter actions are not atomic with L1 execution. As noted in the [QuillAudits Hyperliquid security analysis](https://www.quillaudits.com/blog/blockchain/hyperliquid-security-beyond-orderbooks) under *Non-Atomicity*: *"If that action fails because the user has insufficient margin on HyperCore, or the order cannot be filled, or the asset has been delisted, the original HyperEVM transaction does not revert. Your contract has already committed its state changes."* Therefore, if L1 cannot fulfill the `SEND_ASSET` action because the free balance is insufficient, the transfer will not be executed — there is no revert, no callback, and no on-chain signal to the EVM.

As a result, `_sendL1Bridge` can emit a bridge action that L1 will never execute, while the EVM-side accounting has already been updated as if the transfer succeeded.


## Impact

Because `_sendL1Bridge` is the shared internal path for both `bridgePrincipalFromL1` and `emergencyBridgePrincipalFromL1`, both functions decrement `outstandingL1Principal` before the L1 transfer executes:

```solidity
outstandingL1Principal -= amount;  // EVM accounting updated immediately
_sendL1Bridge(amount);             // L1 action may silently drop
```

When L1 does not execute the transfer, the EVM records `outstandingL1Principal` as reduced by `amount`, while the actual USDC never leaves L1. This creates a permanent accounting gap: the vault believes it has bridged funds back to cover a pending redemption, but `RedeemEscrow` never receives any USDC. The subsequent guard:

```solidity
require(amount <= outstandingL1Principal, "invalid bridge amount")
```

now enforces a lower ceiling than the real L1 balance, making it impossible to re-issue the correct bridge amount. The user's `claimRedeem` call reverts with `"RedeemEscrow: insufficient liquidity"` and there is no recovery path — the USDC remains stranded on L1 while the EVM accounting permanently understates the principal.

## Proof of Concept

Attack path:

1. User deposits 100k USDC → operator bridges all 100k to L1 (`outstandingL1Principal = 100k`)
2. Keeper places open orders on L1: `hold = 50k`, `free = 50k`
3. User requests redemption of 80k USDM → `redemptionShortfall = 80k`
4. Operator calls `bridgePrincipalFromL1(80k)`:
   - EVM guard checks `total (100k) >= 80k` → passes (bug: ignores `hold`)
   - `outstandingL1Principal` decremented to 20k
   - L1 does not execute `SEND_ASSET` because `free (50k) < 80k`; 0 USDC arrives on EVM
5. User calls `claimRedeem` after cooldown → `RedeemEscrow` has 0 USDC → reverts

**Proof of code** 

copy the following test case into `test/c4/C4Submission.t.sol`:

```solidity
 function test_bridgeAvailabilityMisusesTotal() public {

        // ── Step 1: Deposit 100k, bridge ALL to L1 ───────────────
        _deposit(user1, 100_000e6);
        assertEq(usdc.balanceOf(address(vault)), 100_000e6, "vault holds 100k before bridge");
        // ── Step2: operator bridge all usdc to L1, netBridgeable Amount = 100k
        vm.warp(block.timestamp + 6 hours + 1);
        vm.prank(operator);
        vault.keeperBridge(MonetrixVault.BridgeTarget.Vault);
        assertEq(vault.outstandingL1Principal(), 100_000e6, "all 100k on L1");
        assertEq(usdc.balanceOf(address(vault)), 0, "vault EVM USDC = 0 after full bridge");

        // ── Step 3: Keeper places limit orders — 30% of L1 USDC locked ──
        // total = 100_000e8, hold = 50_000e8 (50%), free = 50_000e8 (70%)
        _PoCMockPrecompile(payable(HyperCoreConstants.PRECOMPILE_SPOT_BALANCE))
            .setResponse(
                abi.encode(address(vault), uint64(HyperCoreConstants.USDC_TOKEN_INDEX)),
                abi.encode(uint64(100_000e8), uint64(50_000e8), uint64(0)) // total, hold, entryNtl
            );

        // ── Step 4: user1 requests redemption of 80k USDM ───────
        // shortfall = 80k (vault EVM = 0, redeemEscrow = 0)
        uint256 redeemId = _requestRedeem(user1, 80_000e6);
        assertEq(vault.redemptionShortfall(), 80_000e6, "shortfall = 80k");

        // ── Step 5: Operator calls bridgePrincipalFromL1(80k) ────
        // Vault gate: 80k <= shortfall(80k) && 80k <= outstandingL1Principal(100k) -> pass
        vm.prank(operator);
        vault.bridgePrincipalFromL1(80_000e6); 
        assertEq(vault.outstandingL1Principal(), 20_000e6, "bridgePrincipalFromL1 success no revert");
       
        assertEq(usdc.balanceOf(address(redeemEscrow)), 0, "redeemEscrow still empty");
        assertEq(vault.redemptionShortfall(), 80_000e6, "shortfall unchanged");

        // ── Step 6: user try to claim after cooldown ─────────────
        vm.warp(block.timestamp + 3 days + 1);
        vm.prank(user1);
        vm.expectRevert("RedeemEscrow: insufficient liquidity");
        vault.claimRedeem(redeemId); // redeemEscrow has 0 USDC -> revert

        // ──  No recovery path ──────────────────────────────
        // outstandingL1Principal = 20k remains, but bridgePrincipalFromL1
        // requires amount <= shortfall(80k) AND amount <= principal(20k).
        // Any further call with free=50k will still exceed free 
        // The 80k accounting gap is permanent; 100k real USDC on L1 is
        // effectively orphaned from the protocol's EVM accounting.
        assertEq(vault.outstandingL1Principal(), 20_000e6, "20k principal left vs 100k real on L1");
    }
```

## Recommended Mitigation

Fix `_sendL1Bridge` to check free balance (`total - hold`) instead of `total`, and move the `outstandingL1Principal` decrement to after L1 confirmation using a two-phase commit pattern:

```diff
  function _sendL1Bridge(uint256 amount) internal {
      uint64 usdcToken = uint64(HyperCoreConstants.USDC_TOKEN_INDEX);
-     uint256 l1Available = uint256(PrecompileReader.spotBalance(address(this), usdcToken).total);
+     SpotBalance memory bal = PrecompileReader.spotBalance(address(this), usdcToken);
+     uint256 l1Available = uint256(bal.total - bal.hold); // free balance only
      if (pmEnabled) {
          l1Available += uint256(PrecompileReader.suppliedBalance(address(this), usdcToken));
      }
      require(
          l1Available >= TokenMath.usdcEvmToL1Wei(amount),
          "L1 USDC insufficient (unwind hedge or wait for settlement)"
      );
      ActionEncoder.sendBridgeToL1(amount);
  }
```

However, since CoreWriter actions are not atomic with L1 execution, the free balance check can pass at call time but the L1 state may still change between blocks. The safer structural fix, as described in the [QuillAudits Hyperliquid security analysis](https://www.quillaudits.com/blog/blockchain/hyperliquid-security-beyond-orderbooks), is to defer the accounting update until L1 execution is confirmed:

```diff
+ mapping(uint256 => uint256) public pendingBridgeAmount;
+ uint256 public nextBridgeId;

  function bridgePrincipalFromL1(uint256 amount) external onlyOperator {
      require(amount > 0 && amount <= redemptionShortfall() && amount <= outstandingL1Principal);
-     outstandingL1Principal -= amount;
      _sendL1Bridge(amount);
-     emit PrincipalBridgedFromL1(amount);
+     uint256 id = nextBridgeId++;
+     pendingBridgeAmount[id] = amount;
+     emit PrincipalBridgePending(id, amount);
  }

+ // Phase 2: confirm after USDC arrives on EVM
+ function confirmBridgeFromL1(uint256 id) external onlyOperator {
+     uint256 amount = pendingBridgeAmount[id];
+     require(amount > 0, "no pending bridge");
+     require(usdc.balanceOf(address(this)) >= amount, "USDC not yet received");
+     delete pendingBridgeAmount[id];
+     outstandingL1Principal -= amount;
+     emit PrincipalBridgedFromL1(amount);
+ }

+ // Escape hatch: cancel if L1 did not execute the action
+ function cancelBridgeFromL1(uint256 id) external onlyOperator {
+     uint256 amount = pendingBridgeAmount[id];
+     require(amount > 0, "no pending bridge");
+     delete pendingBridgeAmount[id];
+     emit PrincipalBridgeCancelled(id, amount);
+ }
```


