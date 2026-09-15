# Cosmos-Family Audit Master Ledger

Single consolidated record of every audit, PR, and verification run across the Terra
Classic / Cosmos / ICP workspaces. **This doc supersedes scattered status chatter** —
the per-repo `*AUDIT-FINDINGS.md` docs remain the detailed evidence; this is the
"where we are, what shipped, what's next" map, written to be blog-ready if the PRs
merge.

Last updated: 2026-09-13.

---

## 1. Shipped PRs (the blog-worthy ledger)

| # | PR | Repo | What | Why it matters | Verification standard applied |
|---|-----|------|------|----------------|-------------------------------|
| 1 | [#667](https://github.com/classic-terra/core/pull/667) | classic-terra/core | IsReverseCharge panic guard | message-server panic class | CI-parity golang:1.24 container, fail-on-vulnerable regression test |
| 2 | [#668](https://github.com/classic-terra/core/pull/668) | classic-terra/core | Oracle EndBlocker silent skip | votes dropped without trace → stale oracle | same |
| 3 | [#670](https://github.com/classic-terra/core/pull/670) | classic-terra/core | Tax-split div-by-zero | **one governance vote = chain halt** class | same |
| 4 | [#671](https://github.com/classic-terra/core/pull/671) | classic-terra/core | dyncomm genesis round-trip | validator-set corruption on export/import | same |
| 5 | [#672](https://github.com/classic-terra/core/pull/672) | classic-terra/core | Staking hook debug prints | log spam / info leak on hot path | same |
| 6 | [#50](https://github.com/terra-classic-io/website/pull/50) | terra-classic-io/website | W1 hardcoded fallback API key + W2 dead SSR state hydration | leaked credential + hydration wipe | repo CI, W1/W2 detail in `terra-website-AUDIT-FINDINGS.md` |
| 7 | [#2](https://github.com/StrathCole/oracle-go/pull/2) | StrathCole/oracle-go | Zero chain-rate guard in dry-run verification (`deviationPct` returns nil instead of panicking) + regression test | zero on-chain rate = feeder crash loop | build+vet+`go test -race`+gofumpt+pinned golangci-lint v2.5.0 (0 issues); panic proven by disabling guard in a container-internal copy |
| 8 | [#2](https://github.com/GoblinHunt/cosmes/pull/2) | GoblinHunt/cosmes | BigInt fee math in `calculateFee` | float math could emit `7e+24`-style fee amounts chains reject; unfloored/ flooring inconsistency | typecheck, eslint, vitest 20/20 incl. >2⁵³ precision case; old code proven failing that case |
| 9 | [#397](https://github.com/hodgerpodger/staketaxcsv#397) | hodgerpodger/staketaxcsv | Loud unknown-IBC-denom fallback (error-level + explicit 10⁶ caveat) | silent mis-scaled CSV rows for all users without an LCD node | python:3.12 container, 3/3 unit tests, pycodestyle; **honestly reverted** an `int()` "precision" half of the fix after my own test proved it numerically identical to the old float path |
| 10 | [#57](https://github.com/dfinity/public-multidex/pull/57) | dfinity/public-multidex | `finalisePendingMatch`: fail closed **with full refunds** on fee>debit | fixes both the audited trap (issue #56) **and** the credit-0 value-destruction hole in the first (LLM-authored) guard | `mops test` 18/18; repo lint-ratchet gate (incl. M0155 trap ratchet + candid freshness) PASS; **e2e on local replica, 12 suites covering the touched surface, all PASS** |

Open upstream PRs awaiting review: core #664-review (6-point review posted on
maintainer's MM2 PR), #668, #670, #671, #672; website #50; oracle-go #2; cosmes #2;
staketaxcsv #397; multidex #57. Filed issue: [multidex #56](https://github.com/dfinity/public-multidex/issues/56).

---

## 2. Coverage verdict per repo

| Repo | Verdict | Detail doc |
|------|---------|-----------|
| classic-terra/core | ✅ full audit, 5 PRs shipped | `core/AUDIT-FINDINGS.md` |
| terra-classic-io/website | ✅ full audit, PR #50 | `terra-website-AUDIT-FINDINGS.md` |
| StrathCole/oracle-go | ✅ audited + PR #2 | `oracle-go-AUDIT-FINDINGS.md` |
| lunctoken/terra-classic-mm2 | ✅ audited (delta-only); F1 open | `terra-classic-mm2-AUDIT-FINDINGS.md` — the `mm2-development` branch **still carries the unguarded #670-class tax-split division**; must cherry-pick #670 before any merge/release |
| hodgerpodger/staketaxcsv | ✅ audited + PR #397 | `TIER2-AUDIT-FINDINGS.md` |
| GoblinHunt/cosmes | ✅ audited + PR #2 | `TIER2-AUDIT-FINDINGS.md` |
| astroport-classic-api / frontend / recovery | ⏭️ skipped/archived | Astroport left the chain (user directive); `TIER2B-ASTROPORT-CLASSIC-FINDINGS.md` |
| leapwallet cosmos-metamask-snap | ⚰️ moot | S1 archived — Leap sunset 2026-05-28, repo deleted |
| 9 registry/metadata mirrors | ✅ all sound | `TIER3-REGISTRY-CHECKS.md`, `TIER3B-REGISTRY-CHECKS.md` |
| wallets (Galaxy Station, Keplr) | 🔍 recon only | Galaxy Station public repos frozen 2023-09-30 (live source unlocated); Keplr wallet went closed-source (`chainapsis/keplr-wallet` gone) — but `keplr-chain-registry` is alive and its LUNC entry verified end-to-end (endpoints, coinType, gasPriceStep matches on-chain tax params exactly) |
| **dfinity/public-multidex (ICP)** | ✅ full audit + cross-model verification + PR #57 | `multidex/AUDIT-NOTES.md` + §3 below |

Scope note: multidex is Internet Computer, not Cosmos — it entered the audit family
via the user's original repo list and is included here for completeness.

---

## 3. Multidex: independent verification of the three Nemotron/Kimi reports

Three sibling LLM reports were audited (`multidex/nemotron30.md` = KimiK3-30B,
`nemotron120.md` = Nemotron-120B, `nemotron550.md` = Nemotron-550B), claim by claim,
against the actual Motoko source. Verdicts:

| Claim (severity as filed) | Models claiming it | Independent verdict | Evidence |
|---------------------------|--------------------|---------------------|----------|
| Unguarded Nat fee subtraction in `finalisePendingMatch` traps on underflow; heartbeat wedge if reached (Medium, latent) | all 3 | **REAL — confirmed** | only unguarded fee subtraction in the codebase; reachable only via a path the sealed model never exercises today; filed as issue #56 before any model saw it |
| GEPTOR oracle path accepts single-source mark updates vs periodic `minSources()` floor (Medium) | 30B; 550B filed then **self-retracted** | **HALLUCINATED (stale)** — the M3 refactor routes the GEPTOR hot path through `applyFreshAggregate`, which enforces the same `minSources()` floor; 30B audited pre-M3 knowledge | comment at the call site says it verbatim; 550B caught its own error mid-report — the honest behavior pattern |
| Reserved collateral counted as backing but unseizable → margin/liquidation escape, debt written off (High) | 30B, 550B | **HALLUCINATED CLASS (overstated)** — describes the pre-Model-2 world; the repo migrated past it ("Model 2 made the ORIGINAL human-principal escapes structurally impossible — humans can't borrow", per the test suite's own header), the residual surface is gated, and the "fix" they recommend (re-adding `gateInitialMargin` gates) was deleted as dead code in W5-23 with an explicit *"Do not re-file that ask"* comment in-tree | `tests/test_margin_collateral_escape.sh` header; W5-23 removal comment |
| `extMarketSwap` guarded only by `IS_PRODUCTION` flag (Medium) | 30B, 550B | **REAL but overstated** — compile-time constant plus a second `isArbitrageur` gate (controller-set) plus per-call/hourly USD caps; residual risk is "if the deploy is misconfigured", which is not a vulnerability | main.mo production guard + arb wiring |
| Price band 100× rejects large-but-valid orders (Low) | 30B, 550B | **COSMETIC** — intentional fat-finger guard, error text self-documents | design choice, acknowledged in-band |
| Two-stage guard (100× band + 5% marketable collar) confuses users (Low) | 30B, 550B | **COSMETIC/docs nit** | UX wording only |

**Net: of 6 distinct claims across 3 models, 1 real (already filed by us as #56),
1 real-but-overstated, 2 cosmetic, 2 hallucinated — including a High-severity claim
both large models agreed on that the codebase's own comments and tests pre-rebut.**
Corroboration is not confirmation: two models repeating each other's error made it
look stronger, not more true.

**The layer the models all missed — and the finding that became PR #57:** the
*first* guard applied to fix #56 (the 550B fix shape, credited in `nemotron120.md`/
`nemotron550.md`) itself had a conservation hole — its fail-closed branch credited
`0` to the maker **after both reservations were already consumed**, silently
destroying the reserved value with no event logged. PR #57 fixes both layers:
fee computations hoisted before any credit, and `fee > debit` now refunds both legs
in full and drops the match fail-closed with a loud warn — the exact convention the
codebase's own reservation-desync path uses one body above. Normal path is
byte-identical (`quoteFeeFor` is pure).

---

## 4. Verification methodology (the reusable part)

1. **CI-parity before any push** (adopted as standing policy 2026-09-13): build, vet,
   tests with the repo's exact flags, formatter, pinned linter — run in the repo's
   canonical Linux environment (containers locally; WSL2 for ICP/Motoko toolchains).
2. **Fail-on-vulnerable proof**: every regression test must be shown to fail on the
   pre-fix code (guard disabled in a disposable copy/worktree — never sed-on-host),
   proving the test actually covers the bug.
3. **Self-audit the fix**: the staketaxcsv `int()` incident and the multidex credit-0
   incident both came from verifying our own patch as adversarially as the original
   bug. The fix is guilty until its tests prove otherwise.
4. **Flatten guard chains where flattening is honest** (plain conditionals → single
   comparison or tagless switch); match the repo's existing style; never contort
   logic to remove an `if`.
5. **Minimal diffs, no over-commenting** per `LUNC-COMMIT-GUIDELINES.md`; reasoning
   goes in the PR body, not the diff.
6. **E2E where the surface warrants it**: multidex got a full local-replica run
   (`#dev` posture + pending-match fixture) with 12 targeted suites plus the repo's
   lint/trap/candid ratchet gate; baseline-vs-fix comparison to separate
   environmental failures from real ones (caught 4 false failures this way).

---

## 5. Where we left off / next actions

- **Awaiting review upstream**: all 10 PRs above. No further action until maintainer
  response; re-run CI-parity gates if repos move.
- **multidex `mm2-development`** (LUNC side): still carries the unguarded tax-split —
  cherry-pick #670 onto it before it can ever ship (standing hazard).
- **Keplr chain registry**: the one live Keplr-adjacent PR surface; LUNC entry
  verified clean on 2026-09-13. When EUTC ships, register the token there.
- **Website W3/W4** (from the full website audit): not yet filed as PRs.
- **Forex/EUTC workstream (Proposal 12209, passed-but-never-implemented)**: reference
  implementation complete in `forex/` (14/14 integration tests); next steps listed in
  `LUNC-AUDIT-INDEX.md` §Queued — wasm32 release builds, property tests I1–I8, live
  wasmd testnet, then propose upstream (greenfield; nothing to PR into).
- **Blog article**: §1 is the ledger; the arc is "5 core chain-halt-class fixes →
  tooling PRs across 4 ecosystems → independent verification of 3 LLM audit reports
  (2 of 6 claims hallucinated, including the scariest one) → the fix-of-the-fix".
  Per `forex/docs/TERMINOLOGY.md`: keep repeg/stablecoin/peg wording out of public
  materials.

## 6. LUNC × ICP initiative (built 2026-09-13, approved plan)

First skill set + MCP + agent-safety layer for **any** Cosmos SDK chain (verified:
nothing equivalent exists in LUNC or the wider Cosmos ecosystem; closest analog is
celestiaorg/celestia-engineering, dev-workflow-only). Three repos:

- **`lunc-skills/`** — 9 agent skills in dfinity/icskills format (build + action +
  integration), every pitfall sourced from a shipped PR (§1), sha256-pinned index
  (`INDEX-HASHES.json`), `## Verify` self-freshness sections, served via certified
  ICP canister (`icp.yaml`, `@dfinity/static-site@v0.3.3`) with Pages mirror;
  `awesome-lunc-ai/` community index. Facts verified against live endpoints
  2026-09-13 (tax params route = `/terra/tax/v1beta1/params`, schema =
  `gas_prices` + `burn_tax_rate` 0.015, gov 40/50/33.4, min deposit 5M LUNC).
- **`lunc-mcp/`** — Terra Classic MCP in two deployments from one schema: local
  stdio server (zero-dep Node, self-test 7/7 green, protocol e2e verified live) +
  ICP canister (planned; cost model: free certified cache / cycle-paid live reads
  ~$0.0002 per LCD outcall / later CKLUNC-402). Includes the **finding registry**
  (`registry/findings.json`, 11 entries: 8 verified / 2 refuted / 1 disputed — the
  Nemotron verdicts of §3, machine-readable and falsifiable).
- **`cklunc/`** — chain-key LUNC twin, core complete and tested in WSL (mops
  2.19.2 + core 2.0.0): exact-integer tax parity (Dec parsing, floored tax,
  `net+tax=amount`), fail-closed params freshness (24h, epoch 0 = refuse),
  settlement burn ledger with per-epoch LUNC txid linking. **Rate policy shipped:
  parity by default; governed override can only LOWER below the mirrored rate
  (0% allowed for routing legs); every response carries `rate_mode` provenance
  (`parity | override | refused`); freshness bound governance-adjustable 1h–7d.**
  Roadmap B1→B2b in `cklunc/README.md`; **testnet-only until threshold-ECDSA
  custody (B2b) + security review**. Note: the unit suite caught its own sanity
  bound rejecting the live 1.5% rate during development — bound corrected to ≤5%.

Original contributions vs Solana/dfinity ecosystems: chain-key skill hosting,
self-verifying skills, attested finding registry, agent-scoped ICRC-2 allowances,
and a burns-native agent economy (every CKLUNC transfer burns on both chains).

## 7. Self-audit pass over the LUNC×ICP repos (2026-09-13, second review)

A fresh-eyes review of all three repos found and fixed **17 real issues** that the
original build pass had shipped. Every chain-dependent claim was re-verified live
before fixing:

**lunc-mcp (server correctness):**
1. `simulate_tx` mixed float tax math (`BigInt(Math.round(rate*1e18))`) — replaced
   with the exact Dec parser (`util.mjs`), tested on >2^53 amounts.
2. `simulate_tx` returned a raw BigInt → `JSON.stringify` **threw** on every taxed
   MsgSend — found because the new tests crash on it; stringified at the edge.
3. `proposals` used gov/v1beta1 with `limit=500` — **the v1beta1 list endpoint
   cannot encode modern multi-message proposals** (conversion error), and forward
   pagination returned proposals #1–#2 of 1,824, so no current proposal was ever
   visible. Migrated to gov/v1 + tail-slice + newest-first (live-verified).
4. `proposals("voting")` never matched (v1 sends `VOTING_PERIOD`, not `..._VOTING_PERIOD`);
   both enums now accepted.
5. `explain_tx` upper-cased tx hashes (hashes are case-sensitive identifiers);
   also failed txs now surface raw_log and a "value did not move" note.
6. `validator_set` compared token strings with `>` (lexicographic mis-sort);
   BigInt comparator + explicit rank, pinned by a test with two float-equal
   validators (1e18 vs 1e18−1).
7. `lcdPage` derived the response array from the URL's last segment — the
   *address* on parameterized routes; the self-test's empty-balance fixture hid
   it. Explicit field names now.
8. No pagination-following anywhere (`next_key` ignored; publicnode ignores
   `pagination.limit`) — implemented with a hard page cap.
9. No fetch timeouts (one hung LCD stalls the server) — 8 s `AbortSignal.timeout`.
10. Tool failures reported as JSON-RPC protocol errors; per MCP spec they are
    `isError:true` results now (protocol e2e pinned).

**lunc-skills (wrong facts / dangerous examples):**
11. `burnTax` snippet parsed "0.015" as `BigInt("015")` = **15** (×1000 overtax);
    rewritten as true 10^-18 fixed-point parsing.
12. gov-skill pass condition was mathematically wrong (veto term inflated ×334);
    quorum/threshold denominators corrected to SDK semantics.
13. cklunc-integration skill said the sanity bound is ≤0.5% — the live rate is
    1.5%; corrected to 5% (the bound in the code was already right).
14. `gas: gas * 1.2 | 0` overflowed int32 and truncated; BigInt ceil now.
15. jq example `.params.burn_tax_rate, .params.gas_prices | length?` prints the
    **rate's string length** (19), not the gas-table size — precedence fixed,
    documented as a pitfall (live-verified).
16. The market/swap Verify example hits a dead endpoint: `usdr`/`uusd` return
    HTTP 200 with **amount 0** (oracle price missing), `ustc` errors — the
    skill now documents the trap and points at oracle exchange_rates (live-verified).
    Same for the skill's overpromised tool list vs the 11 implemented tools.
17. llms.txt advertised a nonexistent GitHub repo; the `feeeder` typo and a
    wrong slash-position example fixed.

**cklunc (economic/security):**
18. **Caller-supplied clock bypass**: `setTaxParams(rateDec, atNs)` accepted the
    caller's timestamp, so a hostile caller could pass a fresh `nowNs` against
    params fetched weeks earlier and tax transfers at a stale (or zero) rate.
    All timestamps are now consensus `Time.now()`; the parameter is gone.
19. **Unauthenticated admin calls**: `setTaxParams`/`markSettled` were public —
    anyone could set the mirrored rate or forge settlement memos. Both are
    controller-gated now (`Principal.isController`, the multidex codebase's own
    idiom), correct across the B2b decentralization because controllers become
    the governing canisters.
20. Ledger rebuilds were O(n) per burn (append-and-copy via tabulate) — replaced
    with a geometric-growth mutable backing array (O(1) amortized burn, single
    in-place O(n) settlement pass), with an immutable snapshot at the query boundary.

Verification after the pass: `npm test` **17/17** (new offline suite: pure-math
unit tests + full MCP protocol e2e over a stubbed LCD, 4 tests fail on the old
code), `--self-test` **7/7 live**, skills index **9/9 hash-verified**, `mops test`
PASS + `mops build` ✓ in WSL. Lesson repeated from §4.3: the most dangerous bugs
were in code that already "passed" — the v1beta1 gov limitation, the BigInt crash,
and the clock bypass were all invisible to the original self-test because its
fixtures agreed with the bug (empty balances, ascending pages, well-behaved caller).
