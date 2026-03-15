---
title: "Building a Prediction Market to Equity Signal Engine"
date: 2026-03-15T12:00:00+01:00
permalink: prediction-market-equity-signal-engine
description: How I built a serverless system that monitors Polymarket prediction markets and generates trading signals when equities haven't yet reacted to probability shifts
categories:
  - blog
tags:
  - AWS
  - TypeScript
  - Serverless
  - Finance
---

Prediction markets are surprisingly good at pricing in future events. Equity markets are too — but sometimes they lag behind. This project exploits that gap.

## The idea

When a prediction market reprices an event — say, the probability of a Fed rate cut jumps from 34% to 46% — rate-sensitive equities like small-cap ETFs should move too. Sometimes they do immediately. Sometimes they don't.

When they don't, that's a signal.

## How it works

The system runs on AWS and polls two data sources every 5 minutes:

1. **Polymarket** — fetches current probabilities for tracked prediction markets via the CLOB API.
2. **Financial Modeling Prep** — fetches equity prices for a list of symbols (IWM, QQQ, XLE, etc.).

Both are stored as time-series snapshots in DynamoDB. A signal engine then compares the two:

```
EventBridge (every 5 min)
        |
  ┌─────┴─────┐
  v            v
Polymarket   Equity
Ingestion    Ingestion
  |            |
  v            v
DynamoDB     DynamoDB
  └─────┬──────┘
        v
  Signal Engine
        |
        v
  Slack Alert
```

## Signal detection

The logic is intentionally simple for the MVP. For each market-to-equity mapping:

1. **Probability change** — compare the last two Polymarket snapshots. If the change is less than 8 percentage points, skip it. We only care about meaningful moves.
2. **Equity reaction** — check if the linked equity has already moved more than 2% in the last hour. If it has, the market already priced it in.
3. **Score** — compute a weighted score:

```
score = abs(pm_change) * weight - abs(equity_change) + volume_factor
```

The `weight` comes from a manually curated mapping that encodes how strongly a prediction market outcome should affect a given equity. The `volume_factor` adds confidence — a high-volume market repricing is more meaningful than a low-volume one.

If the score exceeds 0.5, a signal is generated and sent to Slack.

## The mapping layer

This is the most opinionated part of the system. Each mapping connects a Polymarket market to an equity and encodes the expected relationship:

```json
{
  "market_id": "0xe7f2...",
  "market_slug": "fed-rate-cut-by-march-2026-meeting",
  "symbol": "IWM",
  "direction": "positive",
  "weight": 0.8,
  "rationale": "Small caps benefit from lower rates"
}
```

Right now, the mappings are maintained by hand. The current set covers:

- **Fed rate cuts** (March and June 2026 meetings) → IWM, QQQ
- **US recession probability** → QQQ, IWM (negative)
- **Crude oil hitting $90** → XLE
- **US-China tariff escalation** → QQQ (negative)

The mapping file is the main thing you'd tweak as new markets appear on Polymarket.

## Architecture decisions

**Why serverless?** The system polls every 5 minutes and does almost nothing in between. Lambda + EventBridge is a natural fit and costs under $2/month.

**Why DynamoDB?** The access patterns are simple — write snapshots keyed by market/symbol ID, query the latest N by sort key. On-demand pricing keeps costs proportional to usage.

**Why not a database for mappings?** With ~10 mappings, a JSON file bundled into the Lambda is simpler than another table. It's versioned in git and deployed with the code.

**Secrets in Secrets Manager** — API keys for Financial Modeling Prep and the Slack webhook URL are stored in AWS Secrets Manager, not environment variables. The shared library caches the secret after the first cold-start fetch.

## Project structure

The repo is an npm workspaces monorepo with four packages:

```
packages/
  shared/       Types, DynamoDB client, secrets, config
  ingestion/    Polymarket + FMP Lambda handlers
  signals/      Signal detection engine
  alerts/       Slack webhook sender
infra/          CDK stack (tables, Lambdas, EventBridge)
config/         mappings.json
```

All infrastructure is defined in a single CDK stack — three DynamoDB tables, four Lambda functions, and two EventBridge schedules.

## What a signal looks like

When the system detects a divergence, it posts a Slack message:

```
📈 Prediction Market Signal

Market:    fed-rate-cut-by-march-2026-meeting
Odds move: 34% → 46% (+12.0pp)

Linked asset:  IWM
Equity move:   +0.40%

Confidence score: 71

Market "fed-rate-cut-by-march-2026-meeting" probability increased
by 12.0pp. Linked equity IWM moved only 0.4%.
Small caps benefit from lower rates. Score: 0.71.
```

## Limitations and next steps

This is an MVP. Some things I'd like to improve:

- **Hourly change calculation** — FMP provides 24h change, not 1h. Computing true 1h change requires comparing against stored snapshots from an hour ago.
- **Automatic market discovery** — instead of manually maintaining mappings, use an LLM to classify new Polymarket markets and suggest equity links.
- **Backtesting** — replay historical snapshots to measure how often the signal actually preceded an equity move.
- **ML scoring** — replace the linear scoring formula with something that learns from signal outcomes.

## Cost

Running this on AWS costs almost nothing:

| Component | Monthly cost |
|-----------|-------------|
| Lambda (4 functions, every 5 min) | Free tier |
| DynamoDB (on-demand) | ~$1 |
| Secrets Manager | ~$0.40 |
| EventBridge | Free tier |
| **Total** | **~$1.50/month** |

The most expensive part is probably the Financial Modeling Prep API key if you exceed their free tier of 250 requests/day.
