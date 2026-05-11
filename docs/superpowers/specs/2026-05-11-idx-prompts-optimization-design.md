# IDX Trading Prompts Optimization — Design

**Date:** 2026-05-11
**Status:** Approved, ready for implementation plan
**Scope:** All four routine prompts in `prompts.md` (daily, mid-day, weekly, dividend)

## Context

`prompts.md` contains four scheduled prompts that generate IDX swing-trading briefings. Audit of the only saved briefing ([briefings/2026-05-11.md](../../../briefings/2026-05-11.md)) revealed:

1. **Round-number price levels** — every Entry/TP/SL is a clean round number with no chart anchor. Reads as model-invented, not chart-derived.
2. **Ambiguous R:R** — labels like "R:R 1:2.5" don't specify TP1 vs TP2 base; numbers don't reconcile to a single formula.
3. **No sizing or exposure logic** — 5 picks shown in isolation; no max-positions rule, no sector cap, no conviction tiers.
4. **Weak avoid-list invalidation** — most avoid entries lack a price level that would falsify the avoid thesis.
5. **No feedback loop** — no journal of open positions, no grading of prior calls. The user cannot measure hit-rate from the briefings alone.

## Goals

- Force chart-anchored levels for every Entry/TP/SL.
- Make R:R unambiguous and self-checking.
- Track open positions in a single ledger so portfolio rules (max positions, sector cap, conviction tiers) are enforceable.
- Auto-grade prior calls in every routine so the journal accumulates a track record.

## Non-Goals

- No backtest harness or analytics scripts in v1. Hit-rate is computable from `journal/closed.md` later.
- No automation/scheduling changes. Prompts still pasted manually at routine times.
- No tracking of dividend captures in the swing journal (deferred — see Open Questions).

## Approach

**Approach 3 (selected)** — Shared `GLOBAL RULES` preamble in `prompts.md` + structured `journal/positions.md` and `journal/closed.md` ledger files. Each routine reads the ledger at start, grades open positions, generates its brief, writes ledger changes back, and commits atomically.

Rejected alternatives:

- **Approach 1 (surgical edits to standalone prompts):** rules drift between routines, self-grading is fragile (model re-derives state from dated brief files each run).
- **Approach 2 (shared preamble, no ledger):** still no source of truth for open positions — "self-grading" becomes theatre.

## File Layout

```
prompts.md
  ├── GLOBAL RULES   ← shared preamble (role, universe, evidence, R:R, sizing, sources)
  ├── A. Daily pre-market routine
  ├── B. Mid-day check routine
  ├── C. Weekly outlook routine
  └── D. Dividend tracker routine

journal/
  ├── positions.md   ← open ledger (PLANNED + TRIGGERED rows)
  └── closed.md      ← append-only archive of resolved positions

briefings/, midday/, weekly/, dividends/   ← unchanged, dated brief files
```

## Per-Routine Flow (same shape for all four)

1. **READ** `journal/positions.md`.
2. **GRADE** every row against today's data:
   - `PLANNED` and entry zone touched yesterday → set `TRIGGERED`, fill `Triggered` date.
   - `TRIGGERED` and TP1 or SL touched yesterday → move to `closed.md` with outcome.
   - `PLANNED` and `Expires` reached → move to `closed.md` as `EXPIRED`.
3. **BRIEF** — generate today's brief, opening with a `Position Review` section that summarizes the grades from step 2.
4. **UPDATE** — append new picks as `PLANNED`, persist grade updates.
5. **COMMIT** — single commit containing brief + ledger changes, using the existing per-routine commit message format.

## GLOBAL RULES (preamble block)

```
GLOBAL RULES (berlaku untuk semua routine di file ini)

ROLE: Profesional portfolio manager saham Indonesia, spesialis swing
trading 1-3 bulan. Universe LQ45/IDX30 only — saham di luar universe
tidak boleh muncul di rekomendasi, watchlist, atau dividend tracker.

EVIDENCE RULE (anti-hallucination):
- Setiap angka konkret (harga, %, flow, yield, level teknikal) WAJIB
  bersumber dari web search. Jika tidak ditemukan, tulis "N/A" —
  jangan dikira-kira, jangan dibulatkan dari memori.
- Sertakan satu baris "Sumber:" di akhir setiap section utama dengan
  2-4 source dari daftar prioritas.
- Jika web search gagal total / data <24 jam tidak tersedia, output:
  "Data tidak tersedia — briefing dibatalkan" dan STOP. Jangan paksa
  menulis brief dengan data lama.

CHART ANCHOR RULE (level harus berbasis chart, bukan angka bulat):
Setiap Entry, TP1, TP2, SL harus diberi tag anchor dalam kurung.
Anchor yang valid:
  - 20D high/low, 52W high/low
  - Prior swing high/low (sebutkan tanggal swing)
  - MA20 / MA50 / MA200
  - Fibonacci 38.2 / 50 / 61.8 (sebutkan range swing)
  - Gap fill (sebutkan tanggal gap)
  - Opening range high/low (intraday only — mid-day routine)
  - Round-number psychological — HANYA jika confluence dengan anchor
    lain di atas
Format: `TP1 8.050 (20D high, 24/4)` atau `SL 7.625 (MA50)`.
Level tanpa anchor = level invalid, drop pick tersebut.

R:R FORMULA (eksplisit, tidak ambigu):
  - Entry = midpoint dari entry zone (atau single price jika 1 level)
  - Risk = Entry − SL (absolute)
  - Reward = TP1 − Entry   ← TP1, BUKAN TP2
  - R:R = Reward / Risk
  - Minimum 1:2 ke TP1. Jika setup terbaik tidak mencapai 1:2, DROP
    pick tersebut. Jangan dorong TP1 lebih jauh untuk memaksa rasio.
  - Tunjukkan perhitungan: "R:R = (8.050−7.800)/(7.800−7.625) = 1.43" —
    pick ini fail, drop.

PORTFOLIO RULES (enforce via journal/positions.md):
  - Max 5 swing positions bersamaan (status: PLANNED atau TRIGGERED).
  - Max 2 posisi per sektor (banking, metals, energy, consumer, dll).
  - Conviction tier per pick:
      FULL = 100% standard unit (high-conviction, setup + katalis kuat)
      HALF = 50% (1 leg lemah, atau confluence belum sempurna)
      TEST = 25% (probe, butuh konfirmasi tambahan)
  - Jika slot full atau sektor cap kena: pick baru harus replace pick
    existing (sebutkan mana yang di-drop dan kenapa).
  - Mid-day intraday picks TIDAK consume slot — close same day.

OUTPUT DISCIPLINE:
  - Skip preamble, filler, dan disclaimer.
  - Bahasa Indonesia, bullet rapi, word limit per routine ditegakkan.
  - Setiap routine WAJIB diawali "Position Review" (grade open positions
    dari journal/positions.md) sebelum section baru.

SOURCE PRIORITY:
  IDX (idx.co.id), Bisnis.com, Kontan, CNBC Indonesia, Stockbit, RTI,
  riset Mandiri Sekuritas / Mirae / BRIDS, Investing.com (kalender),
  Bareksa, sahamidx.com, Investortrust.
```

## Routine A — Daily Pre-Market (07:30 WIB)

Output structure:
1. **Position Review** — table of every PLANNED/TRIGGERED row with status, mark-to-market vs entry, days remaining until expiry, action item (hold/cut/scale).
2. **Snapshot Makro** — IHSG, FX, Wall Street, Asia pagi, komoditas, event hari ini.
3. **New Picks** — maximum 3 new entries, LQ45/IDX30 only. Must respect 5-slot total and 2-per-sector caps; if full, declare which existing pick is replaced and why. Each pick shows: kode, sektor, conviction tier, katalis (with specific numbers), Entry/TP1/TP2/SL (with anchors), R:R calculation, sources.
4. **Avoid List** — every entry must have an invalidation level.
5. **One-Liner Strategi** — bias + rationale.

Word limit: 600.
Holiday handling: output "Pasar libur" and STOP after only running the expiry-grade step.
Files: write to [briefings/](../../../briefings/) and update `journal/`, commit `Daily IDX brief - YYYY-MM-DD`.

## Routine B — Mid-Day Check (12:30 WIB)

Output structure:
1. **Position Review** — sesi 1 intraday touches only. Did any PLANNED trigger? Did any TRIGGERED hit TP1 or SL during the morning session?
2. **Sesi 1 Snapshot** — IHSG sesi 1 close, volume, foreign flow, sector gainers/laggards, anomalies.
3. **Leadership Check** — sektor rotation analysis, implication.
4. **Setup Sesi 2** — up to 3 intraday picks; anchors limited to opening-range or sesi 1 high/low. **Not written to ledger** (close same day).
5. **Alert & Risk** — news flow, UMA/suspend/AR, Asia siang.
6. **One-Liner Sesi 2** — continuation/reversal/sideways + rationale.

Word limit: 400.
Holiday handling: "Sesi 1 tidak berjalan normal" + STOP.
Files: write to [midday/](../../../midday/) and update `journal/`, commit `Mid-day IDX brief - YYYY-MM-DD`.

## Routine C — Weekly Outlook (Sunday 19:00 WIB)

Output structure:
1. **Position Review** — heaviest grading event of the week. 5 days of price action graded against every open row.
2. **Recap Minggu Lalu** — IHSG WoW, sektor terbaik/terburuk, ekstrem LQ45, tema dominan.
3. **Kalender Event Minggu Ini** — domestic, global, corporate actions, index events. Tabular.
4. **Tema Sunrise** — 2-3 narratives + LQ45/IDX30 beneficiaries.
5. **Proposed Week Picks (≤5)** — explicit reconciliation against the 5-slot cap: which fit, which require replacing existing PLANNED rows, which to drop.
6. **Risk Map** — 3 critical risks + bear-case trigger.
7. **One-Liner Weekly Bias** — risk-on / risk-off / mixed + key catalyst.

Word limit: 800.
Files: write to [weekly/](../../../weekly/) and update `journal/`, commit `Weekly IDX brief - YYYY-MM-DD`.

## Routine D — Dividend Tracker (Monday 06:30 WIB)

Output structure:
1. **Position Review (cross-reference only)** — does any open swing position have a cum/ex date this week? Note inline; do not double-track.
2. **Cum Date 7 Hari** — LQ45/IDX30 table.
3. **Cum Date 8-14 Hari** — same table format.
4. **Top 3 Dividend Capture Opportunity** — yield + fundamental check + recovery pattern + entry/exit strategy + risks.
5. **Dividend Trap Warning** — high-yield red flags.
6. **Rekap Ex-Date 1 Minggu Terakhir** — recovery-pattern grading, inline.

Word limit: 500.
**Dividend captures are NOT written to `journal/positions.md`** in v1 — graded inline only (see Open Questions for v2 upgrade).
Files: write to [dividends/](../../../dividends/) and update `journal/positions.md` only if expiry-grades hit on existing rows, commit `Dividend tracker - YYYY-MM-DD`.

## Ledger Schemas

### `journal/positions.md`

```markdown
# Open Positions — IDX Swing Journal

<!-- Status: PLANNED (entry not hit) | TRIGGERED (entry hit, holding)  -->
<!-- Sector tags: Banking, Metals, Energy, OilGas, Consumer, CPO,      -->
<!-- Property, Auto, Healthcare, Tech, Telco, Tower, Power, Cement,    -->
<!-- Conglomerate                                                      -->

| Kode | Sektor | Status | Conviction | Entry (anchor) | TP1 (anchor) | SL (anchor) | R:R | Katalis | Opened | Triggered | Expires | Source |
|------|--------|--------|------------|----------------|--------------|-------------|-----|---------|--------|-----------|---------|--------|
```

Column rules:
- `Status`: only PLANNED or TRIGGERED. Closed positions move to `closed.md`.
- `R:R`: precomputed at entry midpoint to TP1. Must be ≥ 2.00 or the row is invalid (do not write it).
- `Triggered`: date entry zone hit; `—` while PLANNED.
- `Expires`: `Opened` + 10 trading days. If still PLANNED at expiry, move to `closed.md` as EXPIRED.
- `Source`: relative path to the brief that introduced the pick.
- TP2 is **intentionally** not in the ledger — it remains aspirational guidance in the brief. Ledger grades only against TP1 for clean pass/fail.

### `journal/closed.md`

```markdown
# Closed Positions — IDX Swing Journal

| Kode | Sektor | Conviction | Entry | TP1 | SL | Opened | Closed | Outcome | Close Price | Return % | Source |
|------|--------|------------|-------|-----|----|----|--------|---------|-------------|----------|--------|
```

Outcome enum: `TP1_HIT` · `SL_HIT` · `EXPIRED` · `MANUAL_CLOSE`.

## Backfill Decision

**Wipe & forget** (option a) — the ledger starts empty. The 5 picks in [briefings/2026-05-11.md](../../../briefings/2026-05-11.md) are pre-ledger history and are not imported. The first routine after bootstrap may propose new picks under the new discipline.

## CLAUDE.md Changes

1. Extend the Output Conventions table with a `journal/` row (live ledger; updated atomically with the routine that touches it; same commit message as the routine).
2. Add a "Trading Discipline" section that mirrors the GLOBAL RULES key parameters (5-position cap, 2-per-sector, R:R ≥ 2.00 to TP1, conviction tiers, 10-trading-day expiry, atomic commit rule).
3. Allowed Domains list stays unchanged.

## Bootstrap (one-time)

1. Create `journal/positions.md` with empty schema (table header only).
2. Create `journal/closed.md` with empty schema.
3. Commit: `chore: bootstrap swing journal (positions + closed)`.

## Key Parameters (defaults, easy to tune later)

| Parameter | Value | Lives in |
|-----------|-------|----------|
| Max open positions | 5 | GLOBAL RULES + CLAUDE.md |
| Max per sector | 2 | GLOBAL RULES + CLAUDE.md |
| Daily new picks max | 3 | Routine A |
| Mid-day intraday picks max | 3 | Routine B |
| Weekly proposed picks max | 5 | Routine C |
| Minimum R:R to TP1 | 2.00 | GLOBAL RULES |
| Position expiry | 10 trading days | GLOBAL RULES |
| Conviction tiers | 100% / 50% / 25% | GLOBAL RULES |

## Open Questions / v2 Candidates

- **Dividend ledger (v2):** add `journal/dividends.md` with the same shape if hit-rate tracking on dividend captures becomes valuable.
- **Sector taxonomy formalization:** current sector list is informal; could lock to IDX sector codes if disagreements arise.
- **Half-close behavior:** ledger collapses TP1/TP2 into a single graded exit at TP1. If real-world practice diverges (e.g., scale at TP1, hold remainder for TP2), revisit ledger schema.

## Out of Scope

- Automated scheduling / cron / GitHub Actions.
- Analytics scripts to compute hit-rate from `closed.md`.
- Position size in IDR or share count (only conviction tier as % of standard unit).
- Order routing or broker integration.
