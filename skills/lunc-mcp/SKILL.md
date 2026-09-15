# lunc-mcp (Terra Classic Live Tools for Agents)

## What This Is
Live Terra Classic tools exposed to AI agents over the Model Context Protocol,
in two deployments from one shared tool schema:

- **Local server** (Node, stdio): free forever, private, uses *your* LCD endpoint,
  plus local-key wallet ops under the action-skill rails.
- **ICP canister** ([Semence2Porc/lunc-mcp](https://github.com/Semence2Porc/lunc-mcp) `canister/`, in progress): public,
  response-certified, free cached reads + cycle-paid live reads.

Skills teach; MCP tools act. This skill documents both surfaces so an agent can
use either transparently.

## Prerequisites
- Node >= 22 for the local server ([`server/lunc-mcp.mjs`](https://github.com/Semence2Porc/lunc-mcp/blob/main/server/lunc-mcp.mjs)).
- An MCP client (Claude Desktop, Cursor, or any stdio MCP host) OR plain HTTP for
  the canister.

## Tool Surface
The shared schema ([`server/schema.mjs`](https://github.com/Semence2Porc/lunc-mcp/blob/main/server/schema.mjs)) is the source of truth — run
`tools/list` rather than trusting any document. Today's 11 tools:

- **Accounts:** `balance` (native), `wallet_overview` (balances + delegations +
  unbonding + rewards + authz grants in one call)
- **Transactions:** `explain_tx` (hash → decoded human text), `simulate_tx`
  (gas + integer burn-tax impact pre-sign)
- **Contracts:** `contract_query` — generic CW query passthrough (any Terra
  Classic smart contract)
- **Staking:** `validator_set` (integer-ranked voting power)
- **Governance:** `proposals` (newest first, status filter — gov/v1 backed, see
  `terra-classic-gov` Mistake 2)
- **Economy:** `tax_params` (live), `burn_snapshot` (rate + raw supply)
- **IBC:** `ibc_denom_trace`
- **Registry:** `findings` — attested, falsifiable audit findings
  ([`registry/findings.json`](https://github.com/Semence2Porc/lunc-mcp/blob/main/registry/findings.json)), served as a tool so verified knowledge
  becomes live context

Growth candidates (not built yet — do not assume): CW20 balances, votes,
tally evaluation, diff-since-height, token metadata, commissions, missed blocks,
channel/client-expiry warnings, supply history, community pool, swap rates,
oracle health.

## Mistakes That Break Your Build

1. **stdio MCP servers must speak pure JSON-RPC on stdout.** Anything else —
   logs, banners, progress — corrupts the stream. Log to stderr only.
2. **The canister's free tier is cached, not live.** Cached reads are
   TTL-bounded and certified; if your use case needs block-fresh values, use the
   live tier (caller attaches cycles) — see [`canister/README.md`](https://github.com/Semence2Porc/lunc-mcp/blob/main/canister/README.md).
3. **`explain_tx` output is informational, never authorization.** Decoded text
   helps a human approve; it is not a substitute for simulate + explicit
   confirmation.
4. **Balances and supplies are raw base-unit strings.** Treat every amount as
   BigInt in consumers (uluna supply exceeds 2^53); unknown denoms must not be
   scaled by any assumed exponent (see `terra-classic-indexing`).

## Verify
```bash
git clone https://github.com/Semence2Porc/lunc-mcp && cd lunc-mcp
node server/lunc-mcp.mjs --self-test   # runs the tool suite against your LCD
```

## Implementation
Registering the server with a stdio MCP client (Claude Desktop example):

```json
{
  "mcpServers": {
    "lunc": {
      "command": "node",
      "args": ["/absolute/path/to/lunc-mcp/server/lunc-mcp.mjs"],  # clone Semence2Porc/lunc-mcp first
      "env": { "LUNC_LCD": "https://terra-classic-lcd.publicnode.com" }
    }
  }
}
```
