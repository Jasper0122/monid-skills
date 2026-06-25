# monid-skills

Agent skills powered by the [Monid](https://monid.ai) pay-as-you-go data API.
Install into any agent that reads `SKILL.md` (Claude Code, Codex, Cursor, …):

```bash
npx skills add <your-gh-username>/monid-skills
```

## Skills

| Skill | What it does |
|---|---|
| [`bookwatch-arb`](skills/bookwatch-arb/SKILL.md) | Cross-venue prediction-market arbitrage scout (Polymarket × Kalshi): matched pairs → drop junk → price both legs → net edge after the official Kalshi fee → ranked table with trust/thin/suspect flags. Read-only / paper. |

## Prerequisites

Each skill uses the **monid** CLI (pay-as-you-go; no free tier):

```bash
npm install -g @monid-ai/cli
monid keys add -k <your-key> -l main      # key from https://app.monid.ai/access/api-keys
monid keys list
```

Then trigger the skill in your agent — e.g. say **"find arbs"**.
