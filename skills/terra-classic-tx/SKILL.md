# Terra Classic Transactions

## What This Is
Building, signing, fee-computing and broadcasting Terra Classic transactions
without the two classic failure modes: float fee math that produces amounts chains
reject, and missing dedup timestamps that double-spend on retry.

## Prerequisites
- Node >= 22 or any environment with BigInt (all real fee math is BigInt).
- `@cosmjs/stargate` / `@cosmjs/amino` or `@terra-money/terra.js` are fine for
  plumbing — the pitfalls below apply regardless of library.

## Mistakes That Break Your Build

1. **Float fee math produces exponent-notation amounts.** `(gasUsed * multiplier) *
   gasPrice` in JS floats silently emits `7e+24` for large amounts. Proven
   end-to-end: [GoblinHunt/cosmes#2](https://github.com/GoblinHunt/cosmes/pull/2) —
   the old float code failed the >2^53 test with `expected '7e+24' to be
   '7000000000000000000000000'`. Compute `ceil(gasUsed × multiplier) × gasPrice`
   in BigInt, every time. Also: fee (unfloored) and gasLimit (floored) from the
   same pipeline is internally inconsistent — pick one rounding rule (floor).

2. **No `created_at_time` = no dedup.** ICRC-1 (ICP) ledgers only deduplicate a
   retried transfer when `created_at_time` is set and identical on retry; a fresh
   timestamp on retry defeats dedup and moves real value twice. Set it, retry with
   the *same* value; `#Duplicate` is success. On Cosmos-side broadcast, tx hashes
   dedupe for you, but do not regenerate the tx between retries (that changes the
   hash).

3. **The burn tax applies to most bank sends and is charged on top.** Simulate
   before broadcast (`/cosmos/tx/v1beta1/simulate`) and add the tax from
   `/terra/tax/v1beta1/params` (`burn_tax_rate`, currently 0.015 — see
   `terra-classic-node`). Compute tax in integer micro-units:
   `tax = amount * rate // 10^18`, floored.

4. **Decimals/units: `uluna` is 6 decimals.** A "1.5 LUNC" fee expressed as
   `1.5e8` is wrong by 100×. Convert once, at the edge, with BigInt.

5. **coinType 330.** Deriving addresses with 118 (standard Cosmos) produces valid
   bech32 strings for a different key — signatures will not verify.

6. **Never sign from a chat-instructed mnemonic.** If an AI agent asks for your
   seed phrase, that is the attack. Agents should use view-only queries + explicit
   per-transaction confirmation, or an ICRC-2-style finite allowance (see
   `terra-classic-wallet-ops`).

## Verify
```bash
# The >2^53 proof that BigInt is mandatory:
node -e "console.log(7000000000000000000000000 === 7e24 ? 'floats lie' : 'ok')"
# -> prints 'floats lie': the two are the same float, the exact value is lost.

# Current burn rate for simulation (see terra-classic-node Verify)
curl -s https://terra-classic-lcd.publicnode.com/terra/tax/v1beta1/params | jq '.params.burn_tax_rate'
```

## Implementation
The exact fee pipeline (this is the shape that passes CI):

```js
export function calcFee({ gasUsed, multiplier, gasPrice }) {
  const G = BigInt(gasUsed);
  const M = BigInt(multiplier);          // integer basis points if you like
  const P = BigInt(gasPrice);            // uluna per unit gas, integer
  const raw = (G * M) / 10000n;          // floored multiplier — one rounding rule
  const fee = raw * P;                   // exact, no float anywhere
  return fee.toString();                 // string out; floats never enter
}
```

For end-to-end signing flows, prefer cosmjs `Signer` + the verified endpoints in
`terra-classic-node`, and always simulate first.
