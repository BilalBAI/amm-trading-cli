# CLI Reference

Full command reference for the `amm-trading` CLI.

```bash
source venv/bin/activate   # or: source activate.sh
amm-trading --help
```

---

## General

```bash
# Query ETH and token balances (defaults to wallet.env address)
amm-trading query balances
amm-trading query balances --address 0x123...

# Wrap ETH to WETH
amm-trading wrap 0.1

# Unwrap WETH to ETH
amm-trading unwrap 0.1

# Generate a new wallet (no RPC needed)
amm-trading wallet generate
amm-trading wallet generate --accounts 5
```

---

## Uniswap V3

V3 liquidity, calculate, and lp-quote commands use **pool names** from `config/uniswap_v3/pools.json`. Each pool entry contains `address`, `token0`, `token1`, `fee`, and `tickSpacing`.

### Query (read-only, no wallet needed)

```bash
# Pools
amm-trading univ3 query pools
amm-trading univ3 query pools --address 0x4e68Ccd3E89f51C3074ca5072bbAC773960dFa36
amm-trading univ3 query pools --refresh-cache

# Positions
amm-trading univ3 query position 1157630          # by NFT token ID
amm-trading univ3 query positions                 # all positions for wallet
amm-trading univ3 query positions --address 0x123...

# Swap quote
amm-trading univ3 quote WETH USDT WETH_USDT_30 0.1

# LP quote — how many tokens do I need?
amm-trading univ3 lp-quote WETH_USDT_30 -0.05 0.05 --amount0 0.1
amm-trading univ3 lp-quote WETH_USDT_30 -0.05 0.05 --amount1 300

# Calculate optimal amounts (tick-based)
amm-trading univ3 calculate amounts WETH_USDT_30 -887220 887220 --amount0 0.1

# Calculate optimal amounts (percentage range)
amm-trading univ3 calculate amounts-range WETH_USDT_30 -0.05 0.05 --amount0 0.1
amm-trading univ3 calculate amounts-range WETH_USDT_30 -0.05 0.05 --amount1 300
```

### Liquidity

```bash
# Add liquidity (tick range) — pool config comes from pools.json
amm-trading univ3 add WETH_USDT_30 -887220 887220 0.1 300
amm-trading univ3 add WETH_USDT_30 -887220 887220 0.1 300 --slippage 1.0

# Add liquidity (percentage range around current price)
amm-trading univ3 add-range WETH_USDT_30 -0.05 0.05 0.1 300    # -5% to +5%
amm-trading univ3 add-range WETH_USDT_30 -0.10 -0.01 0.1 300   # below current price
amm-trading univ3 add-range WETH_USDT_30 0.01 0.10 0.1 300     # above current price

# Remove liquidity
amm-trading univ3 remove 1157630 50 --collect-fees
amm-trading univ3 remove 1157630 100 --collect-fees --burn

# Migrate liquidity to a new range
amm-trading univ3 migrate 1157630 -887220 887220
amm-trading univ3 migrate 1157630 -887220 887220 --percentage 50
amm-trading univ3 migrate 1157630 -887220 887220 --slippage 1.0 --no-collect-fees --burn-old
```

### Swap

```bash
amm-trading univ3 swap WETH USDT WETH_USDT_30 0.1
amm-trading univ3 swap WETH USDT WETH_USDT_30 0.1 --slippage 100   # 100 bps = 1%
amm-trading univ3 swap WETH USDT WETH_USDT_30 0.1 --deadline 60    # 60 minutes
amm-trading univ3 swap WETH USDT WETH_USDT_30 0.1 --dry-run        # simulate only
```

---

## Uniswap V4

V4 supports **native ETH** directly — use `ETH` as a token symbol. No WETH wrapping needed.

V4 liquidity and calculate commands use **pool names** from `config/uniswap_v4/pools.json`. Each pool entry contains `currency0`, `currency1`, `fee`, `tickSpacing`, `hooks`, and (for hooked pools) `hook_type` — no need to pass these individually. `hook_type` is required whenever `hooks` is non-zero; see [Hook management](#hook-management--univ4-hook-hook_id-op) for the list of registered hook ids.

### Query (read-only, no wallet needed)

```bash
# Pools
amm-trading univ4 query pools
amm-trading univ4 query pools --name ETH_USDC_30

# Positions
amm-trading univ4 query position 12345
amm-trading univ4 query positions
amm-trading univ4 query positions --address 0x123...

# Swap quote
amm-trading univ4 quote ETH USDC ETH_USDC_30 1.0

# Calculate optimal amounts (tick-based)
amm-trading univ4 calculate amounts ETH_USDC_30 -887220 887220 --amount0 1.0

# Calculate optimal amounts (absolute price range)
amm-trading univ4 calculate amounts-price-range ETH_USDC_HOOKED 2000 2400 --amount1 511.51

# Calculate optimal amounts (percentage range)
amm-trading univ4 calculate amounts-pct-range ETH_USDC_30 -0.05 0.05 --amount0 1.0
```

### Liquidity

Three ways to specify the price range when adding liquidity:

```bash
# 1. Add liquidity by tick range — pool config comes from pools.json
amm-trading univ4 add ETH_USDC_30 -887220 887220 1.0 2000
amm-trading univ4 add ETH_USDC_HOOKED -887220 887220 1.0 2000    # hook pool

# 2. Add liquidity by absolute price range (token1/token0)
amm-trading univ4 add-range ETH_USDC_30 2000 2400 1.0 2200       # price 2000-2400 USDC/ETH
amm-trading univ4 add-range ETH_USDC_HOOKED 1800 2200 0.5 1000   # hook pool

# 3. Add liquidity by percentage range around current price
amm-trading univ4 add-pct-range ETH_USDC_30 -0.05 0.05 1.0 2000  # -5% to +5%
amm-trading univ4 add-pct-range ETH_USDC_30 -0.10 -0.01 1.0 2000 # below current price

# Remove liquidity
amm-trading univ4 remove 12345 100 --collect-fees --burn
```

### Rebalance — `univ4 rebalance`

Atomically move an out-of-range position to a new tick range in **one transaction**. Replaces the manual 3-step process (`query` → `remove` → `add`) and avoids the price-drift window between separate txs.

```bash
# Dry-run first — prints the plan without sending the tx
amm-trading univ4 rebalance <token_id> --width-bps 500 --gap-bps 0 --percent 100 --dry-run

# Full rebalance: move all liquidity, leave the old NFT empty
amm-trading univ4 rebalance <token_id> --width-bps 60

# Full rebalance + burn the now-empty old NFT in the same tx
amm-trading univ4 rebalance <token_id> --width-bps 60 --burn-old

# Partial: move 50%, old position keeps the rest at its original range
amm-trading univ4 rebalance <token_id> --width-bps 120 --percent 50

# Push the new range further from current (limit-order-style placement)
amm-trading univ4 rebalance <token_id> --width-bps 60 --gap-bps 60
```

Accrued fees always go to your wallet — they are not compounded into the new position.

**What it does** — single `PositionManager.modifyLiquidities` call with these actions:
```
DECREASE_LIQUIDITY (percent% of L)
[BURN_POSITION]                       ← only with --burn-old, requires --percent 100
MINT_POSITION (new range)
TAKE_PAIR  (sweeps fees + dust to wallet)
SETTLE_PAIR (settles any rounding deficit)
```

V4's flash accounting nets all token movements internally — recovered principal flows from old position → new position with no actual transfer, only accrued fees materialize as a wallet credit.

**Direction is auto-inferred** from the position's current composition:
- All in token1 (e.g., USDC) → range placed **below** current tick
- All in token0 (e.g., ETH) → range placed **above** current tick
- In-range (both tokens) → **rejected** with an error (no swap support; use `remove` + `add`)

**Argument reference:**

| Flag | Default | Meaning |
|---|---|---|
| `--width-bps N` | required | Width of new range in basis points (1 bp ≈ 1 tick). Snapped UP to the pool's tickSpacing. For tickSpacing 60: `--width-bps 50` becomes 60 ticks. |
| `--percent N` | 100 | Fraction of liquidity to move (1–100). Below 100, the old NFT keeps the leftover at its original range; a new NFT is minted for the moved portion. |
| `--gap-bps N` | 0 | Distance the new range's near edge is pushed *further from current*, beyond the nearest valid tick. 0 = the range starts at `nearest_below` (for `below`) or `nearest_above` (for `above`). Always snapped DOWN to a tickSpacing multiple regardless of direction — sub-spacing values like `--gap-bps 30` (with spacing 60) collapse to 0. |
| `--burn-old` | off | Burn the empty old NFT in the same tx. Only valid with `--percent 100`. Saves ~$0.20 of follow-up gas. |
| `--slippage-bps N` | 50 | Buffer above the computed mint amount maxes. |
| `--dry-run` | off | Print the plan without sending the tx. |

**Sample dry-run output:**

```
Position 225260 — ETH/USDC
  current_tick: -198840    in_range: False
  old_range:    [-199380, -198840]      old_liquidity: 6239320492464
  composition:  0.000000 ETH,  7.999999 USDC
  inferred direction: below (single-sided in USDC)

Plan:
  percent moved:        50%
  width:                540 ticks (requested 500 bps)
  gap:                  60 ticks (requested 60 bps)
  new_range:            [-199440, -198900]
  new_liquidity:        3129032027909
  principal recovered:  0.000000 ETH,  3.999999 USDC
  new mint consumes:    0.000000 ETH,  3.999998 USDC
  fees accrued:         0.000000 ETH,  0.000000 USDC  → wallet

=== DRY RUN — no transaction sent ===
```

**Cost** — typically ~270k–320k gas for the modifyLiquidities tx, plus ~30k for a Permit2 approval refresh if 1h has elapsed. At current gas rates: **~$0.30–0.40 total** vs ~$0.85–1.10 for the manual remove+add flow.

**Discovery picks up the new NFT automatically** (within ~5 min of the rebalance). No `position_monitor.json` edit needed.

**Limitations:**
- Does NOT swap. Both directions assume single-sided funding from the principal you already have. If you want to flip from one token to the other, do that as a separate `swap` first.
- "Above" direction with native-ETH currency0 is rejected (can't reliably settle native deficit via Permit2). Use `remove` + `add` for that case.
- Old NFT is left empty (zero liquidity) unless `--burn-old`. The empty NFT costs ~0 to keep but ~$0.20 to burn later if you decide to clean up.

### Swap

```bash
amm-trading univ4 swap ETH USDC ETH_USDC_30 1.0
amm-trading univ4 swap ETH USDC ETH_USDC_30 1.0 --slippage 100
amm-trading univ4 swap ETH USDC ETH_USDC_30 1.0 --dry-run
```

### Hook management — `univ4 hook <hook_id> <op>`

Hook operations are **namespaced by hook type**. Commands are dispatched via a registry, so the operation surface for each hook matches its on-chain ABI exactly.

**Registered hook types:**

| `hook_id` | Contract | Key features |
|---|---|---|
| `acdf-v1` | AccessControlDynamicFeeHook V1 | Single LP fee, whitelisted providers, trusted periphery routers, owner-only writes |
| `acdf-v2` | AccessControlDynamicFeeHook V2 | Directional fees (0→1 and 1→0), delegated fee managers, two-step ownership transfer |

All write commands are **owner-only** unless noted otherwise.

#### `acdf-v1` — single-fee dynamic fee hook

```bash
# Read hook state (no wallet required)
amm-trading univ4 hook acdf-v1 info --hook 0xYourHook

# Set LP fee (single rate, both directions). Owner only.
amm-trading univ4 hook acdf-v1 set-fee --hook 0xYourHook --fee 500    # 0.05%
amm-trading univ4 hook acdf-v1 set-fee --hook 0xYourHook --fee 3000   # 0.30%
amm-trading univ4 hook acdf-v1 set-fee --hook 0xYourHook --fee 10000  # 1.00%

# Liquidity provider whitelist
amm-trading univ4 hook acdf-v1 add-provider    --hook 0xYourHook --provider 0xLP
amm-trading univ4 hook acdf-v1 remove-provider --hook 0xYourHook --provider 0xLP

# Trusted periphery routers (e.g. V4 PositionManager)
amm-trading univ4 hook acdf-v1 add-router    --hook 0xYourHook --router 0xPositionManager
amm-trading univ4 hook acdf-v1 remove-router --hook 0xYourHook --router 0xPositionManager

# Transfer ownership (one-step, Ownable)
amm-trading univ4 hook acdf-v1 transfer-ownership --hook 0xYourHook --to 0xNewOwner
```

#### `acdf-v2` — directional-fee hook with delegated fee management

```bash
# Read hook state — shows fee_0_for_1, fee_1_for_0, pending_owner, fee managers
amm-trading univ4 hook acdf-v2 info --hook 0xYourHook

# Symmetric fee (sets both directions to the same value). Owner OR fee manager.
amm-trading univ4 hook acdf-v2 set-fee --hook 0xYourHook --fee 500

# Asymmetric fee (e.g. higher fee on toxic/arb flow). Owner OR fee manager.
#   zeroForOne = token0 → token1 (for ETH/USDC: sell ETH direction)
#   oneForZero = token1 → token0 (for ETH/USDC: buy ETH direction)
amm-trading univ4 hook acdf-v2 set-directional-fee \
    --hook 0xYourHook --fee-0for1 1000 --fee-1for0 500

# Liquidity provider whitelist (same as V1)
amm-trading univ4 hook acdf-v2 add-provider    --hook 0xYourHook --provider 0xLP
amm-trading univ4 hook acdf-v2 remove-provider --hook 0xYourHook --provider 0xLP

# Trusted periphery routers (same as V1)
amm-trading univ4 hook acdf-v2 add-router    --hook 0xYourHook --router 0xPositionManager
amm-trading univ4 hook acdf-v2 remove-router --hook 0xYourHook --router 0xPositionManager

# Fee-manager delegation — let another address call set-fee / set-directional-fee
amm-trading univ4 hook acdf-v2 add-fee-manager    --hook 0xYourHook --manager 0xBot
amm-trading univ4 hook acdf-v2 remove-fee-manager --hook 0xYourHook --manager 0xBot

# Two-step ownership transfer (Ownable2Step)
amm-trading univ4 hook acdf-v2 transfer-ownership --hook 0xYourHook --to 0xNewOwner
# Then the recipient runs (from their wallet):
amm-trading univ4 hook acdf-v2 accept-ownership  --hook 0xYourHook
```

### Pool initialization — `univ4 pool init-dynamic-fee-hook`

`--hook-type` is **required** and must match a registered hook id (currently `acdf-v1` or `acdf-v2`). The value is validated against the hook address's permission bits, and then written into the new entry in `config/uniswap_v4/pools.json` so all downstream commands (fee display, CLI routing) know how to talk to this hook.

```bash
amm-trading univ4 pool init-dynamic-fee-hook \
    --token0 ETH --token1 USDC \
    --hook         0xYourHook \
    --hook-type    acdf-v1 \
    --price        2000 \
    --tick-spacing 60 \
    --name         ETH_USDC_HOOKED
```

> Uses `fee=0x800000` (DYNAMIC_FEE_FLAG). The PoolManager emits an Initialize event, making the pool discoverable by UniswapX and 1inch. The pool entry written to `pools.json` includes `hook_type` — do not hand-edit pools to add a hook without this field, or downstream commands will fail with a `ConfigError`.

---

## End-to-end workflow — deploy a hook and go live

> **Requires Foundry** for hook deployment. Commands assume `wallet.env` has `RPC_URL`, `PRIVATE_KEY`, and (optionally) `SAFE_ADDRESS` set.

The example uses V2 (`acdf-v2`). For V1, substitute `acdf-v1` and use the V1-specific ops from the table above.

```bash
# Step 1: Deploy the hook (Foundry — from the univ4-acdf-hook repo)
export POOL_MANAGER_ADDRESS=0x000000000004444c5dc75cB358380D2e3dE08A90
export INITIAL_FEE=3000
forge script contracts/script/DeployAccessControlDynamicFeeHook.s.sol \
    --rpc-url $RPC_URL --broadcast --private-key $PRIVATE_KEY -vvvv
# → prints: Hook address: 0xAbCd...0880   (must end in bits 0x0880)

# Step 2: Initialize the pool with the deployed hook
amm-trading univ4 pool init-dynamic-fee-hook \
    --token0 ETH --token1 USDC \
    --hook         0xAbCd...0880 \
    --hook-type    acdf-v2 \
    --price        2000 \
    --name         ETH_USDC_HOOKED
# Writes ETH_USDC_HOOKED entry to config/uniswap_v4/pools.json with hook_type: acdf-v2

# Step 3 (optional): Register the V4 PositionManager as a trusted router so the
# Python LP flow encodes the original caller in hookData correctly.
amm-trading univ4 hook acdf-v2 add-router \
    --hook 0xAbCd...0880 \
    --router 0x<V4_POSITION_MANAGER_ADDRESS>

# Step 4 (optional): Whitelist liquidity providers.
amm-trading univ4 hook acdf-v2 add-provider --hook 0xAbCd...0880 --provider 0xSomeLP

# Step 5 (optional for V2): Delegate fee management to a bot key.
amm-trading univ4 hook acdf-v2 add-fee-manager --hook 0xAbCd...0880 --manager 0xBotKey

# Step 6: Add liquidity (pool name resolves via pools.json — hook/hook_type/ticks automatic)
amm-trading univ4 add-range ETH_USDC_HOOKED 1800 2200 1.0 2000

# Step 7: Adjust fees at any time.
amm-trading univ4 hook acdf-v2 set-fee            --hook 0xAbCd...0880 --fee 500
amm-trading univ4 hook acdf-v2 set-directional-fee --hook 0xAbCd...0880 --fee-0for1 1000 --fee-1for0 500

# Step 8: Remove liquidity when done.
amm-trading univ4 remove <token_id> 100 --collect-fees --burn
```

### Third-party and pre-existing hooked pools

`amm-trading` can only manage hooks that are **registered in the hooks registry** and tagged in `pools.json` with a known `hook_type`. If you add a pool by hand whose `hook_type` is missing or unknown, `univ4 query pools` and `univ4 query position` will fall back to a generic `"dynamic (hook-controlled)"` fee string, and `univ4 hook <id> ...` commands will reject the pool. This is deliberate — it prevents silent misconfiguration.

---

## Kill-switch — `kill-switch`

> ⚠️ **EMERGENCY USE ONLY.** Withdraws 100% of liquidity from every position you list, in parallel, with no chance to cancel mid-flight once `--confirm` is passed. Do **not** use as a routine cleanup tool — use `univ3 remove` / `univ4 remove` for that.
>
> Use it when something is going wrong and you need to be flat across all LP exposure as fast as possible: pool exploit suspected, hook misbehaving, oracle anomaly, gov-attack window, key rotation under threat, etc.

This command **does not discover positions**. The caller supplies a JSON file listing exactly which positions to drain. Pair it with `univ3 query positions` / `univ4 query positions` (or your monitor's discovery cache) to build the list ahead of time.

```bash
# Preview — reads positions, fetches their current liquidity, prints what would be drained, exits.
amm-trading kill-switch --positions positions.json

# Execute — broadcast withdrawals for every listed position in parallel.
amm-trading kill-switch --positions positions.json --confirm

# Higher gas-tip multiplier (default 2.0×) for faster inclusion on a congested mempool.
amm-trading kill-switch --positions positions.json --confirm --gas-multiplier 3.0
```

### Positions JSON format

```json
{
  "positions": [
    {"protocol": "v4", "token_id": 12345},
    {"protocol": "v4", "token_id": 67890},
    {"protocol": "v3", "token_id": 1157630}
  ]
}
```

Both protocols can be mixed in a single file. `protocol` must be `"v3"` or `"v4"`; `token_id` is the NFT id from the corresponding PositionManager.

A bare top-level list (without the `"positions"` wrapper) is also accepted for convenience:

```json
[
  {"protocol": "v4", "token_id": 12345},
  {"protocol": "v3", "token_id": 1157630}
]
```

### Argument reference

| Flag | Default | Meaning |
|---|---|---|
| `--positions PATH` | required | Path to the JSON file listing positions to drain. |
| `--confirm` | off | **Required** to actually broadcast. Without it, only a preview is printed. The mandatory two-step (preview → confirm) is the safety net — read the preview before passing this. |
| `--gas-multiplier N` | 2.0 | Multiplier applied to your configured `maxFeePerGas` and `maxPriorityFeePerGas` (from `config/gas.json`). Higher = faster inclusion at higher cost. Use 3.0–5.0 in genuine emergencies on a congested mempool. |

### What it does

1. Reads the positions JSON; rejects malformed entries early.
2. Fetches the current `liquidity` for each position; **drops empties** so you don't waste gas on already-drained NFTs (re-runs after partial failure are therefore free of dead-weight calls).
3. Without `--confirm`: prints a preview list and exits. **Nothing is broadcast.**
4. With `--confirm`:
   - Reads current `pending` nonce, assigns sequential nonces.
   - Builds + locally signs one tx per position with the boosted gas params:
     - **V4** → a single `modifyLiquidities` call encoding `DECREASE_LIQUIDITY(100%) + TAKE_PAIR`. Fees come back in the same call.
     - **V3** → a `multicall(decreaseLiquidity, collect)` so principal **and** fees land in the wallet in one tx.
   - Broadcasts all raw txs in parallel via `eth_sendRawTransaction` (up to 16 in flight).
   - Waits for all receipts in parallel.
   - Prints a result table and saves a JSON summary to `results/kill_switch_<addr_prefix>.json`.

### What it does NOT do

- ❌ **Does not discover positions.** The caller supplies the list. This is intentional: a robust LP monitor (or a manual `univ4 query positions` snapshot you take ahead of time) is the right source of truth — block-history scans miss positions on pruned RPCs.
- ❌ **Does not burn empty NFTs.** After kill-switch, your positions hold zero liquidity but the NFTs still exist. Clean up later with `univ4 remove --burn` / `univ3 remove --burn`. Burn is intentionally skipped because it adds a second tx per position (~2× the gas, ~2× the latency) for no emergency benefit — your funds are already out.
- ❌ **Does not support Safe mode.** Kill-switch requires direct signing — Safe proposals need owner approval and would defeat the "fast" requirement. If `SAFE_ADDRESS` is set in `wallet.env`, the command exits with an error. Pass `--direct` (or unset `SAFE_ADDRESS`) to override.
- ❌ **Does not swap or rebalance.** It only decreases liquidity. The recovered tokens land in your wallet in their original pool composition (e.g., out-of-range positions return purely token0 or purely token1).

### Operational notes

- **Concurrency:** Up to 16 raw txs broadcast in parallel. Sequential nonces ensure the chain orders them correctly even though they arrive at the RPC out of order.
- **Cost (per position):** V4 ~150–200k gas; V3 ~250–300k gas (decrease + collect via multicall). With `--gas-multiplier 2.0` against a 1.0 gwei `maxFeePerGas`, that's roughly 0.0003–0.0006 ETH per position. Multiply by your live mainnet rate.
- **Failure handling:** Each position is independent — one revert doesn't block the rest. The summary marks per-position outcomes (`success` / `reverted` / `timeout` / `no-bcast`). If any position fails, the command exits with status `1`.
- **Idempotent on re-run:** Already-empty positions are filtered out by the live-liquidity check, so re-running kill-switch after a partial failure only retries the still-funded positions.

### Building the positions file

The simplest approach is to run the discovery commands ahead of time and convert the saved results:

```bash
# Snapshot current positions (no wallet needed beyond an address)
amm-trading univ4 query positions
amm-trading univ3 query positions

# Then build positions.json from the resulting:
#   results/univ4_positions_<address>.json
#   results/positions_<address>.json
```

Or maintain a hand-curated list of positions you care about — for an emergency tool, having an authoritative list you trust is more valuable than relying on on-the-fly RPC discovery.

### When you should NOT use kill-switch

- Closing a single position → use `univ4 remove <token_id> 100 --burn` / `univ3 remove <token_id> 100 --burn`.
- Routine portfolio rebalancing → use `univ4 rebalance` or `remove` + `add`.
- Multisig-controlled funds → use `univ4 remove` / `univ3 remove` per-position via Safe; kill-switch refuses Safe mode.
- "I want to clean up empty NFTs" → use `remove --burn`; kill-switch never burns.

---

## Safe Wallet (Multisig)

The CLI supports [Safe{Wallet}](https://safe.global/) multisig integration. When enabled, fund-touching operations (swap, add/remove liquidity, wrap/unwrap) are **proposed** to the Safe Transaction Service instead of signed directly. A Safe owner then approves the transaction via the Safe{Wallet} mobile app.

Hook management, pool initialization, and fee changes are **not** routed through Safe — they don't touch LP funds.

### Setup

1. Add your Safe address to `wallet.env`:
   ```
   SAFE_ADDRESS=0x5bd19Ea9E14205Bce413994D2640E4e9fb204DD3
   ```

2. Register the hot wallet as a delegate (one-time):
   ```bash
   amm-trading safe register-delegate
   ```

3. Use commands normally — proposals are sent to Safe instead of executing on-chain:
   ```bash
   amm-trading univ4 swap ETH USDC ETH_USDC_30 1.0
   # → SAFE PROPOSAL: Swap 1.0 ETH -> USDC (V4)
   #   Approve in Safe{Wallet} app: https://app.safe.global/transactions/tx?safe=...
   ```

### Commands

```bash
# Show Safe info (address, threshold, owners, delegates)
amm-trading safe info

# Register hot wallet as delegate/proposer
amm-trading safe register-delegate
amm-trading safe register-delegate --label my-bot
```

### Direct Mode

Bypass Safe mode for testing with the `--direct` flag:

```bash
amm-trading --direct univ4 swap ETH USDC ETH_USDC_30 0.001
```

Or remove/comment out `SAFE_ADDRESS` from `wallet.env`.

---

## Gas management — `gas`

Inspect and tune EIP-1559 gas parameters in [config/gas.json](config/gas.json) without hand-editing the file.

### `gas show` — print live cost matrix

Shows your configured `maxFeePerGas` / `maxPriorityFeePerGas` against the live network base fee, then renders a per-operation cost table in both ETH and USD (USD via the same Chainlink ETH/USD feed used by the position monitor).

```bash
amm-trading gas show
```

Sample output:

```
Gas config: /…/config/gas.json
  maxFeePerGas:         2.0000 gwei
  maxPriorityFeePerGas: 0.1000 gwei

Live network state:
  Base fee (latest block): 1.7243 gwei
  Est. effective fee:      1.8243 gwei (base + priority, capped at max)
  ETH/USD (Chainlink):     $2,347.28

Operation                  Gas units       Max ETH    Max USD     Est ETH    Est USD
─────────────────────────────────────────────────────────────────────────────────────
approve                       65,000      0.000130     $0.31    0.000119     $0.28
v4_mint                      450,000      0.000900     $2.11    0.000821     $1.93
v4_swap_native               150,000      0.000300     $0.70    0.000274     $0.64
…
```

How to read the columns:

| Column | Meaning |
|---|---|
| **Max ETH / USD** | Worst case: `gas_units × maxFeePerGas`. What you'd pay if base fee spiked all the way to your `maxFeePerGas` cap. |
| **Est ETH / USD** | Typical case: `gas_units × min(base + priority, max)` using the **live** base fee. What you'd actually pay if the tx landed right now. |

Two flags worth eyeballing on every run:
- If **Est == Max**, your `maxFeePerGas` cap is being squeezed by the live base fee. Bump it (see `gas set` below) before submitting anything time-sensitive.
- If the live base fee is far below your cap, you're paying for protection against spikes that aren't currently happening — fine, just know you have headroom.

A snapshot is also saved to `results/gas_show.json` for record-keeping.

### `gas set` — update `maxFeePerGas` / `maxPriorityFeePerGas`

Update either or both fields in `config/gas.json` without touching the file by hand. The rest of the JSON (gas-limit table, comments) is preserved.

```bash
# Update both
amm-trading gas set --max-fee 3.0 --priority 0.1

# Update one
amm-trading gas set --max-fee 5.0
amm-trading gas set --priority 0.05
```

Output reports old → new for both values:

```
Updated /…/config/gas.json:
  maxFeePerGas:         2 -> 3.0 gwei
  maxPriorityFeePerGas: 0.1 -> 0.1 gwei
```

Validation:
- `--max-fee` must be `> 0`.
- `--priority` must be `>= 0`.
- Warns (but still applies) if `maxFeePerGas < maxPriorityFeePerGas` — you should fix this before submitting txs because they'd revert.

The write is atomic (`.tmp` + rename), so an interrupt mid-write can never leave a half-written `gas.json`.

---

## Fee Tiers

| Fee | Percentage | Tick Spacing | Use Case |
|-----|-----------|-------------|---------|
| `100` | 0.01% | 1 | Stable pairs |
| `500` | 0.05% | 10 | Stable / correlated pairs |
| `3000` | 0.30% | 60 | Most token pairs |
| `10000` | 1.00% | 200 | Exotic / high-volatility pairs |
| `8388608` | dynamic | set in `pools.json` | V4 hook-controlled pools (`DYNAMIC_FEE_FLAG = 0x800000`) |

---

## Slippage units

| Command group | `--slippage` unit | Example |
|---------------|-------------------|---------|
| `add`, `add-range`, `add-pct-range`, `migrate` | **percent** | `0.5` = 0.5% |
| `swap` | **basis points** | `50` = 0.5% |

---

## Caching

Pool queries cache static data (token metadata, fee tier) to reduce RPC calls.

- **V3 pool cache:** `.cache/pools/v3.json` — token metadata + fee tier per pool address
- **Token metadata cache:** `.cache/tokens.json` — ERC20 symbol/decimals (shared by V3 and V4)

V4 does not need a pool-level cache: PoolKey fields (`currency0`, `currency1`, `fee`, `tickSpacing`, `hooks`, `hook_type`) are already file-backed in `config/uniswap_v4/pools.json`, and ERC20 metadata is handled by `.cache/tokens.json`.

V3 pool cache behavior:
- **First query:** ~6 RPC calls per pool → **subsequent:** ~2 RPC calls (~70% reduction)
- Static fields cached: `pool_name`, `address`, `token0`, `token1`, `fee`, `tick_spacing`
- Dynamic fields always fresh: `current_tick`, `current_price`, `liquidity`
- Use `univ3 query pools --refresh-cache` to force-update.

---

## Output Files

All commands save JSON results to `results/` in the working directory.

| Command | Output file |
|---------|-------------|
| `query balances` | `results/balances_<address>.json` |
| `wallet generate` | `results/wallet.json` |
| `wrap` | `results/wrap_<tx_hash>.json` |
| `unwrap` | `results/unwrap_<tx_hash>.json` |
| `univ3 query pools` | `results/univ3_pools.json` |
| `univ3 query position <id>` | `results/univ3_position_<id>.json` |
| `univ3 query positions` | `results/positions_<address>.json` |
| `univ3 quote` | `results/quote_<token_in>_<token_out>.json` |
| `univ3 lp-quote` | `results/lp_quote_<token0>_<token1>.json` |
| `univ3 calculate` | `results/calculate_amounts_<token0>_<token1>.json` |
| `univ3 add` | `results/add_liquidity_<id>.json` |
| `univ3 remove` | `results/remove_liquidity_<id>.json` |
| `univ3 migrate` | `results/migrate_<old>_to_<new>.json` |
| `univ3 swap` | `results/swap_<tx_hash>.json` |
| `univ4 query pools` | `results/univ4_pools.json` |
| `univ4 query position <id>` | `results/univ4_position_<id>.json` |
| `univ4 query positions` | `results/univ4_positions_<address>.json` |
| `univ4 quote` | `results/univ4_quote_<token_in>_<token_out>.json` |
| `univ4 calculate` | `results/univ4_calculate_<token0>_<token1>.json` |
| `univ4 add` | `results/univ4_add_liquidity_<id>.json` |
| `univ4 remove` | `results/univ4_remove_liquidity_<id>.json` |
| `univ4 rebalance` (real tx) | `results/univ4_rebalance_<new_token_id>.json` |
| `univ4 rebalance --dry-run` | `results/univ4_rebalance_dryrun_<token_id>.json` |
| `kill-switch --confirm` | `results/kill_switch_<address_prefix>.json` |
| `univ4 swap` | `results/univ4_swap_<tx_hash>.json` |
| `univ4 pool init-dynamic-fee-hook` | `results/univ4_pool_init_<name>.json` |
| `univ4 hook acdf-v1 info` | `results/univ4_hook_acdf-v1_<address>.json` |
| `univ4 hook acdf-v1 set-fee` | `results/univ4_hook_acdf-v1_set_fee_<tx_hash>.json` |
| `univ4 hook acdf-v1 add-provider` | `results/univ4_hook_acdf-v1_add_provider_<tx_hash>.json` |
| `univ4 hook acdf-v1 remove-provider` | `results/univ4_hook_acdf-v1_remove_provider_<tx_hash>.json` |
| `univ4 hook acdf-v1 add-router` | `results/univ4_hook_acdf-v1_add_router_<tx_hash>.json` |
| `univ4 hook acdf-v1 remove-router` | `results/univ4_hook_acdf-v1_remove_router_<tx_hash>.json` |
| `univ4 hook acdf-v1 transfer-ownership` | `results/univ4_hook_acdf-v1_transfer_ownership_<tx_hash>.json` |
| `univ4 hook acdf-v2 info` | `results/univ4_hook_acdf-v2_info_<address>.json` |
| `univ4 hook acdf-v2 set-fee` | `results/univ4_hook_acdf-v2_set_fee_<tx_hash>.json` |
| `univ4 hook acdf-v2 set-directional-fee` | `results/univ4_hook_acdf-v2_set_directional_fee_<tx_hash>.json` |
| `univ4 hook acdf-v2 add-provider` | `results/univ4_hook_acdf-v2_add_provider_<tx_hash>.json` |
| `univ4 hook acdf-v2 remove-provider` | `results/univ4_hook_acdf-v2_remove_provider_<tx_hash>.json` |
| `univ4 hook acdf-v2 add-router` | `results/univ4_hook_acdf-v2_add_router_<tx_hash>.json` |
| `univ4 hook acdf-v2 remove-router` | `results/univ4_hook_acdf-v2_remove_router_<tx_hash>.json` |
| `univ4 hook acdf-v2 add-fee-manager` | `results/univ4_hook_acdf-v2_add_fee_manager_<tx_hash>.json` |
| `univ4 hook acdf-v2 remove-fee-manager` | `results/univ4_hook_acdf-v2_remove_fee_manager_<tx_hash>.json` |
| `univ4 hook acdf-v2 transfer-ownership` | `results/univ4_hook_acdf-v2_transfer_ownership_<tx_hash>.json` |
| `univ4 hook acdf-v2 accept-ownership` | `results/univ4_hook_acdf-v2_accept_ownership_<tx_hash>.json` |
| `gas show` | `results/gas_show.json` |
