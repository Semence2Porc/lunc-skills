# Terra Classic Indexing & Accounting

## What This Is
Reading Terra Classic state into ledgers, CSVs, and analytics without the classic
indexing bugs: silent mis-scaling of unknown denoms, lost precision, and
pagination mistakes that silently drop data.

## Prerequisites
- Node >= 22 (BigInt) or Python 3.11+; HTTP access to the endpoints in
  `terra-classic-node`.

## Mistakes That Break Your Build

1. **Unknown IBC denoms must fail loudly, not default to 10^6.** When a denom has
   no registry entry (no LCD metadata, not in your local map), the worst default
   is the chain's base decimals — a 12-decimal IBC asset gets scaled by 10^6 and
   every exported row is wrong by a million. The correct behavior: log at
   *error* level with the caveat explicit, and skip or flag the row. Real case:
   [hodgerpodger/staketaxcsv#397](https://github.com/hodgerpodger/staketaxcsv/pull/397).

2. **"Exact integer division" that isn't.** In Python, `int(x) / 10**6` is a
   float division — the int is promoted first and the precision loss is
   identical to the original float bug. Verify your "fix" numerically before
   shipping it: the same audit caught its own proposed precision fix as a
   no-op (2^53 test).

3. **Pagination: default page sizes are small.** LCD list endpoints default to
   100 items. Following `pagination.next_key` is mandatory for complete data;
   "it looked complete" is how transfers vanish from reconciliations.

4. **Aggregates must reconcile to zero.** For any ledger view: sum(credits) −
   sum(debits) = known delta. Run this invariant on every batch; a broken
   invariant means your indexer misread a memo, a tax, or a malformed event.

5. **Tax rows are separate line items.** The burn tax deducts from the *sender*
   side; naive accountants double-count it into the transfer amount. Model it
   as its own row referencing the transfer.

## Verify
```bash
# Denom metadata lookup works and 6-decimals entries dominate:
curl -s "https://terra-classic-lcd.publicnode.com/cosmos/bank/v1beta1/denoms_metadata" | jq '.metadatas | length'
```
Your unknown-denom path should log ERROR and skip — grep your logs for the
caveat string in a fixture run to prove the loud path fires.

## Implementation
The loud unknown-denom handler (Node):

```js
export function scaleDenom(amountRaw, decimals) {
  if (!Number.isInteger(decimals)) throw new Error(`unknown decimals for denom`);
  const a = BigInt(amountRaw);
  const q = a / 10n ** BigInt(decimals);
  const r = a % 10n ** BigInt(decimals);
  return `${q}.${r.toString().padStart(decimals, "0")}`;
}

export function safeRow(denom, amountRaw, knownDecimals) {
  if (knownDecimals === undefined) {
    console.error(`UNKNOWN DENOM ${denom}: decimals unknown; row NOT scaled by 10^6 — fix registry`);
    return { denom, amountRaw, scaled: false };   // fail loud, not wrong
  }
  return { denom, amount: scaleDenom(amountRaw, knownDecimals), scaled: true };
}
```
