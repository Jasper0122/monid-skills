---
name: honeypot-killer
version: 0.2.0
description: Crypto Honeypot Killer — a layered scam screener you run BEFORE you ape into a token. Confirms the official contract (anti-impersonation), runs the honeypot/security check across TWO independent sources, pulls the deployer's rap sheet (has this wallet rugged before?), checks holder concentration, contract verification, upgradeability, and unlocks, then returns a blunt KILL / CAUTION / CLEAR verdict. Read-only research — never buys, approves, or signs. Use when asked to "screen this token", "honeypot check", "貔貅", "rug check", "打新", "防骗", "is this token safe", or given a 0x token address.
---

# Crypto Honeypot Killer

A layered pre-buy scam screener. Where a basic checker runs one honeypot test, this runs a
**defense-in-depth** stack: verify the real contract → dual-source honeypot test → deployer rap
sheet → distribution / upgradeability / unlocks → blunt **🚫 KILL / ⚠️ CAUTION / ✅ CLEAR** verdict.
Read-only — it never buys, approves, or signs anything.

**It is a strong filter, not a bulletproof vest.** It catches the honeypots and on-chain traps a
human eyeballing Etherscan misses, but a CLEAR verdict is NOT "safe to buy" — team intent, novel
zero-day mechanisms, and post-scan upgrades cannot be fully prevented, only managed (small size +
monitoring).

## Prerequisites

Pay-as-you-go **monid** CLI (no free tier):
1. https://app.monid.ai → add funds → API key from /access/api-keys
2. `monid keys add -k <key> -l main` → `monid keys list`
- Preflight: `monid --version || npm install -g @monid-ai/cli`
- `NO_COLOR=1`. CLI only, never MCP. `monid run -p <provider> -e <endpoint> --query '<json>' -i '<json>' --wait <sec> -j -o <file>`. `-o` strips to `.output`. On 402 → fund wallet.

### Chain naming (providers differ — get this right)
| chain | Strale / Heurist `chain_id` | BlockRun `chain` |
|---|---|---|
| Ethereum | `1` | `ethereum` |
| BSC | `56` | `bsc` |
| Base | `8453` | `base` |
| Solana | `solana` | `solana` |
Default Ethereum (`1` / `ethereum`) if unspecified.

## State & files
`${XDG_DATA_HOME:-$HOME/.local/share}/monid/honeypot-killer/screens/<address>.md`

---

## STEP 0 — Anti-impersonation (verify the REAL contract) — FIRST, always

The #1 scam is a fake token reusing a real name/ticker. A perfect scan of the WRONG address is the
trap, not the safeguard.
- User gave a **0x address** → use it, but resolve the canonical one and compare (warn on mismatch).
- User gave a **name/ticker only** → do NOT guess. Resolve via CoinGecko + the project's official
  site/Twitter, show your source, and flag if several contracts share the name.

Legitimacy sanity (real coins have a market; fresh scams don't):
```bash
monid run -p api.strale.io -e /x402/crypto-price \
  --query '{"coin":"<name-or-symbol>","vs_currencies":"usd"}' --wait 30 -j -o price.json
```
No market cap / not listed → treat as "unlisted, thin" (a caution, never a clear).

---

## STEP 1 — Honeypot / security core (source A: GoPlus via Strale)

**`api.strale.io /x402/token-security-check`** · ~$0.024
```bash
monid run -p api.strale.io -e /x402/token-security-check \
  --query '{"contract_address":"<0x...>","chain_id":"1"}' --wait 30 -j -o secA.json
```
Read: `is_honeypot`, `cannot_sell_all`, `buy_tax`, `sell_tax`, `is_mintable`, `can_take_back_ownership`,
`transfer_pausable`, `is_blacklisted`, `is_open_source`, `is_proxy`, `owner_address`, `creator_address`, `holder_count`, lp fields.

## STEP 2 — Second-source cross-check (source B: GoPlus via Heurist)

Redundancy catches one source being stale/wrong. **Tool made by Heurist.**
**`mesh.heurist.xyz /x402/agents/GoplusAnalysisAgent/fetch_security_details`** · ~$0.011 · POST body
```bash
monid run -p mesh.heurist.xyz -e /x402/agents/GoplusAnalysisAgent/fetch_security_details \
  -i '{"contract_address":"<0x...>","chain_id":"1"}' --wait 30 -j -o secB.json
```
**Rule: if A and B DISAGREE on honeypot / sellability, treat the token as unsafe** (escalate to KILL).
Unknown == unsafe.

## STEP 3 — Contract verification
**`api.strale.io /x402/contract-verify-check`** · ~$0.011
```bash
monid run -p api.strale.io -e /x402/contract-verify-check \
  --query '{"contract_address":"<0x...>","chain_id":"1"}' --wait 30 -j -o verify.json
```
`is_verified` false / not open source → red flag. `is_proxy` true → upgradeable (rules can change after buy).

## STEP 4 — Holder concentration
**`blockrun.ai /api/v1/surf/token/holders`** · ~$0.008
```bash
monid run -p blockrun.ai -e /api/v1/surf/token/holders \
  --query '{"address":"<0x...>","chain":"ethereum"}' --wait 30 -j -o holders.json
```
Top-10 % and top-1 % **excluding** LP pools, burn (0xdead/0x0), known lockers, and exchange wallets.
A single non-LP, non-exchange wallet with a large share = dump risk.

## STEP 5 — Deployer rap sheet (NEW in v2) — has this wallet rugged before?

Take `creator_address` from STEP 1 and pull its history. A deployer that spun up tokens that died /
dumped is the single strongest off-the-shelf scam signal.
**`blockrun.ai /api/v1/surf/wallet/history`** · ~$0.008
```bash
monid run -p blockrun.ai -e /api/v1/surf/wallet/history \
  --query '{"address":"<creator_address>"}' --wait 30 -j -o deployer.json
```
Look for: many token deployments, prior contracts now dead/zeroed, fast liquidity pulls.
Optional cluster of connected insider wallets: `blockrun.ai /api/v1/pm/polymarket/wallet/:address/cluster` (~$0.006).

## STEP 6 — Unlock / vesting
**`blockrun.ai /api/v1/surf/token/tokenomics`** · ~$0.008 · `--query '{"address":"<0x...>","chain":"ethereum"}'`
Flag any large unlock within 30 days. Empty for brand-new microcaps → say "no schedule found", don't imply safety.

---

## STEP 7 — Verdict

**🚫 KILL — any single hard fail:**
- `is_honeypot` true on EITHER source, or `cannot_sell_all` true
- **Sources A and B disagree on sellability** (unknown == unsafe)
- `sell_tax` or `buy_tax` > 15%
- `is_mintable` true with a live (non-renounced) owner
- ownership not renounced AND (`can_take_back_ownership` / `transfer_pausable` / blacklist controls)
- contract not verified / not open source
- **deployer rap sheet shows prior rug/deploy-and-dump pattern**

**⚠️ CAUTION — any soft flag:**
- `is_proxy` true (upgradeable — rules can change after you buy)
- top-10 > 50%, or a single non-LP/non-exchange wallet > 10%
- buy/sell tax 5–15%
- LP not locked/burned, or thin liquidity
- large unlock within 30 days
- not listed on CoinGecko / no market cap (too new / thin)

**✅ CLEAR-ish — none of the above.** Still: on-chain only, DYOR, not advice, CLEAR ≠ safe.

## Optional / advanced
- **Real transaction simulation** (beats pattern-matching for novel traps): spin a sandbox, fork the
  chain, simulate buy→sell. `blockrun.ai /api/v1/modal/sandbox/create` (~$0.011 flat ≤300s). Use when
  sources are thin or the token is minutes old.
- **Sniper/bot share:** `emc2ai.io /x402/bitquery/tokensniping/raw` (~$0.94).
- **Discover new launches:** `emc2ai.io /x402/bitquery/latest-pairs/raw` (~$0.28) then screen top N.
- **Post-buy approval safety:** `api.strale.io /x402/approval-security-check` (~$0.011).

## Output: verdict card
```markdown
🛡️ Honeypot Killer — <NAME> (<chain>)
VERDICT: 🚫 KILL              ← or ⚠️ CAUTION / ✅ CLEAR
Contract: <0x…> — official? <verified vs CoinGecko/site, or ⚠️ unconfirmed>

Honeypot test     A(GoPlus/Strale): 🚫 honeypot   B(GoPlus/Heurist): 🚫 honeypot   → agree ✅
Sellable          🚫 NO — buy works, sell blocked
Tax (buy/sell)    ✅ 0% / 0%        (clean-looking ≠ safe)
Mint              ✅ none
Ownership         ✅ renounced / 🚫 active + can blacklist
Verified source   ✅ open-source     Proxy(upgradeable): ⚠️ yes / ✅ no
Distribution      ⚠️ top-10 63% · largest non-LP 18% 0xab…
Deployer rap      🚫 creator 0xad… deployed 7 tokens, 5 now dead
Unlocks (30d)     ✅ none

Why: <the worst flag in one line>
DYOR · on-chain snapshot · not financial advice · Honeypot Killer never buys or signs · (security via GoPlus, second source by Heurist)
```

## Cost summary
| Run | Cost |
|---|---|
| Full v2 screen (STEP 0–6) | ~$0.075/token |
| + real sandbox simulation | +$0.011 |
| + sniper check | +$0.94 |

Report `cost.value`. Before >$5 cumulative, check `monid balance`.

## Honesty (keep in every verdict)
- On-chain SNAPSHOT at scan time; proxy/upgradeable contracts and team actions can change after.
- Two GoPlus sources reduce but don't eliminate false-clears; a novel zero-day trap can still pass — a CLEAR verdict is NOT "safe to buy".
- Verify the OFFICIAL contract first; a clean scan of an impersonation token is the trap.
- Read-only. Never buys, approves, signs, or recommends a purchase. Not financial advice.

## Validated endpoints
| Purpose | Provider / Endpoint | Cost | Params |
|---|---|---|---|
| Legitimacy / price (anti-impersonation) | `api.strale.io /x402/crypto-price` | $0.0055 | `--query` (`coin`,`vs_currencies`) |
| Security source A (GoPlus) | `api.strale.io /x402/token-security-check` | $0.024 | `--query` (`contract_address`,`chain_id`) |
| Security source B (GoPlus via Heurist) | `mesh.heurist.xyz /x402/agents/GoplusAnalysisAgent/fetch_security_details` | $0.011 | `-i` (`contract_address`,`chain_id`) |
| Contract verified? | `api.strale.io /x402/contract-verify-check` | $0.011 | `--query` (`contract_address`,`chain_id`) |
| Holders | `blockrun.ai /api/v1/surf/token/holders` | $0.008 | `--query` (`address`,`chain`) |
| Deployer rap sheet | `blockrun.ai /api/v1/surf/wallet/history` | $0.008 | `--query` (`address`) |
| Insider cluster (opt) | `blockrun.ai /api/v1/pm/polymarket/wallet/:address/cluster` | $0.0055 | path `:address` |
| Unlocks | `blockrun.ai /api/v1/surf/token/tokenomics` | $0.008 | `--query` (`address`,`chain`) |
| Real simulation sandbox (opt) | `blockrun.ai /api/v1/modal/sandbox/create` | $0.011 | body |
| Sniper/bot (opt) | `emc2ai.io /x402/bitquery/tokensniping/raw` | $0.94 | body |
| New pairs (opt) | `emc2ai.io /x402/bitquery/latest-pairs/raw` | $0.28 | body (`chain`,`limit`) |
