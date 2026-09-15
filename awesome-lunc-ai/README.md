# awesome-lunc-ai

A curated index of AI-agent tooling for Terra Classic (LUNC). PR to add yours —
one row, one link, keep the description honest.

The rules mirror [awesome-solana-ai](https://github.com/solana-foundation/awesome-solana-ai):
open index, community PRs, no pay-to-list. Items marked *verified* link to a
reproducing test or shipped PR.

## Skills & Knowledge

| Project | What | Verified |
|---|---|---|
| [lunc-skills](../) | Agent skills for building & operating on Terra Classic (build + action + integration) | PRs below |
| [dfinity icskills](https://github.com/dfinity/icskills) | The ICP skill set this format mirrors | — |

## Live Tools (MCP)

| Project | What | Verified |
|---|---|---|
| [lunc-mcp](https://github.com/Semence2Porc/lunc-mcp) | Terra Classic MCP — local stdio (free) + ICP canister (certified); balances, explain_tx, simulate_tx, burn series, finding registry | this repo |
| [oracle-go zero-rate guard](https://github.com/StrathCole/oracle-go/pull/2) | Voter refuses to vote a zero chain rate instead of dividing by it | PR open |
| [staketaxcsv loud IBC denoms](https://github.com/BryanStaketaxCsv/staketaxcsv/pulls) | Unknown IBC denoms log loudly instead of silently defaulting to 10^6 scaling | PR open |
| [cosmes BigInt fees](https://github.com/InterWiseOS/cosmes/pulls) | Fee math moved to BigInt — no `7e+24`-style exponent regression in tx bodies | PR open |
| [multidex fee guard](https://github.com/StrathCole/public-multidex/pulls) | Fee subtraction guarded so failed paths cannot double-spend ledger accounting | PR open |

## Audits & Findings

| Project | What | Verified |
|---|---|---|
| [COSMOS-AUDIT-MASTER.md](../COSMOS-AUDIT-MASTER.md) | Cross-family audit ledger: 10 PRs, LLM-report verification verdicts | all links in doc |
| [multidex finding registry](https://github.com/Semence2Porc/lunc-mcp/tree/main/registry) | Attested, falsifiable audit findings (machine-readable) | repro commands |

## Protocols

| Project | What | Verified |
|---|---|---|
| [x402](https://www.x402.org) | HTTP-402 machine payments (Solana-origin; the CKLUNC-settled analog is planned in cklunc B5) | — |

## Rules

1. One PR per project; keep the table sorted alphabetically.
2. Description ≤ 100 chars, no marketing superlatives.
3. "Verified" means a reviewer can reproduce the claim from the linked artifact.
4. Dead items get removed after 30 days of 404s (automated check welcome).
