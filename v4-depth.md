# `univ4 depth` — Reading the Liquidity Distribution of a Uniswap V4 Pool

> **Audience.** Written for traders coming from order-book backgrounds (CEX
> market making, FX, HFT) who want to understand what a Uniswap V3/V4 pool
> looks like under the hood. You're assumed to know what bid/ask, depth of
> book, slippage, and market impact mean. You're **not** assumed to know any
> Uniswap-specific math.

This doc has three parts:

1. **The mental model** — translating concentrated AMMs to order-book intuition.
2. **The primitives** — what ticks, sqrt-price, virtual liquidity, and `liquidityNet` actually are.
3. **Reading the output** — column-by-column walkthrough of `univ4 depth`, with a worked example.

---

## 1. The mental model: AMM vs. order book

### Traditional CLOB

You're used to a discrete book:

```
       Asks
       2,302.50  →   12 ETH
       2,301.20  →    8 ETH
       2,300.40  →    3 ETH
  -- mid 2,300.00 --
       2,299.60  →    4 ETH
       2,298.50  →   10 ETH
       2,297.00  →   15 ETH
       Bids
```

To buy 10 ETH you walk the asks: take 3 + 8, then 12 partially, weighted-average → effective price ≈ 2,301.5. Slippage = `(effective − mid) / mid`.

The **depth** at any price level is the resting size at that level. The book's shape determines slippage as a function of trade size.

### Uniswap V2 (constant product, for context)

A V2 pool has **one** continuous AMM curve `x · y = k` covering every price from 0 to ∞. Every LP is equally exposed across that whole range. There's no "level" — slippage is just a function of how far the swap pushes the curve.

The downside: most of the LP's capital is "deployed" at prices that almost never trade (10× and 0.1× of fair value). Capital efficiency is terrible.

### Uniswap V3/V4 (concentrated liquidity)

V3/V4 lets each LP **choose a price range** `[P_lower, P_upper]` to provide liquidity in. Their capital only earns fees while the pool's price is in that range, but it's **deployed densely** within the range — so a small position can provide deep liquidity *if it's in range*.

The big consequence: **the pool now has shape**. Different price levels have different amounts of LP capital backing them, exactly like a CLOB. The pool's slippage curve depends on where LPs have stacked their ranges.

This is what `univ4 depth` shows you: the shape.

### The cleanest analogy

```
  Concentrated AMM bucket  ≈  CLOB price level (continuous, not discrete)
  Active liquidity L       ≈  resting size at the level
  Bucket boundary (tick)   ≈  the price gap between adjacent levels
  liquidityNet at a tick   ≈  size added/removed at that "level transition"
  Walking outward from mid ≈  walking the book to estimate slippage
```

The differences:

- A V3/V4 "level" is a **continuous price interval** (a "bucket"), not a single price. Within it, the AMM curve takes over — slippage compounds smoothly until you hit the next boundary.
- `L` is **virtual liquidity**, not a token amount. (See §2.3 — this trips everyone up.)
- The book is **two-sided by construction**: every LP range simultaneously provides bid-side and ask-side depth as price oscillates inside it.

---

## 2. The primitives

### 2.1 Ticks: the price grid

Uniswap stores price on an integer grid called **ticks**. The relationship is purely a coordinate transform:

```
price = 1.0001 ^ tick
```

So every 1-tick step is exactly **+0.01 %** in price. tick `0` = price 1.0; tick `1` = price 1.0001; tick `100` = +1.0050 %; tick `−100` = −0.9950 %.

The grid covers `[MIN_TICK, MAX_TICK] = [−887272, +887272]`, which spans roughly `1e−38` to `1e+38` in price. Plenty.

> **Decimal adjustment.** "price" above is the raw ratio of token1/token0 in their on-chain integer units. If you're looking at ETH/USDC where ETH has 18 decimals and USDC has 6, the displayed (human) price = `1.0001^tick × 10^(18−6) = 1.0001^tick × 10^12`. The CLI does this conversion for you in the **Price** column.

**`tickSpacing`.** Each pool picks a stride between *usable* ticks: `1` (tightest, for stablecoins), `10` (0.05 % tier), `60` (0.30 % tier), `200` (1 % tier). LP positions can only have boundaries on ticks that are multiples of the spacing. So in the 0.30 % tier, "valid" tick boundaries are `…, −60, 0, 60, 120, …`.

This means the pool effectively has a **coarser grid** than 0.01 %: a 60-tick spacing implies adjacent LP boundaries can be no closer than ≈ 0.60 % apart in price. That's the minimum bucket width for that pool.

### 2.2 `sqrtPriceX96`: why we use the square root

The pool stores `sqrt(price)` rather than `price`, scaled to a 96-bit fixed-point integer (`Q96 = 2^96 ≈ 7.92e28`). So `sqrtPriceX96 = floor(sqrt(price) × 2^96)`.

Three reasons:

1. **Liquidity math becomes linear in `sqrt(P)`.** The amounts you need to span a price interval are proportional to differences in `sqrt(P)`, not `P` itself (see §2.4). This eliminates a lot of square roots from the on-chain swap loop.
2. **Precision.** Storing `sqrt(P)` in Q96 fixed-point means the pool can represent price to ~10 decimal digits across the whole tick range, with no float drift.
3. **Symmetry.** The formulas for "how much token0 to traverse this slice" and "how much token1 to traverse this slice" become cleanly symmetric in sqrt-price space.

You don't need to do this math by hand — but when you see "sqrtPriceX96" in JSON outputs or events, that's why.

### 2.3 Virtual liquidity `L` — the most-confused concept

`L` is **not a dollar amount**. It is **not the number of tokens in the pool**. It's a single scalar that parameterizes a constant-product AMM curve, and it has weird-looking units (roughly: `√(token0 × token1)` in the pool's raw integer wei).

The intuition that *does* work for traders:

> **`L` is "depth per tick".** A bigger `L` in a bucket means the constant-product curve in that bucket is "stiffer" — a given token-in pushes the price less. It's the AMM analog of resting size at a price level.

For a constant-product pool with virtual liquidity `L`, the relationship is:

```
sqrt(P) × token1_in_pool = L
sqrt(P) × token0_in_pool = L  (after re-arrangement)
```

So if you double `L`, you halve the price impact of any given trade in the same bucket. That's the property that matters: **`L` scales depth**.

When `univ4 depth` shows you `L_active = 1.98e8`, you're not supposed to convert that to dollars. You compare it across buckets to see *where* depth is concentrated.

### 2.4 Bucket math: how amounts and prices relate inside one bucket

Inside a bucket `[tick_a, tick_b]` with constant `L`, the AMM is just a constant-product curve. To swap from `sqrt(P_a)` to `sqrt(P_b)`:

```
amount0_traded  =  L × (sqrt(P_b) − sqrt(P_a)) / (sqrt(P_a) × sqrt(P_b))
amount1_traded  =  L × (sqrt(P_b) − sqrt(P_a))
```

The first formula is for token0 (the "base" / left side); the second for token1. The exact rearrangement comes from `x · y = k` with `x = L/sqrt(P)` and `y = L × sqrt(P)`.

Don't worry about deriving these. The takeaway:

- **Both deltas scale with `L`.** Bigger `L` → less price move for the same token in.
- **Both deltas scale with `Δsqrt(P)`.** A bigger price move costs more, predictably.
- **token1 is "linear" in sqrt-price; token0 is "inverse".** This asymmetry is why the two columns in the depth table look different.

### 2.5 `liquidityNet`: the per-tick boundary delta

Every LP position contributes the same amount of `L` to every bucket inside its range. When the pool's price crosses a tick boundary, some LP positions become active (price moved into their range) and others become inactive (price moved out).

The pool tracks this with a single signed number per initialized tick: `liquidityNet`. The convention:

- When a position [a, b] is minted with virtual liquidity `L_pos`:
  - `liquidityNet[a] += L_pos`  (position activates when price crosses upward through `a`)
  - `liquidityNet[b] -= L_pos`  (position deactivates when price crosses upward through `b`)

To compute the active liquidity after crossing tick T:

```
crossing UP   (price increasing):  L_new = L_old + liquidityNet[T]
crossing DOWN (price decreasing):  L_new = L_old − liquidityNet[T]
```

So `liquidityNet` is **sign-stable** (the same number regardless of which direction you cross), but it gets **applied with opposite signs** depending on direction. This is the key to understanding the **ΔL** column.

### 2.6 The bitmap: how the pool finds the next initialized tick

Most ticks have no LP boundary on them. Walking from current tick to the next initialized one efficiently is a bookkeeping problem the pool solves with a **bitmap**:

- Each "word" of the bitmap is 256 bits, indexed by `wordPos` (a signed `int16`).
- Each bit at position `bitPos` (0..255) within word `wordPos` is set iff the **compressed tick** `(wordPos << 8) | bitPos` is initialized.
- `compressed_tick = real_tick / tickSpacing`.

To find every initialized tick within ±200 ticks of current with `tickSpacing = 60`: cover `±200 / 60 ≈ ±4` compressed positions, which fit in **one word**. One RPC call gives you the entire neighborhood.

`univ4 depth` does this walk for you (`StateView.getTickBitmap`), then batch-reads the `liquidityNet` of each set bit it finds.

---

## 3. Reading the `univ4 depth` output

### Sample run

```
amm-trading univ4 depth ETH_USDC_30
```

```
ETH_USDC_30  fee 0.3000%   tickSpacing 60
Current: tick=-198,920   price=2,300.0000 USDC/ETH   active L=198,402,777,123,456

Liquidity depth around current  (range ±200 bps,  showing 8 rows each side)

  Side    Tick     Price          L_active             ΔL          Cum. token-in        Cum. token-out
  -----------------------------------------------------------------------------------------------------
  Sell  -201,180   2,144.30    1.860e+08      +1.240e+07         52.81 ETH         113,612.40 USDC
  Sell  -200,400   2,202.10    1.740e+08      +3.040e+07         33.20 ETH          71,401.55 USDC
  Sell  -199,680   2,251.50    1.430e+08      +4.600e+07         18.51 ETH          40,200.21 USDC
  Sell  -199,140   2,289.20    9.710e+07      +2.500e+07          4.80 ETH          10,800.50 USDC
  ───── current ──────────────────────────────────────────────────────────────────────────────────────
  Buy   -198,720   2,313.10    1.984e+08      -2.800e+07     11,210.40 USDC          4.85 ETH
  Buy   -198,300   2,361.40    1.700e+08      -4.550e+07     35,128.50 USDC         14.92 ETH
  Buy   -197,640   2,438.70    1.250e+08      -3.210e+07     74,288.61 USDC         31.04 ETH
  Buy   -197,040   2,510.20    9.280e+07      -1.980e+07    119,540.90 USDC         48.21 ETH

Quick reference (slippage vs mid 2,300.0000 USDC/ETH):

  Side    Size            Input                  Output     Effective       Slip
  ----------------------------------------------------------------------------------
  Buy     1.0000      2,304.10 USDC          1.0000 ETH      2,304.10    +0.178%
  Buy     5.0000     11,584.30 USDC          5.0000 ETH      2,316.86    +0.733%
  Buy    10.0000     23,512.40 USDC         10.0000 ETH      2,351.24    +2.228%
  Sell    1.0000          1.0000 ETH      2,295.40 USDC      2,295.40    -0.200%
  Sell    5.0000          5.0000 ETH     11,407.10 USDC      2,281.42    -0.808%
  Sell   10.0000         10.0000 ETH     22,490.50 USDC      2,249.05    -2.215%
```

### Header

```
ETH_USDC_30  fee 0.3000%   tickSpacing 60
Current: tick=-198,920   price=2,300.0000 USDC/ETH   active L=198,402,777,123,456
```

- **fee** — pool's static LP fee (1 pip = 0.0001 %; 3000 pips = 0.30 %). For dynamic-fee (hooked) pools, this shows the *current* `slot0.lpFee` and a footer note warns that the hook may change it on a per-swap basis.
- **tickSpacing** — minimum spacing between LP boundaries on this pool's grid.
- **tick / price** — current state. `price` is human-readable (decimals applied).
- **active L** — current bucket's virtual liquidity. The number is in raw on-chain units (no decimal conversion); compare across buckets to see relative depth.

### Depth table — column by column

#### `Side`

`Buy` rows are above current price (you'd be buying token0 / pushing price up).
`Sell` rows are below current (selling token0 / pushing price down).

The middle separator marks the current bucket — the pool is somewhere inside it.

> The choice of "Buy = up" assumes token0 is the asset you care about. For ETH/USDC, currency0 is ETH, so buying = buying ETH = price goes up. For a USDC/USDT pool, currency0 might be USDC, and "buy USDC" means price moves up too — the convention is consistent: **Buy = price up = swap into token0**.

#### `Tick`

The integer tick number where this LP boundary sits. Always a multiple of the pool's `tickSpacing`. Empty ticks are skipped — only **initialized** ticks (i.e. at least one LP position has a boundary here) appear.

#### `Price`

Human-readable price = `1.0001^tick × 10^(decimals0 − decimals1)`. The "level" you'd quote a counterparty.

#### `L_active`

The virtual liquidity in the **bucket beginning at this tick** (going outward from current). Concretely:

- A Buy row at tick `−198,720` with `L_active = 1.984e8` → that's the L in the slice `[−198,720, next_initialized_tick_above]`. If you swap further than `−198,720`, this is the L the AMM uses next.
- A Sell row at tick `−199,140` with `L_active = 9.71e7` → that's the L in the slice `[next_tick_below, −199,140]`. If you swap further down past `−199,140`, this is the L next.

The intuition: **`L_active` per row = size of the next "level" you'd hit if you walked the book one more step**.

Notice the typical pattern: `L_active` is largest near current price and falls off as you move outward. That's because most LPs concentrate liquidity near where they think the price will trade.

#### `ΔL`

The pool's stored `liquidityNet` at this tick — a **signed** value, sign-stable across both swap directions. To use it:

- Going **up** through the tick (Buy direction): `L_new = L_prev + ΔL`.
- Going **down** through the tick (Sell direction): `L_new = L_prev − ΔL`.

Why the signs in the sample look the way they do:

- Sell-side ticks show **positive** ΔL (`+1.24e7`, `+3.04e7`, …). Going down through them, we *subtract* — so L drops, which is why `L_active` shrinks as we walk further down.
- Buy-side ticks show **negative** ΔL (`−2.80e7`, `−4.55e7`, …). Going up through them, we *add* — but adding a negative means L drops. Same intuition: depth falls off as you move outward.

In an order-book mental model: `ΔL` is like the **size delta** at each level transition. A big `|ΔL|` means lots of LPs have a boundary at this tick — either entering or leaving the active set.

#### `Cum. token-in`

The total **input** required (including fee) to push the pool's price all the way from current to this tick boundary, summed across every bucket in between.

- Buy row → input is **token1** (USDC). "11,210.40 USDC" means: spend 11,210.40 USDC; price rises from 2,300 → 2,313.10.
- Sell row → input is **token0** (ETH). "4.80 ETH" means: sell 4.80 ETH; price drops from 2,300 → 2,289.20.

This is your **slippage curve**: read down the column to see "how much input does it take to move price by N bps".

#### `Cum. token-out`

The matching **output** received for the same swap. Lets you compute effective price directly: `effective = cum_in / cum_out` (in the right direction). For the row at `−198,720`:

```
effective_buy_price = 11,210.40 USDC / 4.85 ETH ≈ 2,311 USDC/ETH
slippage vs mid = (2,311 / 2,300) − 1 = +0.48%
```

(The exact effective price for a partial fill landing inside this bucket would be slightly less; the cum-out figure is at the boundary, which is the worst case.)

### Quick reference table

This is the trade-size-centric view of the same data. For each requested size (`--quick-quotes`, defaults to `1, 5, 10` of token0):

- `Input` / `Output` — actual amounts the swap would route.
- `Effective` — `output / input` adjusted for direction; the price you actually achieve.
- `Slip` — `(effective / mid) − 1`. Positive on Buy (you paid more), negative on Sell (you got less).

The Quick reference is computed by walking buckets (and partial buckets via `compute_swap_step`) until the requested input is consumed. It will agree with the depth table at exact bucket boundaries and interpolate cleanly inside a bucket.

### Worked example: deciding a 5-ETH buy

You want to buy 5 ETH and want to know:

1. **How much will it cost?** Quick reference → "Buy 5: 11,584.30 USDC, effective 2,316.86, slip +0.73 %". Done.

2. **What price will it leave the pool at?** Find the depth-table row whose `Cum. token-out` ≈ 5 ETH. Between rows `Buy −198,720` (4.85 ETH out) and `Buy −198,300` (14.92 ETH out). So we land inside the bucket `[−198,720, −198,300]`, near the bottom — the new pool tick will be just above `−198,720`. Final price ≈ 2,315 USDC/ETH.

3. **Is there a cliff right above where I'd land?** Look at the next row's ΔL. `Buy −198,300` has `ΔL = −4.55e7` — a big drop in active liquidity right after our landing point. So if you trade 5 ETH and someone else immediately trades another 5 ETH right behind you, *their* slippage will be much worse than yours. This is the kind of insight an order-book trader would get from "what's two levels deep" — you get it from reading one row ahead.

4. **What's the bid-ask asymmetry?** Compare `Buy 5` slip (+0.73 %) to `Sell 5` slip (−0.81 %). Sells are slightly worse here, meaning more of the *current* depth sits above current price than below. LPs are skewed bullish (or somebody just sold a lot of ETH and rebalanced upward).

This is the same intuition you'd build by staring at a CLOB; you're just reading it off a structurally-different book.

---

## 4. Caveats

- **Hooked pools.** If the pool has a hook that mutates the fee in `beforeSwap` (e.g. the ACDF v2 hook here), the depth math uses the static fee from `slot0.lpFee` and may under- or over-state the true cost. The CLI prints a warning footer when this applies. For a hook-correct quote, use `univ4 quote` (which calls the on-chain Quoter and is bit-perfect for any hook).
- **Snapshot in time.** Depth changes whenever an LP mints/burns a position, even between blocks via JIT. The numbers reflect the latest block when you ran the command.
- **Window only.** Only initialized ticks within `--range-bps` of current are shown. Liquidity beyond that exists; it's just not loaded. Widen `--range-bps` (and bump `--rows`) if your trade size could push past the displayed window.
- **Reorg risk.** All numbers come from the latest block; reorgs can re-order or cancel LP mints/burns. For large trades, re-run immediately before sending.

## 5. Further reading

- [`docs/TICKS_AND_PRICES.md`](TICKS_AND_PRICES.md) — Uniswap's tick math from first principles.
- [`docs/UNISWAP_V4_GUIDE.md`](UNISWAP_V4_GUIDE.md) — V4 protocol overview.
- Uniswap V3 whitepaper (Adams et al., 2021) — the canonical reference for the math; V4 inherits the same liquidity model.
- [v4-core source](https://github.com/Uniswap/v4-core) — `TickMath.sol`, `SqrtPriceMath.sol`, `SwapMath.sol` are the libraries this CLI's `swap_math.py` ports from.
