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
| `journal/` | Live positions ledger (`positions.md`) + closed archive (`closed.md`) | Updated atomically with the routine that touches it; same commit message as that routine |

After generating a briefing, always save it to the correct directory and commit with the exact message format above.

## Trading Context

- **Strategy**: Swing trading, 1–3 month holding horizon
- **Universe**: LQ45 and IDX30 stocks only — never recommend stocks outside this universe
- **Persona**: Act as a professional Indonesian portfolio manager
- **Language**: Output briefings in Bahasa Indonesia

## Trading Discipline (enforced by GLOBAL RULES in `prompts.md`)

- Max **5** swing positions open across `journal/positions.md` (status PLANNED or TRIGGERED)
- Max **2** positions per sector (Banking, Metals, Energy, OilGas, Consumer, CPO, Property, Auto, Healthcare, Tech, Telco, Tower, Power, Cement, Conglomerate)
- Every Entry / TP1 / TP2 / SL must cite a chart anchor (20D high, MA50, swing low + date, gap fill, Fibonacci, opening-range for intraday). Round-number-only levels are invalid — drop the pick.
- R:R ≥ **2.00** measured as `(TP1 − Entry) / (Entry − SL)`. TP1-base only; do not measure to TP2. If a setup fails, drop it — do not stretch TP1 to force the ratio.
- Conviction tiers: **FULL** 100% / **HALF** 50% / **TEST** 25% of the standard position unit.
- Position expires after **10 trading days** untriggered (auto-moved to `closed.md` as `EXPIRED`).
- Ledger writes (`journal/positions.md`, `journal/closed.md`) happen in the **same commit** as the brief that triggered them. Never split brief and ledger across commits.
- Mid-day intraday picks do NOT consume the 5-position cap — they close same day and stay out of the ledger.
- Dividend captures are NOT written to the ledger in v1 — graded inline in each Monday's dividend brief.

## Prompts

The four routine prompts (daily, mid-day, weekly, dividend) are stored in `prompts.md`. Each prompt specifies word limits, source priorities, and output format — follow them exactly.

## Web Search Sources (priority order)

IDX, Bisnis.com, Kontan, CNBC Indonesia, Stockbit, RTI Business, Mandiri Sekuritas / Mirae / BRIDS research, Investing.com (economic calendar), Bareksa, sahamidx.com, Investortrust.

## Allowed Domains

Pre-approved fetches (no permission prompt): `www.cnbcindonesia.com`, `stocksetup.kontan.co.id`, `investortrust.id`, `swa.co.id`.
