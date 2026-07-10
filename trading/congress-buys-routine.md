# Congress Buys — Paper Trading Routine (Liquid.trade)

**Owner:** mike.browning@gmail.com
**Account:** Liquid.trade **PAPER** account (simulated funds only — never live)
**Created:** 2026-07-10
**Status:** Active

This file is the single source of truth for the scheduled routine that mirrors
congressional stock **buys** into the Liquid.trade paper account. The scheduled
trigger runs the "Execution prompt" at the bottom of this file.

---

## 1. What the routine does

Every 30 minutes **during US equity market hours**, it:

1. Confirms the Liquid account is in **paper mode**.
2. Pulls the latest **Congress Buys** trades from the QuiverQuant "Congress
   Buys" strategy, **prioritizing** trader **Tim Moore** (R-NC, House;
   QuiverQuant/CapitolTrades ID `M001236`) but not limited to him. Moore's own
   picks skew to small/mid-cap names that are often **not listed on Liquid**, so
   monitoring the broader Congress Buys strategy is what surfaces the large-cap
   buys that are actually executable on the paper account. Tim Moore signals are
   always flagged first when present.
3. **Checks each candidate buy is tradeable on Liquid** (`search_markets`).
   Congressional buys in tickers not listed on Liquid are logged and surfaced to
   the user as "signal — not tradeable on Liquid", but no order is proposed.
4. Compares any newly disclosed **tradeable** trades against:
   - the **recommendation log** (`recommendations-log.md`) — what we've already
     seen/acted on, and
   - the **current paper portfolio** (open positions).
4. If there is a **new BUY** (purchases only — sales are ignored) that we have
   not already logged or already hold, it:
   - calculates a position size from the current balance and the configured
     risk level,
   - builds a full recommendation summary (the 6 points below),
   - **posts the summary into the chat session and asks for approval**, and
   - appends the recommendation to the log with status `PENDING APPROVAL`.
5. It **does NOT place the trade automatically.** A trade is only placed after
   the user explicitly approves it in chat, at which point the log entry is
   updated to `PLACED` (or `SKIPPED` if declined).
6. Every run appends a one-line heartbeat to the log, even when there is no new
   signal, so there is a complete audit trail.

---

## 2. Parameters

| Parameter | Value |
|---|---|
| Strategy | QuiverQuant **Congress Buys** |
| Focus trader | **Tim Moore** (House R-NC), ID `M001236` |
| Cadence | Every 30 minutes |
| Trading window | US market hours only: **Mon–Fri, 09:30–16:00 America/New_York** |
| Account | Liquid.trade **paper** (simulated) |
| Risk level | **Aggressive** |
| Sizing rule | **~10% of current equity per new position**, rounded to a clean notional |
| Leverage | **1x** (no leverage) by default for equity mirrors |
| Per-position cap | Never size a single new position above **12%** of equity |
| Signal types acted on | **BUY / purchase only** (sales are logged but not traded) |
| Auto-place? | **No.** Human approval required in chat before any order. |

> Risk level was chosen as **Aggressive** by the account owner. To change it,
> edit this table and the sizing math in the execution prompt.

---

## 3. Data sources (in priority order)

The primary QuiverQuant table is JavaScript-rendered and not readable via a
plain fetch. Use these server-rendered sources instead:

1. **StockTaper** (primary, server-rendered, parseable):
   `https://www.stocktaper.com/congress/timmoore`
2. **QuiverQuant news feed** (for fresh disclosure alerts):
   `https://www.quiverquant.com/news/` and per-politician page
   `https://www.quiverquant.com/congresstrading/politician/Tim%20Moore-M001236`
3. **CapitolTrades** (fallback): `https://www.capitoltrades.com/politicians/M001236`
4. **MarketBeat / AltIndex / Unusual Whales** congressional trackers as
   secondary confirmation.

Performance-vs-S&P data for the summary can come from the same trackers
(QuiverQuant reports an "Excess Return" column and yearly performance).

---

## 4. "New buy" detection logic

A disclosed trade is a **new actionable buy** when ALL of these hold:

- Transaction type is a **purchase/buy** (ignore `Sale`/`Sell`).
- The `(ticker, transaction_date, amount_range)` tuple does **not** already
  appear in `recommendations-log.md`.
- We do **not** already hold an open position in that ticker in the paper
  portfolio.
- The disclosure is recent (filed within the last ~14 days) — older backfilled
  rows are recorded to the log's "known trades" baseline but not re-recommended.
- **The ticker is tradeable on Liquid** (`search_markets` returns an exact
  match). If it is a valid new buy but NOT on Liquid, log it and post a short
  "signal — not tradeable on Liquid" note to chat, but do not size or propose an
  order.

On the **first run**, seed the baseline from the current StockTaper table so we
don't fire recommendations for months-old trades; only genuinely new filings
after the baseline should trigger recommendations. (The baseline as of
2026-07-10 is recorded at the bottom of `recommendations-log.md`.)

---

## 5. Position sizing (Aggressive)

```
equity              = paper account equity (from get_portfolio)
target_notional     = round_to_clean( 0.10 * equity )      # ~10%
cap                 = 0.12 * equity                          # hard cap
notional            = min(target_notional, cap)
price               = current Liquid price for the ticker (analyze_market)
shares (approx)     = notional / price
leverage            = 1x
```

Use a **market buy** unless the ticker is illiquid/gapping, in which case use a
limit at/near the ask. Report the exact order in the summary before asking for
approval.

---

## 6. The recommendation summary (always these 6 points)

When a new buy is found, post this to chat and log it:

1. **Ticker** — symbol + company name.
2. **Insider / strategy signal** — "Tim Moore (Congress Buys) purchased $X–$Y on
   <date>, disclosed <filing date>."
3. **Why the trade matters** — sector/thesis, size relative to his other trades,
   any clustering (repeat buys), conviction read.
4. **Performance vs S&P 500** — how this trader / the Congress Buys strategy has
   done against SPX (e.g. 2025: Moore +52% vs S&P +16.6%), with a source.
5. **Suggested position size** — notional $ (~10% of equity) and approx shares,
   1x, and % of equity.
6. **Exact order** — e.g. `PAPER BUY 23 T @ market, 1x, ~$1,000 notional` plus
   any TP/SL if used.

Then explicitly ask: **"Approve this paper trade? (approve / skip)"** and stop.
Do not place the order until the user approves.

---

## 7. On approval

- On **approve**: call the Liquid order tool to place the **paper** market buy
  with the exact size, then update the log entry to `PLACED` with the fill.
- On **skip/decline**: update the log entry to `SKIPPED` with the reason.
- Either way, add the trade's tuple to the baseline so it isn't re-recommended.

---

## 8. Guardrails

- **Never** disable paper trading. Verify `paper_trading_status` is enabled at
  the start of every run; if it isn't, enable it before doing anything else.
- **Never** auto-place a trade without explicit chat approval.
- Only act **during market hours** (the schedule enforces this, but re-check the
  clock — skip if it's a US market holiday).
- Ignore **sales**; this routine only mirrors buys.
- Keep the log append-only; never rewrite history, only update status fields.

---

## 9. Scheduling (durable Routines)

The 30-minute cadence is implemented as **two offset hourly Routines** (the
scheduler's minimum interval is hourly), both firing into the owner's chat
session so the approval step lands in chat. Cron is in **UTC**; the window is
widened for DST and the routine self-gates on the real America/New_York clock.

| Routine | ID | Cron (UTC) | Fires |
|---|---|---|---|
| Congress Buys (paper) — top of hour | `trig_011BhrwJp97sELuPGnYS5GVC` | `0 13-21 * * 1-5` | :00 each hour |
| Congress Buys (paper) — half past | `trig_01K4tvawFp6wJQZsZ7846v7R` | `30 12-20 * * 1-5` | :30 each hour |

Together they fire every 30 minutes across US market hours, Mon–Fri. Firings
outside the real 09:30–16:00 ET window are skipped by the routine's own
market-hours check (logged as heartbeats). To pause the routine, disable/delete
both triggers.

## 10. Execution prompt (what the scheduled trigger runs)

> This is the exact instruction fired into the chat session every 30 minutes
> during market hours. It is intentionally self-contained.

```
Run the Congress Buys paper-trading routine defined in
trading/congress-buys-routine.md.

Steps:
1. Confirm the current time is within US market hours (Mon–Fri 09:30–16:00
   America/New_York) and it is not a market holiday. If not, append a
   "skipped — outside market hours" heartbeat to
   trading/recommendations-log.md and stop.
2. Verify Liquid paper trading is ENABLED (enable it if not).
3. Fetch the latest Congress Buys trades, prioritizing Tim Moore
   (https://www.stocktaper.com/congress/timmoore) and scanning the broader
   Congress Buys feed (fallbacks in the routine file). Parse ticker / buy-sell /
   amount / dates.
4. Load the baseline + prior entries from trading/recommendations-log.md and
   the current paper portfolio (get_portfolio). Determine if there is a NEW
   BUY per the detection logic (purchases only, not already logged, not already
   held, filed within ~14 days). For each new buy, check tradeability on Liquid
   with search_markets.
5. If NO new buy: append a one-line heartbeat to the log and stop (no chat spam).
   If there IS a new buy but the ticker is NOT on Liquid: log it and post a brief
   "signal — not tradeable on Liquid" note, then stop.
6. If there IS a new buy that IS tradeable on Liquid: get its current Liquid
   price (analyze_market), compute the Aggressive size (~10% of equity, 1x, 12%
   cap), and post the 6-point recommendation summary to chat, append it to the
   log as PENDING APPROVAL, then ask "Approve this paper trade? (approve / skip)"
   and STOP. Do not place the order.
7. When I later approve, place the PAPER market buy at the stated size and
   update the log entry to PLACED (or SKIPPED if I decline).
```
