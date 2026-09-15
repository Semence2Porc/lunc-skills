# Terra Classic Market Data (Agent-Safe Reads)

## What This Is
The economic read-surface an AI agent needs on Terra Classic — tax and burn
state, supplies, swap rates, oracle health — with the pitfalls that make naive
reads wrong (schema drift, staleness, mis-scaled denoms).

## Prerequisites
- Read-only LCD access (`terra-classic-node`). Everything here is safe: no
  signing, no keys.

## Mistakes That Break Your Build

1. **Tax params route + schema drift.** `/terra/tax/v1beta1/params` (NOT
   `/cosmos/...`) and today's fields are `gas_prices` + `burn_tax_rate` — the
   legacy `tax_rate`/`tax_caps`/`split` names are gone from the live response.
   Parse defensively; see `terra-classic-node` Mistake 2.

2. **Burn statistics have multiple "burn" definitions.** The community burn
   wallet, the on-chain burn module, and tax-burn remittances are different
   numbers that blogs conflate. Attribute every figure to its exact address or
   module account, and date it.

3. **Oracle health is missed-vote counts, not price freshness alone.** A feeder
   can be fresh and still failing quorum. Check both the price and the
   validator's vote-period participation before judging "oracle is down."

4. **The market-module swap endpoint can return success with amount 0.**
   `GET /terra/market/v1beta1/swap?offer_coin=1000000uluna&ask_denom=uusd`
   returned `{"return_coin":{"denom":"uusd","amount":"0"}}` — HTTP 200, zero
   value — while `ask_denom=ustc` errors outright. Do not treat `amount: "0"`
   as a price, and do not build on this route: use the oracle's
   `/terra/oracle/v1beta1/denoms/exchange_rates` (20 denoms live, 2026-09-13)
   for prices. Never mix oracle and CEX sources in one computation.

5. **6 decimals everywhere; big numbers need BigInt.** Burn stats exceed 2^53
   uluna on this chain. Float math on totals silently corrupts the last digits
   (see `terra-classic-tx` Mistake 1).

## Verify
```bash
curl -s https://terra-classic-lcd.publicnode.com/terra/tax/v1beta1/params | jq '.params.burn_tax_rate'
curl -s "https://terra-classic-lcd.publicnode.com/cosmos/bank/v1beta1/supply" | jq '.supply[] | select(.denom=="uluna")'
# The live price surface (the market/swap route is dead — Mistake 4):
curl -s "https://terra-classic-lcd.publicnode.com/terra/oracle/v1beta1/denoms/exchange_rates" | jq '.exchange_rates | length'
```

## Implementation
Burn-rate time series entry point (pairs with [`lunc-mcp`](https://github.com/Semence2Porc/lunc-mcp)'s accumulating dataset):

```js
export async function burnSnapshot(lcd) {
  const [params, supply] = await Promise.all([
    fetch(`${lcd}/terra/tax/v1beta1/params`).then(r => r.json()),
    fetch(`${lcd}/cosmos/bank/v1beta1/supply?pagination.limit=1000`).then(r => r.json()),
  ]);
  const uluna = supply.supply.find(s => s.denom === "uluna");
  return {
    ts: Date.now(),
    burnTaxRate: params.params.burn_tax_rate,     // schema-tolerant parse
    ulunaSupplyRaw: uluna?.amount ?? null,        // BigInt in consumers, never Number
  };
}
```
Append-only series + rate-of-change over a window is all an agent needs for
"how fast is LUNC burning" — and it is computable from public endpoints alone.
