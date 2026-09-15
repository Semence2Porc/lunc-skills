# Cosmos SDK Gotchas (Terra Classic Edition)

## What This Is
Chain-halt-class bug patterns for Go contributors on Terra Classic / Cosmos SDK
chains. Every pattern here maps to a real, shipped PR with a reproducing test —
this is the corpus of what actually breaks a live Cosmos chain, not a style guide.

## Prerequisites
- Go 1.24+ for the referenced repos; familiarity with Cosmos SDK module structure.

## Mistakes That Break Your Build

1. **Division by zero in governance-triggered paths = chain halt.** Any `%` or `/`
   on a value a proposal can set to zero (tax split denominator, reward weights,
   commission rates) must be guarded *before* the division, not after. Real case:
   one governance vote away from a halt —
   [classic-terra/core#670](https://github.com/classic-terra/core/pull/670).

2. **Message-server panics on malformed input.** `IsReverseCharge`-style boolean
   checks that dereference or panic on edge-case inputs take down the node that
   executes the tx. Return errors, never panic, in any `msg_server` path —
   [#667](https://github.com/classic-terra/core/pull/667).

3. **Genesis round-trips must be lossless.** Export → import must produce
   identical validator state (dynamic-commission drift corrupts the validator set
   on upgrade) — [#671](https://github.com/classic-terra/core/pull/671). Test:
   export, import, export again, deep-compare.

4. **Silent skips in EndBlockers hide consensus-visible state.** An oracle
   EndBlocker that drops votes without event/log produces stale prices downstream
   and un-debuggable feeder behavior —
   [#668](https://github.com/classic-terra/core/pull/668). Every skip path needs a
   loud, greppable trace.

5. **Zero-value chain rates crash feeder verification.** Any deviation math like
   `(voted - chain) / chain` divides by zero when the chain rate is zero —
   guard it and return nil/neutral, with a test proving the panic existed —
   [StrathCole/oracle-go#2](https://github.com/StrathCole/oracle-go/pull/2) (the
   test fails with `panic: division by zero` on the pre-fix code).

6. **Fail-closed must still conserve value.** When you add a guard to a money
   path, audit what the guard does with *already-debited* funds: crediting `0`
   after debit silently destroys user value (this bug was found inside an LLM's
   proposed fix during the multidex audit — dfinity/public-multidex#57). Refund
   and drop, log loudly, bump user versions.

7. **Corroboration is not confirmation.** Two audit reports repeating the same
   claim make it look stronger, not more true. Verify every claim against the
   code before filing or fixing; document verdicts with evidence.

## Verify
```bash
# All referenced PRs are public and reviewable:
for pr in 670 667 671 668; do
  gh pr view $pr -R classic-terra/core --json state,title --jq '"\(.state): \(.title)"'
done
gh pr view 2 -R StrathCole/oracle-go --json state,title --jq '"\(.state): \(.title)"'
```

## Implementation
The guard shape that passes review on a chain-halt class bug (flattened, minimal):

```go
if !chain.IsPositive() {
    logger.Warn("zero chain rate; skipping deviation check", "denom", denom)
    continue
}
dev := deviationPct(voted, chain) // never called with chain == 0
```

Regression tests must fail on the vulnerable code (delete the guard → test panics
→ restore), proving the test covers the bug — see each referenced PR for the exact
pattern.
