# Terra Classic Wallet Ops (Agent-Safe)

## What This Is
Performing wallet operations on Terra Classic from AI-agent workflows without
becoming an attack vector. The rails here are structural, not advisory: dry-run
first, per-transaction human confirmation, and allowance-scoped spending instead
of key custody.

## Prerequisites
- Read-only access to an LCD (see `terra-classic-node`).
- For signing: the human user's explicit approval flow. **An agent never holds a
  mnemonic.** If a product design requires the agent to hold keys, redesign the
  product.

## Mistakes That Break Your Build

1. **Signing from a chat-instructed mnemonic is the attack.** The #1 vector:
   "just paste your seed phrase so I can check your balance." Agents must use
   view-only reads and hand a ready-to-sign payload to the human's wallet
   (Keplr/Station/WalletConnect), or operate inside a finite allowance (rule 4).

2. **Broadcast without simulate.** Always `/cosmos/tx/v1beta1/simulate` first:
   it returns gas estimate + fee requirements and fails safely without moving
   value. Fold the burn tax in (see `terra-classic-tx` Mistake 3).

3. **Address validation is bech32 + HRP.** A valid bech32 string for another
   Cosmos chain decodes fine and burns funds. Validate `terra1…` HRP AND
   32-byte data length before any transfer call.

4. **Agents spend from allowances, not balances.** The safe pattern (ICRC-2
   proven, works for any approve/transferFrom chain): the human grants a *finite*
   allowance to an agent-controlled burner account, capped per day/week, and
   revokes it in one tx. The blast radius of a compromised agent is the
   allowance, never the wallet. On ICP this is native (`icrc2_approve` with
   `expires_at`); on Terra Classic use authz send-authorization with expiry.

5. **Authz grants need expiration and caps.** `cosmos.authz.v1beta1` send
   authorizations without `expiration` or spend limits are open doors. Set both,
   always, and audit existing grants (`wallet_overview` in [`lunc-mcp`](https://github.com/Semence2Porc/lunc-mcp) surfaces
   them).

6. **Display precision:** show 6-decimal LUNC, never scientific notation, in
   anything a human reads (see `terra-classic-tx` for the float ban).

## Verify
```bash
# Read-only sanity: balance endpoint for any address you control
curl -s "https://terra-classic-lcd.publicnode.com/cosmos/bank/v1beta1/balances/<terra1...>"

# Your agent config contains NO mnemonic (grep your own config):
grep -r "seed\|mnemonic" ./agent-config/ && echo "FAIL: secrets in config"
```

## Implementation
The agent-safe transfer envelope:

```js
export async function prepareTransfer({ lcd, from, to, amountUluna, memo }) {
  assertTerraAddress(to);                        // HRP + length checks
  const sim = await simulate({ lcd, tx: buildTx({ from, to, amountUluna, memo }) });
  const { gas, fee, burnTax } = sim;             // tax folded per current params
  return {
    kind: "SIGNING_PROPOSAL",                    // never a signed tx
    tx: buildTx({ from, to, amountUluna, memo, gas: (gas * 12n) / 10n, fee }), // ceil 1.2×, BigInt — `gas * 1.2 | 0` overflows int32 and truncates
    summary: `Send ${amountUluna} uluna (${BigInt(amountUluna) / 10n ** 6n} LUNC) + ${burnTax} tax to ${to}`,
    requires: "human_approval",                  // the wallet signs, not the agent
  };
}
```
