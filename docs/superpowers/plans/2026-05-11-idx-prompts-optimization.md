# IDX Prompts Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the four routine prompts in `prompts.md` to enforce chart-anchored levels, explicit R:R, portfolio caps, and a self-grading journal — backed by a new `journal/positions.md` ledger.

**Architecture:** A shared `GLOBAL RULES` preamble at the top of `prompts.md` is referenced by all four routines (daily, mid-day, weekly, dividend). A `journal/` directory holds `positions.md` (live PLANNED/TRIGGERED rows) and `closed.md` (append-only archive). Each routine reads the ledger, grades open rows against the day's data, generates its dated brief, writes ledger changes back, and commits atomically.

**Tech Stack:** Markdown files only. No code, no build step. Verification is `grep`/`wc -l`/visual inspection.

**Spec:** [docs/superpowers/specs/2026-05-11-idx-prompts-optimization-design.md](../specs/2026-05-11-idx-prompts-optimization-design.md)

**Pre-existing state to know about:** The working tree has uncommitted edits to `prompts.md` from earlier (minor save-instruction wording). Task 2 fully overwrites `prompts.md`, so those edits will be naturally subsumed. If you want to discard them deliberately before starting, run `git checkout prompts.md` — otherwise no action needed.

---

### Task 1: Bootstrap `journal/` scaffold

**Files:**
- Create: `journal/positions.md`
- Create: `journal/closed.md`

- [ ] **Step 1: Create `journal/positions.md` with empty schema**

Write this exact content to `journal/positions.md`:

```markdown
# Open Positions — IDX Swing Journal

<!-- Status: PLANNED (entry not hit) | TRIGGERED (entry hit, holding)  -->
<!-- Sector tags: Banking, Metals, Energy, OilGas, Consumer, CPO,      -->
<!-- Property, Auto, Healthcare, Tech, Telco, Tower, Power, Cement,    -->
<!-- Conglomerate                                                      -->
<!-- R:R column must be >= 2.00. TP2 is intentionally NOT a column —   -->
<!-- the ledger grades only against TP1; TP2 is aspirational guidance  -->
<!-- in each brief.                                                    -->

| Kode | Sektor | Status | Conviction | Entry (anchor) | TP1 (anchor) | SL (anchor) | R:R | Katalis | Opened | Triggered | Expires | Source |
|------|--------|--------|------------|----------------|--------------|-------------|-----|---------|--------|-----------|---------|--------|
```

- [ ] **Step 2: Create `journal/closed.md` with empty schema**

Write this exact content to `journal/closed.md`:

```markdown
# Closed Positions — IDX Swing Journal

<!-- Outcome enum: TP1_HIT | SL_HIT | EXPIRED | MANUAL_CLOSE          -->
<!-- Append-only. Return % is computed from entry midpoint.            -->

| Kode | Sektor | Conviction | Entry | TP1 | SL | Opened | Closed | Outcome | Close Price | Return % | Source |
|------|--------|------------|-------|-----|----|----|--------|---------|-------------|----------|--------|
```

- [ ] **Step 3: Verify both files exist and have the header**

Run:
```bash
ls -la journal/
grep -c "^|" journal/positions.md
grep -c "^|" journal/closed.md
```

Expected: both files listed, each grep returns `2` (one header row + one separator row).

- [ ] **Step 4: Commit**

```bash
git add journal/positions.md journal/closed.md
git commit -m "chore: bootstrap swing journal (positions + closed)"
```

---

### Task 2: Rewrite `prompts.md` with GLOBAL RULES + journal-aware routines

**Files:**
- Modify (full rewrite): `prompts.md`

- [ ] **Step 1: Read current `prompts.md`**

```bash
wc -l prompts.md
```

Expected: ~217 lines (current). After rewrite the file will be ~360 lines.

- [ ] **Step 2: Replace `prompts.md` with the new content (full overwrite)**

Write this exact content to `prompts.md`:

````markdown
# IDX Trading Routine Prompts

This file contains all four scheduled prompts for the IDX swing-trading
journal. Each routine references the GLOBAL RULES block below — when
running a routine, paste GLOBAL RULES together with the relevant routine
section into a fresh Claude session at the scheduled time.

================================================

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
  - Max 2 posisi per sektor (Banking, Metals, Energy, OilGas, Consumer,
    CPO, Property, Auto, Healthcare, Tech, Telco, Tower, Power, Cement,
    Conglomerate).
  - Conviction tier per pick:
      FULL = 100% standard unit (high-conviction, setup + katalis kuat)
      HALF = 50% (1 leg lemah, atau confluence belum sempurna)
      TEST = 25% (probe, butuh konfirmasi tambahan)
  - Jika slot full atau sektor cap kena: pick baru harus replace pick
    existing (sebutkan kode yang di-drop + alasan).
  - Mid-day intraday picks TIDAK consume slot — close same day.
  - Position expires after 10 trading days untriggered.

OUTPUT DISCIPLINE:
  - Skip preamble, filler, dan disclaimer.
  - Bahasa Indonesia, bullet rapi, word limit per routine ditegakkan.
  - Setiap routine WAJIB diawali "Position Review" (grade open positions
    dari journal/positions.md) sebelum section baru.

SOURCE PRIORITY:
  IDX (idx.co.id), Bisnis.com, Kontan, CNBC Indonesia, Stockbit, RTI,
  riset Mandiri Sekuritas / Mirae / BRIDS, Investing.com (kalender),
  Bareksa, sahamidx.com, Investortrust.

================================================

A. Daily pre-market — Senin-Jumat, 07:30 WIB

KONTEKS: Pasar IDX buka 09:00 WIB. Sesi 1 09:00-11:30, sesi 2 13:30-15:00.

------ APPLY GLOBAL RULES (lihat blok di atas) ------

LANGKAH 1 — POSITION REVIEW:
Baca journal/positions.md. Untuk setiap row:
  - PLANNED: cek apakah entry zone tersentuh kemarin (intraday range).
    Jika ya, update Status → TRIGGERED dan isi tanggal Triggered.
    Jika hari ini > Expires, pindahkan row ke journal/closed.md dengan
    outcome EXPIRED, Close Price = harga close H-1, Return % dari
    entry midpoint.
  - TRIGGERED: cek apakah TP1 atau SL tersentuh kemarin. Jika ya,
    pindahkan ke closed.md (outcome TP1_HIT atau SL_HIT) + Close Price
    + Return % dari entry midpoint.

LANGKAH 2 — WEB SEARCH untuk data H-1 close, overnight US, Asia pagi,
komoditas, FX, dan event hari ini.

LANGKAH 3 — TULIS BRIEF dengan struktur:

## 1. POSITION REVIEW
Tabel singkat dari positions.md post-grading.
Kolom: Kode | Status | Entry midpoint | Harga close H-1 | P&L % | Days to Expiry | Action (hold/cut/scale/triggered).
Jika ledger kosong: "Tidak ada posisi aktif."

## 2. SNAPSHOT MAKRO
- IHSG kemarin close + %change, foreign net flow
- USD/IDR terkini
- Wall Street semalam (Dow, S&P, Nasdaq)
- Bursa Asia pagi ini (Nikkei, Hang Seng, KOSPI, CSI 300)
- Komoditas: emas, Brent, batubara Newcastle, nikel LME
- Event hari ini (rilis ekonomi, RDG BI, FOMC, dll)
Sumber: [2-4 link]

## 3. NEW PICKS (maksimal 3, LQ45/IDX30 only)
Cek dulu slot dan sektor cap di positions.md:
- Slot tersisa: X dari 5
- Sektor terpakai: [list]
Jika full atau sektor cap kena: pick baru harus REPLACE existing —
sebutkan kode yang di-drop + alasan.

Setiap pick:
- **KODE** (sektor) — Conviction: FULL/HALF/TEST
  - Katalis: [spesifik dengan angka konkret]
  - Entry: [zone] (anchor)
  - TP1: [level] (anchor)
  - TP2 aspirasional: [level] (anchor)
  - SL: [level] (anchor)
  - R:R = (TP1−Entry)/(Entry−SL) = [perhitungan eksplisit, ≥ 2.00]
  - Sumber: [2-3 link]

## 4. AVOID LIST
Saham overheat / distribusi asing / red flag. WAJIB punya level
invalidasi: "KODE avoid sampai konfirmasi reversal di atas X (anchor)".

## 5. ONE-LINER STRATEGI
Bias hari ini: bullish / sideways / bearish + rationale 1 kalimat.

LANGKAH 4 — UPDATE journal/positions.md:
- Append new picks sebagai PLANNED dengan semua kolom terisi.
- Persist grading changes dari Langkah 1.
- Hapus row yang sudah dipindahkan ke closed.md.

LANGKAH 5 — SIMPAN & COMMIT (atomic):
- File brief: briefings/YYYY-MM-DD.md
- Stage: briefings/YYYY-MM-DD.md, journal/positions.md, journal/closed.md
- Commit: `Daily IDX brief - YYYY-MM-DD`

ATURAN OUTPUT:
- Maks 600 kata
- Jika pasar libur: hanya jalankan grade expiry pada Langkah 1, output
  "Pasar libur — tidak ada brief baru hari ini" + commit hanya ledger
  changes (commit message: `Daily IDX - expiry sweep YYYY-MM-DD`).

================================================

B. Mid-day check — Senin-Jumat, 12:30 WIB

KONTEKS: Sesi 1 IDX ditutup 11:30. Sesi 2 buka 13:30 WIB.

------ APPLY GLOBAL RULES (lihat blok di atas) ------

LANGKAH 1 — POSITION REVIEW (intraday update):
Baca journal/positions.md. Untuk setiap row:
  - PLANNED: cek apakah entry zone tersentuh selama sesi 1 (high/low
    sesi 1). Jika ya: Status → TRIGGERED, Triggered = hari ini.
  - TRIGGERED: cek apakah TP1 atau SL tersentuh selama sesi 1. Jika ya,
    pindahkan ke closed.md.

LANGKAH 2 — WEB SEARCH untuk data sesi 1 hari ini.

LANGKAH 3 — TULIS BRIEF:

## 1. POSITION REVIEW (sesi 1)
Hanya row yang berubah status selama sesi 1. Format singkat:
"MDKA: TRIGGERED 11:14 di 2.785, holding."
"INCO: SL hit 10:42 di 3.405 (−2.1% dari entry midpoint)."
Jika tidak ada perubahan: "Tidak ada aktivitas pada open positions sesi 1."

## 2. SESI 1 SNAPSHOT
- IHSG sesi 1 close + %change vs opening
- Volume vs rata-rata 20D (high/normal/low)
- Foreign net flow sesi 1 (jika tersedia)
- Top 3 sektor gainer & laggard sesi 1
- Anomali: saham yang gap up/down >3% dengan volume signifikan
Sumber: [2-4 link]

## 3. LEADERSHIP CHECK
- Apakah saham yang lead sesi 1 sama dengan tema pagi (daily brief hari ini)?
- Jika rotasi sektor terjadi: dari sektor apa → ke sektor apa.
- Implication untuk swing trader: hold / rotate / wait.

## 4. SETUP SESI 2 (LQ45/IDX30, INTRADAY ONLY — TIDAK masuk ledger)
Maksimal 3 saham. Anchor terbatas:
- Opening range high/low sesi 1
- Sesi 1 high / sesi 1 low
- VWAP sesi 1
- Gap fill dari opening

Setiap setup:
- **KODE** — close sesi 1: [harga]
  - Trigger: [break level X / pullback ke Y / volume confirmation]
  - Entry: [zone] (anchor opening-range / sesi 1)
  - TP intraday: [level] (anchor)
  - SL ketat: [level] (anchor)
  - R:R = [perhitungan, minimum 1:1.5 untuk intraday]
  - Validasi: foreign accumulation sesi 1 atau bandar action?

## 5. ALERT & RISK
- News flow penting 09:00-12:00 WIB
- Saham yang masuk UMA / suspend / auto-reject (AR)
- Bursa Asia siang (impact ke sentimen sesi 2)

## 6. ONE-LINER SESI 2
Bias sesi 2: continuation / reversal / sideways + rationale.

LANGKAH 4 — UPDATE journal/positions.md jika ada perubahan dari
Langkah 1. Setup sesi 2 TIDAK ditulis ke ledger.

LANGKAH 5 — SIMPAN & COMMIT (atomic):
- File brief: midday/YYYY-MM-DD.md
- Stage: midday/YYYY-MM-DD.md, journal/positions.md, journal/closed.md
- Commit: `Mid-day IDX brief - YYYY-MM-DD`

ATURAN OUTPUT:
- Maks 400 kata
- Fokus AKSI, bukan analisa panjang
- Jika pasar libur / early close: output "Sesi 1 tidak berjalan normal"
  + alasan, skip Langkah 3-4.

================================================

C. Weekly outlook — Minggu, 19:00 WIB

KONTEKS: Pasar Senin buka 09:00 WIB. Butuh outlook lengkap untuk
5 hari trading ke depan.

------ APPLY GLOBAL RULES (lihat blok di atas) ------

LANGKAH 1 — POSITION REVIEW (full week grading):
Baca journal/positions.md. Untuk setiap row, grade terhadap 5 hari
price action minggu lalu (Senin–Jumat):
  - PLANNED triggered minggu lalu → set TRIGGERED, isi tanggal terdekat.
  - TRIGGERED hit TP1 atau SL → move ke closed.md (outcome + tanggal +
    Close Price + Return % dari entry midpoint).
  - PLANNED dengan hari ini > Expires → closed.md as EXPIRED.

LANGKAH 2 — WEB SEARCH untuk recap minggu lalu + kalender 7 hari
ke depan + earnings calendar LQ45/IDX30.

LANGKAH 3 — TULIS BRIEF:

## 1. POSITION REVIEW (weekly)
Tabel lengkap semua posisi (open + closed minggu lalu):
Kolom: Kode | Status awal | Status akhir | Entry → harga Jumat | P&L % | Days to Expiry / Closed date | Note.
Hit rate minggu lalu: X dari Y picks closed mencapai TP1 (%).
Lessons learned: 1-2 kalimat (pattern yang work / fail).

## 2. RECAP MINGGU LALU
- IHSG WoW change, foreign net flow weekly
- Sektor terbaik & terburuk + magnitude
- Saham LQ45 dengan move ekstrem (>5% atau <−5%)
- Tema dominan minggu lalu
Sumber: [2-4 link]

## 3. KALENDER EVENT MINGGU INI (kritikal)
Tabel: Tanggal | Jam WIB | Event | Dampak.
- Domestik: RDG BI, rilis ekonomi (inflasi, neraca dagang, devisa),
  earnings LQ45/IDX30
- Global: FOMC, ECB, data US (NFP, CPI, PMI), Fed speakers
- Corporate action: RUPS, cum dividen, stock split, IPO besar
- Index: MSCI / FTSE rebalancing

## 4. TEMA SUNRISE MINGGU INI
2-3 narasi paling kuat + beneficiaries di LQ45/IDX30 (1 line each).

## 5. PROPOSED WEEK PICKS (≤5, LQ45/IDX30 only)
Slot reconciliation dulu:
- Slot tersisa setelah Langkah 1 closes: X dari 5
- Sektor terpakai: [list]
- Jika usulan baru melebihi slot tersisa: list mana yang replace
  PLANNED existing + alasan.

Setiap pick:
- **KODE** (sektor) — Conviction: FULL/HALF/TEST
  - Setup teknikal: breakout / pullback / reversal (anchor)
  - Katalis fundamental atau flow asing (angka spesifik)
  - Entry: [zone] (anchor)
  - TP1: [level] (anchor, target 3-5 hari)
  - TP2 aspirasional: [level] (anchor, target 1-2 minggu)
  - SL: [level] (anchor)
  - R:R = [perhitungan, ≥ 2.00]
  - Sumber: [2-3 link]

## 6. RISK MAP
- 3 risiko paling critical minggu ini (geopolitik, makro, technical IHSG)
- Skenario bear case: trigger yang harus bikin cut exposure full.

## 7. ONE-LINER WEEKLY BIAS
risk-on / risk-off / mixed + key catalyst penentu.

LANGKAH 4 — UPDATE journal/positions.md (append new PLANNED rows +
grade updates dari Langkah 1).

LANGKAH 5 — SIMPAN & COMMIT (atomic):
- File brief: weekly/YYYY-MM-DD.md (tanggal Minggu hari ini)
- Stage: weekly/YYYY-MM-DD.md, journal/positions.md, journal/closed.md
- Commit: `Weekly IDX brief - YYYY-MM-DD`

ATURAN OUTPUT:
- Maks 800 kata
- Tabel wajib untuk kalender event & week picks
- Skip teori, fokus actionable

================================================

D. Dividend tracker — Senin, 06:30 WIB

KONTEKS: Investor LQ45/IDX30 ingin maksimasi dividend capture dengan
minimum risk. Track emiten cum-dividen 14 hari ke depan.

------ APPLY GLOBAL RULES (lihat blok di atas) ------

LANGKAH 1 — POSITION REVIEW (expiry sweep saja):
Baca journal/positions.md. Weekend tidak ada price action, TP1/SL
grading sudah diselesaikan oleh weekly brief. Routine ini hanya:
  - Cek setiap PLANNED row apakah hari ini > Expires. Jika ya,
    pindahkan ke closed.md as EXPIRED.

LANGKAH 2 — WEB SEARCH untuk jadwal dividen IDX 14 hari ke depan.

LANGKAH 3 — TULIS BRIEF:

## 1. POSITION REVIEW (cross-reference dividend)
Untuk setiap open position di positions.md, apakah punya cum-date
dalam 14 hari? Jika ya: flag "KODE punya cum date YYYY-MM-DD — adjust
SL untuk ex-date drop."
Jika tidak ada cross-reference: "Tidak ada open position yang
approaching cum/ex date."

## 2. CUM DATE 7 HARI KE DEPAN (LQ45/IDX30 only)
Tabel: Kode | Cum date | Ex date | Payment date | DPS (Rp) | Harga close terkini | Yield (%) | Total dividen (Rp T) | Kategori (final/interim/spesial).
Sumber: [2-4 link]

## 3. CUM DATE 8-14 HARI KE DEPAN
Tabel format sama dengan section 2.

## 4. TOP 3 DIVIDEND CAPTURE OPPORTUNITY
Ranking: yield tinggi + risk dividend trap rendah.
Setiap opportunity:
- **KODE** — yield X%, cum date YYYY-MM-DD
  - Fundamental check: laba growth YoY, payout ratio, FCF coverage
  - Historical pattern: rata-rata hari recovery dari ex-date drop
  - Strategi: entry 1-2 hari pre-cum, expected ex-date drop, target
    hold sampai recovery
  - Risk: news flow negatif yang bisa block recovery
  - Sumber: [2-3 link]

## 5. DIVIDEND TRAP WARNING
Saham yield tinggi tapi RED FLAG:
- Laba turun YoY signifikan (>20%)
- Payout ratio >100% (tidak sustainable)
- Harga sudah pre-rally besar (priced in)
- News negatif menjelang cum date

## 6. REKAP EX-DATE 1 MINGGU TERAKHIR (inline grading)
Tabel: Kode | Ex date | Pre-ex close | Ex-date drop % | Hari recovery ke pre-ex close | Status (recovered / still below / failed).
Pattern insight: 1-2 kalimat.

LANGKAH 4 — UPDATE journal/positions.md jika ada expiry move di
Langkah 1. Dividend captures TIDAK ditulis ke ledger di v1.

LANGKAH 5 — SIMPAN & COMMIT (atomic):
- File brief: dividends/YYYY-MM-DD.md
- Stage: dividends/YYYY-MM-DD.md, journal/positions.md (jika ada changes),
  journal/closed.md (jika ada expiry move)
- Commit: `Dividend tracker - YYYY-MM-DD`

ATURAN OUTPUT:
- Maks 500 kata, prioritas tabel
- Yield dihitung dari harga close Jumat terkini
- Hanya LQ45/IDX30
- Jika tidak ada cum date dalam 14 hari (off-season): output "Tidak
  ada cum date LQ45/IDX30 dalam 14 hari ke depan" + list cum date
  terdekat berikutnya, skip section 4 & 5.
````

- [ ] **Step 3: Verify the rewrite landed correctly**

Run each check separately:

```bash
wc -l prompts.md
grep -c "^GLOBAL RULES" prompts.md
grep -c "^A. Daily pre-market" prompts.md
grep -c "^B. Mid-day check" prompts.md
grep -c "^C. Weekly outlook" prompts.md
grep -c "^D. Dividend tracker" prompts.md
grep -c "^------ APPLY GLOBAL RULES" prompts.md
grep -c "CHART ANCHOR RULE" prompts.md
grep -c "R:R FORMULA" prompts.md
grep -c "PORTFOLIO RULES" prompts.md
grep -c "journal/positions.md" prompts.md
```

Expected:
- `wc -l`: ~340-370 lines (close to 360)
- `GLOBAL RULES`: 1
- Each of `A./B./C./D.`: 1
- `APPLY GLOBAL RULES`: 4 (one per routine)
- `CHART ANCHOR RULE`, `R:R FORMULA`, `PORTFOLIO RULES`: 1 each
- `journal/positions.md`: at least 10 (referenced repeatedly across routines)

If any count is off, re-read the file and fix before committing.

- [ ] **Step 4: Commit**

```bash
git add prompts.md
git commit -m "feat: rewrite prompts with GLOBAL RULES and journal-aware routines"
```

---

### Task 3: Update `CLAUDE.md` to document `journal/` and trading discipline

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add the `journal/` row to the Output Conventions table**

Use Edit on `CLAUDE.md`. Replace the existing table:

```
| `dividends/` | Monday dividend tracker (06:30 WIB) | `Dividend tracker - YYYY-MM-DD` |
```

with two rows:

```
| `dividends/` | Monday dividend tracker (06:30 WIB) | `Dividend tracker - YYYY-MM-DD` |
| `journal/` | Live positions ledger (`positions.md`) + closed archive (`closed.md`) | Updated atomically with the routine that touches it; same commit message as that routine |
```

- [ ] **Step 2: Insert a "Trading Discipline" section after "Trading Context"**

Use Edit on `CLAUDE.md`. After the `## Trading Context` block (ends with the line `- **Language**: Output briefings in Bahasa Indonesia`) and BEFORE the `## Prompts` heading, insert this new section:

```
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
```

- [ ] **Step 3: Verify both edits landed**

Run:
```bash
grep -c "^| \`journal/\`" CLAUDE.md
grep -c "^## Trading Discipline" CLAUDE.md
grep -c "PLANNED or TRIGGERED" CLAUDE.md
grep -c "R:R ≥" CLAUDE.md
wc -l CLAUDE.md
```

Expected:
- `journal/` table row: 1
- `## Trading Discipline` heading: 1
- `PLANNED or TRIGGERED`: 1
- `R:R ≥`: 1
- File length: roughly 52-58 lines (was 40)

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: document journal/ and trading discipline in CLAUDE.md"
```

---

## Done state

After all three tasks:

- `journal/positions.md` and `journal/closed.md` exist with empty schemas (bootstrap commit).
- `prompts.md` is fully rewritten — GLOBAL RULES at top, four routines below, every routine reads/writes the ledger and includes a Position Review section.
- `CLAUDE.md` documents the new `journal/` directory and the trading-discipline rules.

The next scheduled routine (whichever fires first — daily, mid-day, weekly, or dividend) will be the first run under the new discipline. It will see an empty ledger, run `LANGKAH 1` as a no-op, generate a brief, propose new PLANNED rows, and write them to `journal/positions.md` — kicking off the live journal.

## Open work for v2 (deferred per spec)

- Separate `journal/dividends.md` ledger for dividend captures.
- Analytics script to compute hit-rate / Sharpe from `closed.md`.
- Formalize sector taxonomy against IDX sector codes if disagreements arise.
