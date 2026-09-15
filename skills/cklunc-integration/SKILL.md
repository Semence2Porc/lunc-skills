# CKLUNC Integration (Chain-Key LUNC on ICP)

## What This Is
CKLUNC is an ICRC-1/ICRC-2 twin of LUNC on the Internet Computer, backed 1:1 by
LUNC held in the minter's custody, with **dynamic tax parity**: LUNC's burn-tax
parameters are fetched from the chain live, and CKLUNC transfers burn on both
chains in proportion. Status: **in development** (`../cklunc/`) — this skill
documents the interface apps will code against; the ledger + tax-parity core
land first, the chain-key (threshold-ECDSA) custody last.

## Prerequisites
- ICP dev environment (`icp-cli`, mops) for building against canisters — see the
  dfinity ICP skills (`icrc-ledger`, `ckbtc`, `https-outcalls`).
- Terra Classic facts from `terra-classic-node`.

## Interface (stable from v1)
- `tax_params()` → `{ burnTaxRate, epoch, override_rate, rate_mode }` — the live
  mirrored rate (N-LCD majority, sanity-bounded 0 ≤ r ≤ 5%, fail-closed freeze
  on staleness). `rate_mode` is `parity` (mirrored rate applies), `override`
  (governed lower rate applies), or `refused` (stale params — nothing executes).
  The override can only LOWER the rate below the mirrored rate, never raise it:
  the twin must never cost more than the chain it mirrors.
- `taxed_transfer(to, amount, from_subaccount)` → pulls via ICRC-2 allowance,
  computes `tax = amount × rate // 10^18` (floored), burns the tax (ICRC transfer
  to minting account = burn), forwards `amount − tax` to `to`. Conservation:
  `net + tax = amount` exactly, integer math. **Tax parity is dynamic:** if a
  CKLUNC leg also incurred LUNC's native tax somewhere in the path, you have a
  double-charge bug — the bridge legs are tax-neutral by design.
- `burn_ledger()` → attested burn epochs: ICP-side burns linked to the LUNC
  settlement txids (batched, `memo = <epoch id>`), public reconciliation.
- `tax_stats()` → lifetime burned on ICP + matching underlying LUNC burns.
- Bridge (deposit/withdraw): tax-neutral — the LUNC leg pays LUNC's tax natively
  at broadcast; CKLUNC legs are taxed by `taxed_transfer`, never both.

## Mistakes That Break Your Build

1. **Do not use raw `icrc1_transfer` for taxed flows** — you skip the burn. Use
   `taxed_transfer` or accept that your app under-burns vs the ecosystem norm.
2. **The supply invariant is `held − burned_underlying ≥ CKLUNC supply`.** If a
   dashboard shows backing without subtracting settled+pending underlying burns,
   it is lying.
3. **Stale params freeze, they never default.** If `tax_params().epoch` is
   older than the freshness bound (ships at 24h, governance-adjustable 1h–7d),
   taxed operations halt until fresh params arrive — that is fail-closed by
   design, not an outage. Responses then carry `rate_mode: "refused"`.
4. **Decimals are 6.** CKLUNC mirrors LUNC, not ICP's 8.
5. **Agent spending still goes through finite ICRC-2 allowances** (see
   `terra-classic-wallet-ops` rule 4) — the minter never takes unlimited
   approvals; `expires_at` is always set.
6. **Transfers never make HTTP outcalls.** The rate is pushed by the params
   feed and cached; taxed transfers are pure integer math. Cross-chain calls
   cost cycles only at feed refresh, not per transfer.

## Hosting your LUNC dapp the same way
The lunc-skills site and CKLUNC itself are deployed as ICP static-site
canisters — see `icp-dapp-hosting` in this collection for the recipe.

## Verify
```bash
# The mirrored facts, from the source chain:
curl -s https://terra-classic-lcd.publicnode.com/terra/tax/v1beta1/params | jq '.params.burn_tax_rate'
# CKLUNC ledger metadata once deployed (decimals must be 6):
# icp canister call <cklunc-ledger> icrc1_metadata '()'
```

## Implementation
Consumer shape (once deployed):

```js
import { Actor, HttpAgent } from "@icp-sdk/core/agent";

const cklunc = Actor.createActor(idlFactory, {
  agent: await HttpAgent.create(),
  canisterId: CKLUNC_MINTER_ID,            // published at deploy; wasm hash pinned
});

const params = await cklunc.tax_params();   // { burnTaxRate, fetchedAt, epoch }
const out = await cklunc.taxed_transfer({ to: recipient, amount: 1_000_000n, from_subaccount: [] });
// out: { net: 985_000n, tax: 15_000n, burnEpoch: 42 }
```
