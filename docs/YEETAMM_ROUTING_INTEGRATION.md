# YeetAMM Routing & Integration Guide

**Audience:** external aggregators / routers (e.g. Jupiter) and anyone building
an off-chain quoter against YeetAMM pools.
**Status:** v1 — 2026-07-08. Matches on-chain `programs/yeet-amm` and the
reference off-chain quoter in `yeetlaunch/backend/src/services/yeetAmm.service.ts`.

> **Read this first — pricing is NOT pure constant-product.** YeetAMM pools
> price against **real + virtual** reserves, and sells are **rate-limited**.
> A naive `x*y=k` quote will be wrong (understated depth) and a naive sell
> router will hit on-chain reverts. Both are fully specified below.

---

## 1. Program IDs

| Cluster | Program ID |
|---------|------------|
| Mainnet | `yeetaecvxpd7DFzZAYTEYracRt1WYJ7DfMVjEeEt2Cp` |
| Devnet  | `yeetMcJ7nBfMZiQV8ns4h5m3WeVekT1ySq27bMWWfio` |

The two binaries differ **only** by `declare_id!` (two `cfg(feature="mainnet")`
gates, both the id) — identical audited logic.

A pool has two lifecycle modes on the same account:
- **DBC** (`pool_mode = 0`) — bonding curve, pre-graduation.
- **AMM** (`pool_mode = 1`) — post-graduation. Graduation is **atomic**: the
  swap that pushes the real quote reserve to `grad_threshold` flips the mode
  in the same transaction. There is no separate migration account or pool.

Side convention: **A = base (the launched token)**, **B = quote (SOL/WSOL)**.
`quote_mint` (offset 537) tells you which side is quote if you need to confirm.

---

## 2. Pool account layout

Anchor account, 8-byte discriminator at offset 0. Current on-chain size is
**593 bytes** (`INIT_SPACE = 585` + 8 disc). Older pools may be 545/577 bytes;
gate reads on `data.length`. All integers little-endian.

| Offset | Field | Type | Notes |
|-------:|-------|------|-------|
| 10  | `mint_a` (base) | Pubkey | |
| 42  | `mint_b` (quote) | Pubkey | |
| 74  | `vault_a` | Pubkey | base token vault |
| 106 | `vault_b` | Pubkey | quote token vault |
| 138 | `lp_mint` | Pubkey | |
| 170 | `creator` | Pubkey | |
| 202 | `creator_fee_vault_a` | Pubkey | |
| 234 | `creator_fee_vault_b` | Pubkey | |
| 266 | `reserve_a` | u64 | **real** base reserve |
| 274 | `reserve_b` | u64 | **real** quote reserve |
| 282 | `fee_bps` | u16 | always `100` (1%) — enforced at init |
| 284 | `creator_fee_share_bps` | u16 | **repurposed** → live adaptive sell-cap bps (see §5). NOT a fee. |
| 286 | `lp_supply` | u64 | |
| 294 | `last_updated_slot` | u64 | |
| 302 | `locked_lp_amount` | u64 | 30% permanent lock |
| 310 | `lock_release_slot` | u64 | `grad_slot + 48h cliff` |
| 318 | `lock_vault` | Pubkey | |
| 350 | `vesting_lp_amount` | u64 | 70% creator vest |
| 358 | `vesting_released` | u64 | |
| 366 | `vesting_start_slot` | u64 | |
| 374 | `vesting_end_slot` | u64 | |
| 382 | `vesting_vault` | Pubkey | |
| 414 | `is_initialized` | bool (1) | **skip pool if false** — partially set up |
| 496 | `virtual_reserve_a` | u64 | base-side anchor (see §4) |
| 504 | `virtual_reserve_b` | u64 | quote-side anchor (see §4) |
| 512 | `slot_sell_window` | u64 | `(slot << 16) \| consumed_bps` (see §5) |
| 520 | `pool_mode` | u8 | 0 = DBC, 1 = AMM |
| 521 | `grad_threshold` | u64 | graduation target on **real** `reserve_b` |
| 529 | `grad_slot` | u64 | slot graduation fired (0 pre-grad) |
| 537 | `quote_mint` | Pubkey | |
| 569 | `launch_lp_supply` | u64 | LP frozen at graduation (bucket math) |
| 577 | `last_sell_cap_update_slot` | u64 | bucket recovery bookkeeping |

`is_initialized == false` ⇒ do not route; the pool is mid-setup.

---

## 3. Fee model (fee-on-input)

Flat **1% (`fee_bps = 100`) taken from `amount_in`**, before the curve. Split
by mode (bps of `amount_in`, informational — the router only needs the 1% total
for pricing):

| Mode | Total | Yeet | Creator | LP (stays in pool) |
|------|------:|-----:|--------:|-------------------:|
| DBC  | 100 | 70 | 30 | 0 |
| AMM  | 100 | 50 | 30 | 20 |

For quoting, only the total matters: `amount_after_fee = amount_in − floor(amount_in × 100 / 10000)`.

---

## 4. Pricing — constant product over **effective** reserves

Effective reserve on each side = **real + effective virtual**:

```
eff_in  = reserve_in  + eff_virtual_in
eff_out = reserve_out + eff_virtual_out
amount_after_fee = amount_in − floor(amount_in × fee_bps / 10000)
amount_out = floor(amount_after_fee × eff_out / (eff_in + amount_after_fee))
```

`amount_out` must never be `>= reserve_out` (real) — the program caps output at
real reserves; treat a quote that would exceed the real out-reserve as
liquidity-limited.

### 4.1 Effective virtual reserves (decay)

Virtual reserves are **preserved at graduation**, then **lazily decay** from
100% of anchor to a **15% floor** over **54,000 slots** (~6h @ 400ms). Mirror
exactly (`Pool::effective_virtual_reserves`, `decay_anchor`):

```
eff_virtual(anchor, mode, grad_slot, current_slot):
    if mode != AMM or grad_slot == 0:      return anchor          # DBC / pre-grad: full anchor
    elapsed = max(0, current_slot - grad_slot)
    floor   = anchor * 1500 / 10000                               # 15%
    if elapsed >= 54000:                   return floor
    return floor + (anchor - floor) * (54000 - elapsed) / 54000
```

Consequences for routers:
- **DBC & pre-grad:** use the full `virtual_reserve_a/b`.
- **A buy that crosses graduation in one tx** prices its AMM leg at
  `elapsed = 0` ⇒ **full anchor**, not zero. (Common integration bug: zeroing
  virtuals post-DBC understates depth.)
- **Post-grad quotes are slot-dependent.** Cache with the current slot; a stale
  slot yields a stale price. Refresh per quote.

---

## 5. Sell rate-limiting (routers MUST model this)

Sells are capped; buys are not. Two independent layers, both keyed on the
**input-side real reserve** (`reserve_in`, i.e. the base reserve for a sell).
Enforced only in **AMM mode**.

`consumed_bps` for a sell = **ceil**(`amount_in × 10000 / reserve_in`).

**Layer 1 — hard per-slot aggregate cap.** `slot_sell_window` packs
`(slot << 16) | cumulative_consumed_bps`. All sells landing in the same slot
serialize on the writable pool account and sum; if the running total would
exceed **`PER_SLOT_SELL_CAP_BPS = 500` (5%)**, the whole tx reverts with
`AdaptiveSellCapExceeded` (custom error **6042 / 0x1789**). Closes the
multi-wallet same-slot bypass.

**Layer 2 — decaying bucket.** `creator_fee_share_bps` (offset 284) holds the
live cap, `[50, 500]` bps. Each sell subtracts its `consumed_bps`; it recovers
**+50 bps every 32 slots** back up to 500. Floor 50 bps (0.5%) always
available.

**Router guidance:**
- Read the live cap from **offset 284** (`min(cap, 500 − slot_consumed_so_far)`
  is the max sellable this slot).
- Size sell legs so `consumed_bps ≤ remaining capacity`; split large exits
  across slots. A single sell over ~5% of the base reserve **cannot** execute
  in one slot regardless of wallets.
- Buys have **no** cap.

---

## 6. Reference quoters

```ts
// BUY: quote in (mint_b), base out (mint_a). No sell cap.
function quoteBuy(pool, amountInQuote, currentSlot) {
  const evA = effVirtual(pool.virtualReserveA, pool.mode, pool.gradSlot, currentSlot);
  const evB = effVirtual(pool.virtualReserveB, pool.mode, pool.gradSlot, currentSlot);
  const afterFee = amountInQuote - (amountInQuote * 100n) / 10000n;
  const effIn  = pool.reserveB + evB;
  const effOut = pool.reserveA + evA;
  let out = (afterFee * effOut) / (effIn + afterFee);
  if (out >= pool.reserveA) out = pool.reserveA - 1n;   // real-reserve cap
  return out;
}

// SELL: base in (mint_a), quote out (mint_b). Must respect the per-slot cap.
function quoteSell(pool, amountInBase, currentSlot) {
  if (pool.mode === 'AMM') {
    const consumedBps = ceilDiv(amountInBase * 10000n, pool.reserveA);   // vs REAL base reserve
    const slot = pool.slotSellWindow >> 16n;
    const used = slot === BigInt(currentSlot) ? (pool.slotSellWindow & 0xFFFFn) : 0n;
    if (used + consumedBps > 500n) return { revert: 'AdaptiveSellCapExceeded(6042)' };
  }
  const evA = effVirtual(pool.virtualReserveA, pool.mode, pool.gradSlot, currentSlot);
  const evB = effVirtual(pool.virtualReserveB, pool.mode, pool.gradSlot, currentSlot);
  const afterFee = amountInBase - (amountInBase * 100n) / 10000n;
  const effIn  = pool.reserveA + evA;
  const effOut = pool.reserveB + evB;
  let out = (afterFee * effOut) / (effIn + afterFee);
  if (out >= pool.reserveB) out = pool.reserveB - 1n;
  return { out };
}
```

Canonical implementation to diff against: `effectiveVirtualReservesAB`,
`quoteExactIn`, `consume_adaptive_sell_capacity` in
`yeetlaunch/backend/src/services/yeetAmm.service.ts` and
`programs/yeet-amm/src/state.rs`.

---

## 7. Graduation & liquidity (routing implications)

- **Graduation trigger:** real `reserve_b` (net quote contribution — sells
  subtract) reaching `grad_threshold`. Not gross volume.
- **Atomic:** no migration window; the crossing tx already reflects AMM state.
- **LP lock:** 30% permanently locked, 70% creator-vesting (48h cliff, then
  5%/day to day 16). LP cannot be pulled during the cliff — relevant if you
  model rug risk. Locked/vesting LP is **not** in the tradable reserves; price
  is set by `reserve_a/b` + effective virtuals only.

---

## 8. Integration checklist

- [ ] Skip pools with `is_initialized == false`.
- [ ] Use `reserve_a/b` (real) **plus** effective virtual reserves — never pure CPMM.
- [ ] Recompute effective virtuals against **current slot** every quote (post-grad decay).
- [ ] For a buy crossing graduation, price the AMM leg with **full** anchor virtuals.
- [ ] Model the 5%/slot sell cap; split large sells across slots; buys uncapped.
- [ ] Cap output at real out-reserve.
- [ ] Fee is 1% on input; don't double-count the mode split.
