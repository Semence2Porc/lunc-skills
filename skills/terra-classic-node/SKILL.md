# Terra Classic Node / Network Constants

## What This Is
Connection and network facts for building against the Terra Classic (columbus-5)
chain: verified endpoints, key constants, module routes, and the pitfalls that come
from the chain's non-standard module paths and changed response schemas.

## Prerequisites
- Any HTTP client. No SDK is required to read chain state — every fact below is a GET away.

## IDs & Endpoints
| Thing | Value | Verified |
|---|---|---|
| chain-id | `columbus-5` | 2026-09-13 |
| LCD (public) | `https://terra-classic-lcd.publicnode.com` | 2026-09-13 |
| LCD (Keplr) | `https://lcd-columbus.keplr.app` | 2026-09-13 |
| RPC (Keplr) | `https://rpc-columbus.keplr.app` | 2026-09-13 |
| coinType | `330` (all registries agree) | 2026-09-13 |
| decimals | `6` (`uluna`); **not** 8 like ICP | 2026-09-13 |
| bech32 HRPs | `terra1…` (accounts), `terravaloper1…` (validators) | 2026-09-13 |
| gas price floor (uluna) | `28.325` | 2026-09-13 |
| burn tax rate | `0.015` (1.5%) — see Mistake 2 | 2026-09-13 |
| gov: quorum / threshold / veto | `0.40` / `0.50` / `0.334` | 2026-09-13 |
| gov: min deposit | `5000000000000 uluna` = 5,000,000 LUNC | 2026-09-13 |
| gov: max deposit period | `1209600s` (14 days) | 2026-09-13 |

## Mistakes That Break Your Build

1. **Tax params live at `/terra/...`, not `/cosmos/...`.** The x/tax module is a
   Terra fork, not a Cosmos SDK module. `GET /cosmos/tax/v1beta1/params` returns
   `"not implemented"` (code 12) on every node; `GET /terra/tax/v1beta1/params`
   works. Both checked on two independent LCDs, 2026-09-13.

2. **The tax-params response schema changed.** Older docs describe
   `{ tax_rate, tax_caps, split }`. The live response today is
   `{ gas_prices: [...], burn_tax_rate: "0.015..." }` — 23 fiat/uluna gas-price
   entries. Do not hardcode the old field names; parse defensively and check the
   `Verify` section below for the current shape.

3. **Gas-price mismatches reject transactions.** The minimum gas price per denom is
   part of the tax params (`gas_prices`); `uluna` is `28.325`. Under-pricing gas in
   a broadcast fails node-side. Wallets hardcode this exact value (the Keplr chain
   registry entry matches it to the decimal — checked 2026-09-13).

4. **Gov params responses are inconsistent per-route.** The combined
   `/cosmos/gov/v1beta1/params` returns zero-valued tallies on some nodes while the
   split routes (`/params/tallying`, `/params/deposit`) return the real values.
   Query the split routes.

5. **Decimals are 6, not 8.** Every registry (cosmos/chain-registry, trustwallet,
   keplr-chain-registry) agrees. Display/parse bugs here are cosmetic-grade but
   ubiquitous; multiply/divide by 10^6 for LUNC <-> uluna.

6. **Passed governance proposals are not shipped features.** Proposals can pass and
   stay unimplemented for months (e.g. the 2022-era on-chain repeg family).
   Before building on a passed proposal, check for a matching upgrade height or
   core PR.

## Verify
```bash
# Tax params + current burn rate (Mistakes 1–2).
# NOTE the parens: jq's comma applies the following filter to BOTH branches, and
# `length` on a STRING is its character count — without them you'd print the
# rate's length (19) instead of the rate.
curl -s https://terra-classic-lcd.publicnode.com/terra/tax/v1beta1/params \
  | jq '.params.burn_tax_rate, (.params.gas_prices | length)'

# Gov tally params (Mistake 4)
curl -s https://terra-classic-lcd.publicnode.com/cosmos/gov/v1beta1/params/tallying

# Node is alive and at height
curl -s https://rpc-columbus.keplr.app/status | jq '.result.sync_info.latest_block_height'
```
If `burn_tax_rate` or the `gas_prices` shape differs from the table above, this
skill is stale — trust the endpoint, not this file, and consider PRing the fix.

## Implementation
Read tax params and compute the burn tax for an amount, defensively:

```js
const LCD = "https://terra-classic-lcd.publicnode.com";

export async function taxParams() {
  const p = (await (await fetch(`${LCD}/terra/tax/v1beta1/params`)).json()).params;
  // Schema-tolerant: the rate field has changed names before (Mistake 2).
  const rate = p.burn_tax_rate ?? p.tax_rate;
  if (rate === undefined) throw new Error("tax params: no rate field — schema changed");
  return { burnTaxRate: rate, gasPrices: p.gas_prices ?? [] };
}

export function burnTax(amountUluna, rate) {
  // rate is a Cosmos Dec string (10^-18 fixed point) — parse it on its own
  // terms: `BigInt("015")` = 15, NOT 0.015e18, and an integer rate ("1") has
  // no fraction part at all. Scale once, then floor the tax.
  const [i, f = ""] = rate.split(".");
  const scaled = BigInt((i || "0") + f.padEnd(18, "0").slice(0, 18)); // 10^-18 fixed point
  return ((BigInt(amountUluna) * scaled) / 1_000_000_000_000_000_000n).toString();
}
```
