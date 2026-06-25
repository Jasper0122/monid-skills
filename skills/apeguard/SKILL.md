---
name: apeguard
version: 0.1.0
description: A crypto new-launch safety screener — run BEFORE you ape into a token. Verifies the official contract, then runs an on-chain safety battery (honeypot / can-you-sell, buy-sell tax, mint authority, ownership renounce, contract verification, holder concentration, upcoming unlocks) via the Monid data API and returns a GO / CAUTION / AVOID verdict with reasons. Read-only research — never buys, never signs. Use when asked to "screen this token", "打新", "rug check", "honeypot check", "is this token safe", "防骗", "check contract", or given a 0x token address.
---

# Apeguard — crypto launch safety screener

## Overview

Apeguard is a pre-buy due-diligence tool for crypto launches ("打新"). Its job is to stop you
getting scammed: it first nails down the **official** contract address (impersonation tokens are
the #1 scam), then runs an on-chain safety battery and returns a blunt **AVOID / CAUTION / CLEAR**
verdict with the specific red flags. It is read-only — it never buys, approves, or signs anything.

A clean scan is NOT a green light to buy — it only means the on-chain mechanics aren't obviously
rigged. Team/social rug risk, stealth upgrades, and post-launch dumps are not fully on-chain.

## Prerequisites

### Monid CLI requirement

Pay-as-you-go data marketplace CLI; no free tier.

**Initial setup:**
1. Account at https://app.monid.ai → add funds
2. API key from https://app.monid.ai/access/api-keys
3. `monid keys add -k <key> -l main` → `monid keys list`

**Preflight** (every invocation): `monid --version || npm install -g @monid-ai/cli`

### CLI rules
- Use CLI exclusively; never MCP tools. `NO_COLOR=1` for scripted output.
- `monid run -p <provider> -e <endpoint> --query '<json>' --wait <sec> -j -o <file>`
- `-o <file>` strips the envelope to `.output`. Inspect unclear endpoints: `monid inspect -p <provider> -e <endpoint>`
- On `402` → fund wallet at app.monid.ai.

### Chain naming (IMPORTANT — providers differ)
| chain | Strale `chain_id` | BlockRun `chain` |
|---|---|---|
| Ethereum | `1` | `ethereum` |
| BSC | `56` | `bsc` |
| Base | `8453` | `base` |
Default to Ethereum (`1` / `ethereum`) if the user doesn't say.

## State & files

`${XDG_DATA_HOME:-$HOME/.local/share}/monid/apeguard/`:
```
screens/<address>.md     # verdict card per token
```

## STEP 0 — Verify the OFFICIAL contract (anti-impersonation) — do this FIRST

The most common scam is a fake token reusing a real project's name/ticker. A perfect safety scan
of the WRONG address is worthless.
- If the user gave a **0x address**, use it but state: "scanning the address you gave — confirm it
  matches the project's official contract from their site/Twitter/CoinGecko."
- If the user gave a **name/ticker only**, do NOT guess an address. Ask for the official contract,
  or look it up and show the source you used. Flag if multiple contracts share the name.

## STEP 1 — Security core (honeypot / tax / mint / ownership)

**Endpoint:** `api.strale.io /x402/token-security-check` (GoPlus, 30+ chains) · **Cost:** ~$0.024

```bash
monid run -p api.strale.io -e /x402/token-security-check \
  --query '{"contract_address":"<0x...>","chain_id":"1"}' --wait 30 -j -o sec.json
```
Read (GoPlus fields): `is_honeypot`, `cannot_sell_all`, `buy_tax`, `sell_tax`, `is_mintable`,
`owner_address` / `can_take_back_ownership`, `transfer_pausable`, `is_blacklisted`,
`is_open_source`, `is_proxy`, `lp_holders` / locked %.

## STEP 2 — Contract verification

**Endpoint:** `api.strale.io /x402/contract-verify-check` · **Cost:** ~$0.011

```bash
monid run -p api.strale.io -e /x402/contract-verify-check \
  --query '{"contract_address":"<0x...>","chain_id":"1"}' --wait 30 -j -o verify.json
```
Read: `is_verified` / source available, `is_proxy`, compiler, contract name. Unverified = red flag.

## STEP 3 — Holder concentration

**Endpoint:** `blockrun.ai /api/v1/surf/token/holders` · **Cost:** ~$0.008

```bash
monid run -p blockrun.ai -e /api/v1/surf/token/holders \
  --query '{"address":"<0x...>","chain":"ethereum"}' --wait 30 -j -o holders.json
```
Compute: top-10 % and top-1 % **excluding** LP pools, burn (0xdead/0x0), and known lockers.
A single non-LP wallet holding a large share = dump risk.

## STEP 4 — Unlock / vesting (future dump risk)

**Endpoint:** `blockrun.ai /api/v1/surf/token/tokenomics` · **Cost:** ~$0.008

```bash
monid run -p blockrun.ai -e /api/v1/surf/token/tokenomics \
  --query '{"address":"<0x...>","chain":"ethereum"}' --wait 30 -j -o unlocks.json
```
Read upcoming unlock cliffs; flag any large unlock within 30 days. (May be empty for brand-new
microcaps — note "no schedule found" rather than implying safety.)

## STEP 5 — Verdict (apply rules)

**🚫 AVOID — any single hard fail:**
- `is_honeypot` true, or `cannot_sell_all` true (you can't exit)
- `sell_tax` > 15% (or buy_tax > 15%)
- `is_mintable` true (owner can print supply)
- ownership not renounced AND (`can_take_back_ownership` / `transfer_pausable` / `is_blacklisted` controls present)
- contract not verified / not open source

**⚠️ CAUTION — any soft flag (buy only with eyes open):**
- top-10 holders > 50% or a single non-LP wallet > 10%
- buy/sell tax 5–15%
- `is_proxy` true (upgradeable — rules can change after you buy)
- LP not locked/burned, or low liquidity
- large unlock within 30 days

**✅ CLEAR-ish — none of the above.** Still: DYOR, on-chain only, not advice.

## Optional modes
- **Sniper/bot check** (only if user asks; pricey): `emc2ai.io /x402/bitquery/tokensniping/raw` (~$0.94) — early-buyer bot/sniper share.
- **Discover new launches:** `emc2ai.io /x402/bitquery/latest-pairs/raw` (~$0.28) — latest deployed tokens + LP; then screen top N with STEP 1.
- **Post-buy approval safety:** `api.strale.io /x402/approval-security-check` (~$0.011) — flag drain-risk approvals on your wallet.

## Output: verdict card

Write to `screens/<address>.md` and echo:

```markdown
🛡️ Apeguard — <NAME or address-prefix> (<chain>)
VERDICT: 🚫 AVOID            ← or ⚠️ CAUTION / ✅ CLEAR-ish
Contract: <0x…> — ⚠️ confirm this is the project's OFFICIAL address

Hard checks
  honeypot / sellable   ✅ sellable
  sell / buy tax        ⚠️ 12% / 3%
  mint authority        🚫 owner can mint
  ownership             🚫 not renounced · can blacklist
  verified source       ✅ open-source

Distribution
  top-10 (excl LP)      ⚠️ 63%
  largest wallet        ⚠️ 18%  0xab…  (not LP)

Unlocks
  next 30 days          ✅ none found

Why: <one-line reason for the verdict — the worst flag>
DYOR · on-chain snapshot only, not financial advice · apeguard never buys or signs
```

## Cost summary

| Run | Cost |
|---|---|
| Core screen (STEP 1–4) | ~$0.05/token |
| + sniper check | +$0.94 |
| Discover + screen top 5 | ~$0.28 + 5×$0.05 ≈ $0.53 |

Report `cost.value`. Before >$5 cumulative, check `monid balance`.

## Honesty (keep in every verdict)
- This is an on-chain SNAPSHOT at scan time; mutable contracts (proxy) and team actions can change after.
- GoPlus/etc. catch known patterns; a novel scam can pass. **A CLEAR verdict is not "safe to buy".**
- Verify the OFFICIAL contract first — a clean scan of an impersonation token is the trap, not the safeguard.
- Read-only research. Apeguard never buys, approves, signs, or recommends a purchase. Not financial advice.

## Validated endpoints

| Purpose | Provider / Endpoint | Cost | Params |
|---|---|---|---|
| Security core (honeypot/tax/mint) | `api.strale.io /x402/token-security-check` | $0.024 | `--query` (`contract_address`,`chain_id`) |
| Contract verified? | `api.strale.io /x402/contract-verify-check` | $0.011 | `--query` (`contract_address`,`chain_id`) |
| Top holders | `blockrun.ai /api/v1/surf/token/holders` | $0.008 | `--query` (`address`,`chain`) |
| Unlock / vesting | `blockrun.ai /api/v1/surf/token/tokenomics` | $0.008 | `--query` (`address`,`chain`) |
| Sniper/bot (opt) | `emc2ai.io /x402/bitquery/tokensniping/raw` | $0.94 | `--query` |
| New pairs (opt) | `emc2ai.io /x402/bitquery/latest-pairs/raw` | $0.28 | `--query` |
| Approval safety (opt) | `api.strale.io /x402/approval-security-check` | $0.011 | `--query` (`address`,`chain_id`) |
