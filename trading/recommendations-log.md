# Congress Buys — Recommendation Log

Append-only audit trail for the Congress Buys paper-trading routine
(see `congress-buys-routine.md`). Every scheduled run writes here: a full entry
for each recommendation, and a one-line heartbeat when there is no new signal.
**Paper account only — simulated funds.**

Status values: `PENDING APPROVAL` · `PLACED` · `SKIPPED` · `NOT TRADEABLE` · `HEARTBEAT` · `SEEN` · `DATA-ERROR`

---

## Data source change — 2026-07-13

The routine now pulls **all members of Congress** from the QuiverQuant bulk API
(`https://api.quiverquant.com/beta/bulk/congresstrading`, auth via
`QUIVER_API_KEY`) instead of the Tim-Moore-only StockTaper feed. The
Tim-Moore-only baseline below is **superseded**. On the first successful
bulk-API fetch the routine will **re-seed a new baseline** (record current
recent purchases as `SEEN`, no recommendations that run); only buys disclosed
after that re-seed are actionable.

### Bulk-API baseline (re-seed pending)

_Not yet seeded — awaiting the first authenticated bulk-API run once
`QUIVER_API_KEY` is set in the environment._

---

## Baseline (Tim-Moore-only, superseded 2026-07-13)

These pre-existing disclosures are recorded so the routine does NOT re-recommend
months-old trades. Only NEW filings after this baseline should trigger a
recommendation. Tradeability checked against Liquid on 2026-07-10.

| Ticker | Type | Amount | Traded | Filed | On Liquid? |
|---|---|---|---|---|---|
| DASH | Sale | $1,001–$15,000 | 2026-06-09 | 2026-07-10 | n/a (sale, ignored) |
| T (AT&T) | Purchase | $15,001–$50,000 | — | 2026-05-18 | ❌ not listed |
| IHG | Purchase | $1,001–$15,000 | — | 2026-05-07 | ❌ not listed |
| CBRL | Purchase | $15,001–$50,000 | 2026-03-23 | — | ❌ not listed |
| LGIH | Purchase | $15,001–$100,000 | 2026-03-18…20 | — | ❌ not listed |
| HOG | Purchase | $15,001–$50,000 | 2026-03-12 | — | ❌ not listed |
| DNUT | Purchase | $1,001–$15,000 | 2026-02-12 | — | ❌ not listed |
| GNPX | Purchase | $1,001–$15,000 | 2026-02-05 | — | ❌ not listed |

> Note: Tim Moore's recent picks are small/mid-cap names not offered on Liquid,
> so they are recorded but not actionable on the paper account. The routine also
> watches the broader Congress Buys strategy to catch large-cap buys that ARE
> tradeable on Liquid.

---

## Log entries (newest first)

- `2026-07-13 09:37 ET` — `HEARTBEAT` — market hours, checked. Paper equity $10,000, 0 positions. Tim Moore latest = DASH sale 2026-06-09 (already in baseline; sale, ignored). No new buy filed within ~14 days. No actionable signal.

- `2026-07-13 09:05 ET` — `HEARTBEAT` — skipped, pre-market (before 09:30 ET open). No check performed.

- `2026-07-13 08:35 ET` — `HEARTBEAT` — skipped, pre-market (before 09:30 ET open). No check performed.

- `2026-07-12 16:39 ET` — `HEARTBEAT` — skipped, outside market hours (Sunday / weekend). No check performed.

### 2026-07-10 — First run / setup (manual dry run)

- **Account:** Liquid paper, equity **$10,000.00**, 0 open positions.
- **Congress Buys / Tim Moore latest filing:** SALE of **DASH** (filed
  2026-07-10) → ignored (routine mirrors buys only).
- **Most recent Moore BUYS:** T, IHG, CBRL, LGIH, HOG — **none tradeable on
  Liquid** → recorded to baseline, no order proposed.
- **Result:** **No actionable paper trade today.** No new, tradeable Congress
  buy since baseline.
- **Status:** `HEARTBEAT` (setup)
