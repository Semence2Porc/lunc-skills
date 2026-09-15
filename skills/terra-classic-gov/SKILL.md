# Terra Classic Governance

## What This Is
Reading and participating in Terra Classic governance from agent workflows:
proposals, tallies, quorum math with the chain's verified parameters, and the
pitfalls around params routes and proposal-state assumptions.

## Prerequisites
- Read-only LCD access (`terra-classic-node`). Voting is a signing operation —
  see `terra-classic-wallet-ops` for the approval rails.

## Verified parameters (2026-09-13)
| Param | Value | Route |
|---|---|---|
| quorum | `0.40` | `/cosmos/gov/v1beta1/params/tallying` |
| threshold | `0.50` | same |
| veto threshold | `0.334` | same |
| min deposit | `5000000000000 uluna` (5M LUNC) | `/cosmos/gov/v1beta1/params/deposit` |
| max deposit period | `1209600s` (14 d) | same |

## Mistakes That Break Your Build

1. **The combined params route lies (returns zeros) on some nodes.** Query the
   split routes (`/params/tallying`, `/params/deposit`) — the combined
   `/cosmos/gov/v1beta1/params` returned zero-valued tallies while split routes
   returned real values (verified on two LCDs, 2026-09-13).

2. **The gov/v1beta1 LIST endpoint cannot encode modern proposals.** Any proposal
   with more than one message fails conversion — the list route errors with
   `can't convert a gov/v1 Proposal to gov/v1beta1 Proposal` the moment one such
   proposal exists, and even `pagination.reverse=true` trips it (verified
   2026-09-13, chain at proposal 12226). Single-proposal GETs still work on
   v1beta1; for LISTS use `/cosmos/gov/v1/proposals` (fields: `id`, `title`,
   `status` — not `proposal_id`/`content.title`). On gov/v1, `pagination.reverse=true`
   works and gives newest-first (verified); forward pagination is the portable
   fallback — take the tail for "newest".

3. **Quorum math: quorum is over (Yes+No+NoWithVeto+Abstain) vs total bonded;
   threshold and veto use (Yes+No+NoWithVeto).** A proposal passes when:
   quorum ≥ 0.40 AND yes/(yes+no+veto) > 0.50 AND veto/(yes+no+veto) < 0.334.
   Getting the denominator wrong (using total votes including abstain for the
   threshold, or excluding veto) flips results on close votes. Do the exact
   integer math on raw power strings.

4. **"Passed" ≠ "implemented".** Proposals can pass and never ship (the repeg
   family passed in 2022 with no code). Check for a matching `app/upgrades/`
   height or a core PR before telling a user a passed proposal is live.

5. **Deposit denominations and periods are params, not constants.** Re-read them
   per session; a new proposal type or param change silently shifts them.

## Verify
```bash
curl -s https://terra-classic-lcd.publicnode.com/cosmos/gov/v1beta1/params/tallying | jq '.tally_params'
# Lists MUST use gov/v1 (see Mistake 2); newest first:
curl -s "https://terra-classic-lcd.publicnode.com/cosmos/gov/v1/proposals?pagination.limit=5&pagination.reverse=true" | jq '.proposals[] | {id, status, title}'
# The v1beta1 list endpoint errors once a multi-message proposal exists — proof:
curl -s "https://terra-classic-lcd.publicnode.com/cosmos/gov/v1beta1/proposals?pagination.reverse=true" | jq -r '.message // "(v1beta1 list still works)"'
```
If quorum/veto differ from the table, re-derive everything in this skill.

## Implementation
Pass/fail evaluation, exact and denominator-correct:

```js
export function evaluateTally(t, quorum = 4n * 10n**17n, threshold = 5n * 10n**17n, vetoT = 334n * 10n**15n) {
  const yes = BigInt(t.yes), no = BigInt(t.no),
        veto = BigInt(t.no_with_veto), abstain = BigInt(t.abstain);
  const voted = yes + no + veto + abstain;
  const bonded = BigInt(t.bonded_tokens ?? "0");       // from /cosmos/staking
  if (voted * 10n**18n < quorum * bonded) return "QUORUM_NOT_MET";
  const nonAbstain = yes + no + veto;                   // SDK: abstain counts for quorum only
  if (yes * 10n**18n <= threshold * nonAbstain) return "REJECTED";
  if (veto * 10n**18n >= vetoT * nonAbstain) return "VETOED";
  return "PASSED";
}
```
(For production, pull the three params from the Verify route instead of the
defaults, and unit-test against a real closed proposal's tally.)
