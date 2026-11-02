# YeetAMM Routing & Integration Guide

**Audience:** external aggregators / routers (e.g. Jupiter) and anyone building
an off-chain quoter against YeetAMM pools.
**Status:** v3 — 2026-11-02. Matches the on-chain `yeet-amm` program (release v1.4.0).

> **Read this first — pricing is NOT pure constant-product, and sells are
> rate-limited.** A naive `x*y=k` quote will misprice, and a naive sell router
> will hit on-chain reverts. Quote through the YeetLaunch API/SDK or on-chain
> simulation rather than replicating the pool's internal pricing (§4), and
> respect the live sell allowance (§5).

---

## 1. Program IDs

| Cluster | Program ID |
|---------|------------|
| Mainnet | `yeetaecvxpd7DFzZAYTEYracRt1WYJ7DfMVjEeEt2Cp` |
| Devnet  | `yeetMcJ7nBfMZiQV8ns4h5m3WeVekT1ySq27bMWWfio` |

The two binaries differ **only** by `declare_id!` — identical logic. The program has been internally security-reviewed and tested; an independent external audit is pending.

A pool has two lifecycle modes on the same account:
- **DBC** (`pool_mode = 0`) — bonding curve, pre-graduation.
- **AMM** (`pool_mode = 1`) — post-graduation. Graduation is **atomic**: the
  swap that pushes the real quote reserve to `grad_threshold` flips the mode
  in the same transaction. There is no separate migration account or pool.

**Which side is SOL? Do not assume.** `mint_a` and `mint_b` are ordered by their
raw 32 bytes, not by role, so wSOL is side A on some pools and side B on
others. Compare the mints (or read `quote_mint`, offset 537) on every pool.
The pools API publishes it as `metadata.quoteReserveSide` (`'a'` or `'b'`).
Assuming a fixed side works on some pools and silently inverts price and
liquidity on the rest.

---

## 2. Pool account layout

Anchor account, 8-byte discriminator at offset 0. Current on-chain size is
**593 bytes** (`INIT_SPACE = 585` + 8 disc). Older pools may be 545/577 bytes;
gate reads on `data.length`. All integers little-endian.

| Offset | Field | Type | Notes |
|-------:|-------|------|-------|
| 10  | `mint_a` | Pubkey | lower of the two mints by raw bytes |
| 42  | `mint_b` | Pubkey | higher of the two mints by raw bytes |
| 74  | `vault_a` | Pubkey | `mint_a` vault |
| 106 | `vault_b` | Pubkey | `mint_b` vault |
| 138 | `lp_mint` | Pubkey | |
| 170 | `creator` | Pubkey | |
| 202 | `creator_fee_vault_a` | Pubkey | |
| 234 | `creator_fee_vault_b` | Pubkey | |
| 266 | `reserve_a` | u64 | **real** `mint_a` reserve |
| 274 | `reserve_b` | u64 | **real** `mint_b` reserve |
| 282 | `fee_bps` | u16 | always `100` (1%) — enforced at init |
| 284 | `creator_fee_share_bps` | u16 | repurposed internal field — **not** a fee; do not use for pricing |
| 286 | `lp_supply` | u64 | |
| 294 | `last_updated_slot` | u64 | |
| 302 | `locked_lp_amount` | u64 | 30% permanent lock |
| 310 | `lock_release_slot` | u64 | `grad_slot` (no cliff); `grad_slot + 432,000` (48h) on pools graduated before the 10-day schedule |
| 318 | `lock_vault` | Pubkey | on new pools its tokens are owned by `FboVV3DsgKhJHbu78RyWxx9zryd4T1W5cQbUXEjJ9e6t`, an off-curve address nothing can sign for |
| 350 | `vesting_lp_amount` | u64 | 70% creator vest |
| 358 | `vesting_released` | u64 | |
| 366 | `vesting_start_slot` | u64 | |
| 374 | `vesting_end_slot` | u64 | |
| 382 | `vesting_vault` | Pubkey | |
| 414 | `is_initialized` | bool (1) | **skip pool if false** — partially set up |
| 496 | `virtual_reserve_a` | u64 | virtual anchor, `mint_a` side; fixed for the life of the pool |
| 504 | `virtual_reserve_b` | u64 | virtual anchor, `mint_b` side; fixed for the life of the pool |
| 512 | `slot_sell_window` | u64 | per-slot sell accounting (see §5) |
| 520 | `pool_mode` | u8 | 0 = DBC, 1 = AMM |
| 521 | `grad_threshold` | u64 | graduation target on the **real** SOL-side reserve (48 SOL for pools launched since 2026-10-14, 32 SOL before; read it per pool) |
| 529 | `grad_slot` | u64 | slot graduation fired (0 pre-grad) |
| 537 | `quote_mint` | Pubkey | wSOL; tells you which side is SOL |
| 569 | `launch_lp_supply` | u64 | LP frozen at graduation |
| 577 | `last_sell_cap_update_slot` | u64 | sell-cap bookkeeping |

`is_initialized == false` ⇒ do not route; the pool is mid-setup.

---

## 3. Fee model (fee-on-input)

Flat **1% (`fee_bps = 100`) taken from `amount_in`**, before the curve. Split
by mode (informational — the router only needs the 1% total):

| Mode | Total | Yeet | Creator | LP (stays in pool) |
|------|------:|-----:|--------:|-------------------:|
| DBC  | 100 | 70 | 30 | 0 |
| AMM  | 100 | 50 | 30 | 20 |

---

## 4. Pricing

YeetAMM pools are **not** pure constant-product: they price against the real
reserves plus an internal virtual-reserve component, so a naive
`reserve_out / reserve_in` (or `x*y=k` on the real reserves) will misprice.
Rather than replicate the pool's internal pricing, quote through one of:

- **Quote API (recommended).**
  `GET https://api.yeetlaunch.io/quote?inputMint=&outputMint=&amount=&slippageBps=`
  returns the server-computed `outputAmount`, `minOutputAmount`, and
  `priceImpactPct` for an exact-in swap. Authoritative and always in sync with
  on-chain behavior.
- **On-chain simulation.** Build the swap instruction (use the SDK, §6) and
  `simulateTransaction` to read the exact output — no local pricing model needed.
- **Executed trades (price feeds / charts).** Every swap emits a `SwapEvent`
  carrying `amount_in`/`amount_out` (and `price_q64`, a Q64.64 convenience). The
  realized rate is exact and needs no pricing model.

Output is capped at the real out-side reserve; a fill that would exceed it is
liquidity-limited. The quote API rejects any single route whose input exceeds
5% of a pool's effective depth (`No route available for requested pair`); the
pair is still routable at a smaller size.

> **Changed 2026-08-13.** Virtual reserves no longer decay after graduation.
> They are fixed at launch and never change, so pricing does not depend on the
> slot. If you built a quoter from v1 of this guide, remove the decay.

---

## 5. Sell rate-limiting (routers must respect this)

Post-graduation (AMM mode), **sells are rate-limited per slot; buys are not.**
A sell that exceeds the current allowance reverts on-chain with
`AdaptiveSellCapExceeded` (custom error **6042 / 0x1789**), so a router that
ignores it will build failing transactions.

**Read the live allowance rather than modeling it.** The per-mint trading
snapshot exposes exactly what you need:

`GET https://yeetlaunch.io/api/yeet-amm/pool/:mint` →
- `trading.sellCapEnabled` — whether the cap is active,
- `trading.maxSellBaseRaw` — max token units sellable **right now**,
- `trading.perSlotRemainingBps` — remaining per-slot sell capacity.

Guidance: size each sell leg within `maxSellBaseRaw`, split large exits across
multiple slots, and re-read the allowance per quote. Buys have no cap.

---

## 6. Tooling

- **TypeScript SDK:** [`@yeetlaunch/yeet-amm-sdk`](https://github.com/YeetLaunch/yeet-amm-sdk) — program
  IDs, PDA derivation, pool-account decoding, swap instruction builders, turnkey
  buy/sell (automatic wSOL wrap/unwrap), error decoding, and a REST client for
  quotes. Build swaps and quotes without reimplementing anything.
- **Quote endpoint:** `https://api.yeetlaunch.io` (see §4).
- **Developer page:** <https://yeetlaunch.io/dev>

---

## 7. Graduation & liquidity (routing implications)

- **Graduation trigger:** the real SOL-side reserve (net contribution — sells
  subtract) reaching the pool's `grad_threshold`. Not gross volume.
- **Atomic:** no migration window; the crossing tx already reflects AMM state.
- **LP lock:** 30% permanently locked, 70% creator-vesting at 10% a day for
  10 days from graduation, nothing in the first day. Pools graduated before
  2026-10-17 keep a 48h cliff, then 14 days. The creator can withdraw vested
  LP, so real depth can fall after graduation. Locked/vesting LP is **not**
  part of the tradable reserves; quote via the API/SDK (§4), not from reserves
  alone.

---

## 8. Integration checklist

- [ ] Skip pools with `is_initialized == false`.
- [ ] Never assume which side is SOL — compare mints or read `quote_mint` (§1).
- [ ] Don't price naively from reserves — quote via the API, on-chain simulation, or the SDK (§4).
- [ ] Respect the per-slot sell cap: read the live allowance and split large sells; buys are uncapped (§5).
- [ ] Fee is 1% on input; the mode split is informational.
- [ ] Verify canonical pools by PDA (`["yeet_amm_pool", mint_a, mint_b]`, sorted) + program owner + wSOL `quote_mint`.
