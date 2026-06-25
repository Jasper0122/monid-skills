# Bookwatch Arb Skill — Full Documentation

```yaml
---
name: bookwatch-arb
version: 0.1.0
description: A cross-venue prediction-market arbitrage scout. Finds the same real-world question priced differently on Polymarket vs Kalshi, prices both legs, computes net edge after the official Kalshi fee, and returns a compact ranked table with trust/thin/suspect flags. Read-only / paper — never places real orders. Use when asked to "find arbs", "套利", "scan odds", "cross-venue arb", "Polymarket vs Kalshi", "book watch", or "arbitrage scan".
---
```

## Overview

Bookwatch Arb surfaces divergences between two prediction-market venues that list the
**same** question. It pulls server-side matched pairs, drops illiquid novelty markets before
spending, prices both legs, and computes the net risk-free edge of locking a $1 payout after
the official Kalshi taker fee. Output is a single ranked table; each candidate is auto-flagged
🟢 trust / 🟡 thin / 🔴 suspect so the user never has to judge raw numbers. Clean arbs are rare
by nature — reporting zero is the normal, honest result.

## Prerequisites

### Monid CLI requirement

This skill requires **monid**, a pay-as-you-go data marketplace CLI. There is no free tier;
users must fund a small balance before the skill can fetch data.

**Initial setup:**
1. Create account at https://app.monid.ai
2. Add funds via the dashboard
3. Generate API key from https://app.monid.ai/access/api-keys
4. Save: `monid keys add -k <key> -l main`
5. Confirm: `monid keys list`

**Preflight check** (run at every invocation):
```bash
monid --version || npm install -g @monid-ai/cli
```

### CLI rules

- Use CLI exclusively; never MCP tools
- Set `NO_COLOR=1` for scripted output
- Workflow: `monid run -p <provider> -e <endpoint> --query '<json>' --wait <sec> -j -o <file>`
- The `-o <file>` flag strips the API envelope and outputs only `.output`
- All endpoints here are GET-style → pass params under `--query`
- Inspect unclear endpoints: `monid inspect -p <provider> -e <endpoint>`
- On `402` → wallet empty: `monid balance`, then top up at https://app.monid.ai

## State & files

Scan output lives in `${XDG_DATA_HOME:-$HOME/.local/share}/monid/bookwatch-arb/`:

```
config.json            # pages, price_cap, min_net_pct, junk keywords, exclusions
scans/YYYY-MM-DD.md    # ranked arb table per run
last_run.json          # per-mode timestamps
```

**First-run setup:**
```bash
DEST="${XDG_DATA_HOME:-$HOME/.local/share}/monid/bookwatch-arb"
mkdir -p "$DEST/scans"
```

## Configuration

On first run or when `config.json` is missing, use defaults (no interactive setup needed).
Override via questions only if the user asks to tune budget or coverage.

Example `config.json`:
```json
{
  "pages": 3,
  "price_cap": 22,
  "min_net_pct": 3.0,
  "fee_rate": 0.07,
  "junk_keywords": ["valorant","lol","league of legends","cs2","counter-strike","dota",
                    "rocket league","game 1","map 1"," vs."," vs ","over/under","o/u",
                    "to finish top","mvp","chopped","survivor","bachelor","masterchef"],
  "exclusions": {"keywords": []}
}
```

- `pages` — how many 100-pair pages to scan (wider = more coverage, more cost). Default 3.
- `price_cap` — max survivors to price after junk filter (budget guard). Default 22.
- `min_net_pct` — report threshold for actionable arbs. Default 3.0%.

## Pipeline

Run all stages silently; emit ONLY the Output digest. Never narrate steps or per-pair math.

---

## Stage 1 — Scan matched pairs (cheap, no prices)

**Endpoint:** `blockrun.ai /api/v1/pm/matching-markets/pairs`
**Cost:** ~$0.0055/page

```bash
monid run -p blockrun.ai -e /api/v1/pm/matching-markets/pairs \
  --query '{}' --wait 30 -j -o pairs_1.json
# page deeper with the previous response's pagination_key:
monid run -p blockrun.ai -e /api/v1/pm/matching-markets/pairs \
  --query '{"pagination_key":"<output.pagination.pagination_key>"}' --wait 30 -j -o pairs_2.json
```

**Steps:**
1. Fetch `pages` pages (default 3 → ~300 pairs) or until `pagination.has_more == false`
2. Keep only pairs with BOTH `POLYMARKET` and `KALSHI` legs present
3. `similarity` is ~always 100 here — it does NOT separate junk, so do not filter on it

---

## Stage 2 — Drop junk before paying for prices

No API calls. Pure title filter — this is where coverage stays cheap.

**Steps:**
1. `combined = (KALSHI.title + " " + POLYMARKET.title)`, lowercased
2. DROP if `combined` matches any `junk_keywords` (esports, single-match sports / player props, reality TV)
3. KEEP (priority): elections/politics, macro/econ (Fed, CPI, GDP, rates, recession), crypto price thresholds, company/regulatory/M&A, major awards
4. Apply `exclusions.keywords`
5. Take up to `price_cap` survivors forward; remember `junk_filtered = dropped count`

---

## Stage 3 — Price both legs

**Endpoints:** `blockrun.ai /api/v1/pm/polymarket/markets` · `blockrun.ai /api/v1/pm/kalshi/markets`
**Cost:** ~$0.0011/call each (~$0.0022 per pair)

```bash
# Polymarket leg (mid prices, no spread)
monid run -p blockrun.ai -e /api/v1/pm/polymarket/markets \
  --query '{"condition_id":"<pair.POLYMARKET.condition_id>"}' --wait 30 -j -o pm.json
# Kalshi leg (real top-of-book bid/ask)
monid run -p blockrun.ai -e /api/v1/pm/kalshi/markets \
  --query '{"ticker":"<pair.KALSHI.market_ticker>"}' --wait 30 -j -o ka.json
```

**Extract:**
- Polymarket `markets[0]`: `pm_yes=outcomes[0].price`, `pm_no=outcomes[1].price`, `pm_liq=liquidity_usd`, `title`, `category=last of tags`
- Kalshi `markets[0]`: `ky=outcome label=="Yes"`, `kn=outcome label=="No"` (each `.bid`/`.ask`), `ka_vol=dollar_volume`
- If either leg lacks a needed price field, SKIP the pair

---

## Stage 4 — Compute net edge

Lock a $1 payout the cheaper of two ways:

```
dirA_cost = pm_yes + kn.ask          # buy YES on Polymarket, NO on Kalshi
dirB_cost = ky.ask + pm_no           # buy YES on Kalshi, NO on Polymarket
best = cheaper; ka_leg = Kalshi ask used in best
gross    = 1 - best_cost
net_pct  = (gross - fee_rate * ka_leg * (1 - ka_leg)) * 100   # fee_rate = 0.07 Kalshi taker
spread   = max(ky.ask-ky.bid, kn.ask-kn.bid)
divergence_pts = abs(pm_yes - (ky.bid+ky.ask)/2) * 100
size_usd = min(pm_liq, ka_vol)                                # OI proxy, not true depth
illiquid = spread > 0.12 OR ka_vol < 50
status   = "illiquid" if illiquid else "arb" if net_pct>min_net_pct else "thin" if gross>0 else "none"
```

The Kalshi fee is the VERIFIED official formula: `ceil_to_cent(0.07 · C · P · (1−P))` taker
(maker 0.0175), where C = contracts, P = price.

---

## Cross-cutting: confidence flags

Auto-classify every `status=="arb"` candidate — do not make the user judge:

| flag | condition | meaning |
|---|---|---|
| 🟢 trust | `pm_liq≥1000 AND ka_vol≥1000 AND divergence_pts≤15` | liquid, equivalent — worth a look |
| 🟡 thin | arb but `pm_liq<1000` or `ka_vol<1000` | real-ish but small / illiquid |
| 🔴 suspect | `divergence_pts>15` | gap too big → non-equivalent or stale quote, NOT money |

---

## Output: digest format

Write one markdown file per run to `${XDG_DATA_HOME:-$HOME/.local/share}/monid/bookwatch-arb/scans/YYYY-MM-DD.md`,
and echo the same to the user. Emit nothing else.

```markdown
# Arb scan — YYYY-MM-DD HH:MM

Scanned N survivors (J junk filtered) · X 🟢 · Y 🟡 · Z 🔴 · cost ~$C

| conf | event | net% | buy | size$ | why-flag |
|------|-------|------|-----|-------|----------|
| 🟢 | Fed cuts in July? | 4.1 | KALSHI YES .58 + PM NO .40 | 12,400 | liquid, Δ6pt |
| 🔴 | Jean-Paul win Chopped | 78 | KALSHI YES .19 + PM NO .02 | 288 | 80pt gap, PM stale |

_Kalshi=real bid/ask · PM=mid(no spread) · size$=OI proxy · signal not guaranteed profit · paper only_
```

If zero `status=="arb"`: print `No arbs. Closest: <event> net <x>%` on one line and stop.

---

## Cadence: manual + scheduled

**Manual / ad hoc:**
- "find arbs" / "套利" → full pipeline, default `pages=3`, `price_cap=22`
- "scan deeper" → bump `pages` to 5–6 (confirm cost first)
- "show thin ones too" → include 🟡 and near-misses in the table
- "smart money" → run the leaderboard endpoint (see Validated endpoints) instead of the arb scan

**Scheduled (after a successful scan):**
Proactively offer recurring setup. Three options:

1. **OS cron** (recommend by default):
   ```bash
   crontab -e
   # Add: */15 9-16 * * 1-5 /usr/local/bin/claude --print "/bookwatch-arb" >> "${XDG_DATA_HOME:-$HOME/.local/share}/monid/bookwatch-arb/cron.log" 2>&1
   ```
2. **`/schedule` skill** (if installed): "schedule /bookwatch-arb every 15 min during market hours"
3. **`/loop` skill**: foreground interval runs

Store schedule choice in `last_run.json`.

---

## Cost summary

| Stage | Cost |
|---|---|
| Stage 1 (matched pairs, 3 pages) | ~$0.017 |
| Stage 3 (price 22 survivors × 2 legs) | ~$0.048 |
| **Per scan total** | **~$0.065** |

Report `cost.value` per run. A 15-min `watch` over one trading day (~28 runs) ≈ **$1.8**.
Before runs >$5 cumulative spend, check `monid balance` and confirm with user.

---

## Validated endpoints

| Purpose | Provider / Endpoint | Cost | Params |
|---|---|---|---|
| Matched pairs | `blockrun.ai /api/v1/pm/matching-markets/pairs` | $0.0055/call | `--query` (`pagination_key`) |
| Polymarket price | `blockrun.ai /api/v1/pm/polymarket/markets` | $0.0011/call | `--query` (`condition_id`) |
| Kalshi price | `blockrun.ai /api/v1/pm/kalshi/markets` | $0.0011/call | `--query` (`ticker`) |
| Smart money | `blockrun.ai /api/v1/pm/polymarket/leaderboard` | $0.0011/call | `--query` (`window`,`sort_by`,`order`,`limit`) |

## Honesty (verified — keep in every report)

- Kalshi prices are real top-of-book bid/ask; Polymarket `price` is mid (no spread → realized PM cost is worse than shown).
- `size_usd` is an OI/volume proxy, NOT true fillable depth (thin books are often empty at the quote).
- An "arb" is a DIVERGENCE SIGNAL, not guaranteed risk-free profit — cross-venue fund movement and settlement-rule differences are real risks.
- Read-only / paper. Never place real orders or give executable order instructions.
