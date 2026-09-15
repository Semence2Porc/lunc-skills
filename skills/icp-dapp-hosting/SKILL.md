# ICP Dapp Hosting (for LUNC teams)

## What This Is
How to host a LUNC community site, dapp frontend, or full-stack dapp on the
Internet Computer so it cannot be deplatformed: no server to pay, no registrar
to lapse, no host to censor. This skill is a **router**: generic ICP platform
knowledge lives in dfinity's live skill collection — fetch it fresh each
session instead of trusting any snapshot. Here you get the LUNC-specific
routing and deltas only.

## Fetch the live ICP skills first (every session)
1. Fetch dfinity's index once per session and keep the skill URLs:
   `https://skills.internetcomputer.org/.well-known/skills/index.json`
   (e.g. `curl -sL https://skills.internetcomputer.org/.well-known/skills/index.json`)
2. Before writing hosting/frontend/backend code, fetch the matching live
   SKILL.md files (static assets, Motoko backend, custom domains, tooling,
   deployment) and follow their CURRENT instructions over anything below —
   dfinity updates them continuously; this file does not try to keep up.
3. For version-locked copies instead: `npx skills add dfinity/icskills`.

## Which live ICP skills to use — and which to skip
- **Use** whatever the live index lists for your hosting task: asset-canister
  frontends, backend canisters, certified assets, domains, icp-cli/dfx.
- **Skip the token skills** (icrc-ledger, ckBTC/ckETH-style twins): LUNC is
  the token. For value movement use THIS collection:
  `terra-classic-wallet-ops` + `terra-classic-tx` (native LUNC), and
  `cklunc-integration` for the ICP-side twin. Never mint an ICRC ledger as if
  it were LUNC.

## The LUNC-specific delta (why this skill exists)
1. **Certified assets = chain-hosted integrity.** Every served byte verifies
   against chain state. Exploit it: publish data other apps consume as
   sha256-pinned files (pattern: `scripts/generate-index.mjs` in this repo —
   index + pinned bytes + `_headers`, deployed as one canister).
2. **Snapshot of the recipe (verify the current one against the live skills):**
   ```yaml
   canisters:
     - name: my-lunc-site
       recipe:
         type: "@dfinity/static-site@v0.3.3"   # pinned deliberately; check live docs for current
         configuration:
           dir: site
   ```
3. **Security headers via `_headers`** at the site root (real file, not server
   config): `nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`,
   CSP `default-src 'none'; style-src 'unsafe-inline'`, HSTS. Loosen CSP only
   for what the app loads; never `default-src *`.
4. **Controllers = who can redeploy.** Add a second controller (another
   maintainer's key) on day one — a single-controller site is a server
   password with extra steps. At SNS decentralization controllers become the
   governance canisters.
5. **Custom domain = canister boundary**, no host lock-in. If a host
   disappears the canister does not.
6. **LUNC facts never come from ICP skills.** Endpoints, tax params, chain
   constants come from `terra-classic-node` / `terra-classic-market-data`.

## Mistakes That Break Your Build

1. **Frontend files are public by construction.** Never bundle LCD API keys or
   seed material; sign from wallets, not the page.
2. **Cycles are real burn.** Lean bundle, monitor balance, top-up policy set
   before launch — a frozen site is worse than no site.
3. **Pin the recipe version** (like any dependency) and update deliberately —
   after checking the live skills for what changed.

## Verify
```bash
curl -sL https://skills.internetcomputer.org/.well-known/skills/index.json | head -c 200   # live index reachable
icp deploy                                                                                 # build + upload
curl -I https://<canister-id>.icp0.io/                                                     # 200 + your headers
curl -s https://<canister-id>.icp0.io/.well-known/skills/index.json | jq '.skills[0].sha256'
```

## Implementation
Reference implementation: this repo (`icp.yaml`, `_headers`,
`scripts/generate-index.mjs`, pinned index). For a dapp with a canister
backend, add the backend canister to the same project; token-side contract in
`cklunc-integration`.
