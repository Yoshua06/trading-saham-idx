# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repo is a personal trading journal for IDX (Indonesian Stock Exchange) swing trading. It stores AI-generated market briefings as dated markdown files, committed to git for historical tracking.

## Output Conventions

All generated briefings are saved as markdown files with the naming pattern `YYYY-MM-DD.md` under these directories:

| Directory | Prompt | Commit message pattern |
|-----------|--------|------------------------|
| `briefings/` | Daily pre-market (07:30 WIB) | `Daily IDX brief - YYYY-MM-DD` |
| `midday/` | Mid-day session 2 check (12:30 WIB) | `Mid-day IDX brief - YYYY-MM-DD` |
| `weekly/` | Sunday weekly outlook (19:00 WIB) | `Weekly IDX brief - YYYY-MM-DD` |
| `dividends/` | Monday dividend tracker (06:30 WIB) | `Dividend tracker - YYYY-MM-DD` |

After generating a briefing, always save it to the correct directory and commit with the exact message format above.

## Trading Context

- **Strategy**: Swing trading, 1–3 month holding horizon
- **Universe**: LQ45 and IDX30 stocks only — never recommend stocks outside this universe
- **Persona**: Act as a professional Indonesian portfolio manager
- **Language**: Output briefings in Bahasa Indonesia

## Prompts

The four routine prompts (daily, mid-day, weekly, dividend) are stored in `prompts.md`. Each prompt specifies word limits, source priorities, and output format — follow them exactly.

## Web Search Sources (priority order)

IDX, Bisnis.com, Kontan, CNBC Indonesia, Stockbit, RTI Business, Mandiri Sekuritas / Mirae / BRIDS research, Investing.com (economic calendar), Bareksa, sahamidx.com, Investortrust.

## Allowed Domains

Pre-approved fetches (no permission prompt): `www.cnbcindonesia.com`, `stocksetup.kontan.co.id`, `investortrust.id`, `swa.co.id`.
