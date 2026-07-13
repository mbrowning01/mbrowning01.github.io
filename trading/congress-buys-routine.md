# Congress Buys — Paper Trading Routine (Liquid.trade)

**Owner:** mike.browning@gmail.com
**Account:** Liquid.trade **PAPER** account (simulated funds only — never live)
**Created:** 2026-07-10 · **Updated:** 2026-07-13 (switched to QuiverQuant bulk API; removed Tim Moore restriction)
**Status:** Active

This file is the single source of truth for the scheduled routine that mirrors
congressional stock **buys** into the Liquid.trade paper account. The scheduled
trigger runs the "Execution prompt" at the bottom of this file.

---

## 1. What the routine does

Every 30 minutes **during US equity market hours**, it:

1. Confirms the Liquid account is in **paper mode**.
2. Pulls the latest **Congress Buys** across **all members of Congress** from the
   QuiverQuant bulk API (see §3). No single trader is prioritized.
3. **Checks each candidate buy is tradeable on Liquid** (`search_markets`).
   Buys in tickers not listed on Liquid are logged and surfaced as
   "signal — not tradeable on Liquid", but no order is proposed.
4. Compares any newly disclosed **tradeable buys** against:
   - the **recommendation log** (`recommendations-log.md`) — what we've already
     seen/acted on, and
   - the **current paper portfolio** (open positions).
5. If there is a **new BUY** (purchases only — sales are ignored) that we have
   not already logged or already hold, it:
   - calculates a position size from the current balance and the configured
     risk level,
   - builds a full recommendation summary (the 6 points below),
   - **posts the summary into the chat session and asks for approval**, and
   - appends the recommendation to the log with status `PENDING APPROVAL`.
6. It **does NOT place the trade automatically.** A trade is only placed after
   the user explicitly approves it in chat, at which point the log entry is
   updated to `PLACED` (or `SKIPPED` if declined).

---

## 2. Parameters

| Parameter | Value |
|---|---|
| Strategy | QuiverQuant **Congress Buys**, **all members** (no trader filter) |
| Data source | QuiverQuant bulk API (§3) |
| Cadence | Every 30 minutes |
| Trading window | US market hours only: **Mon–Fri, 09:30–16:00 America/New_York** |
| Account | Liquid.trade **paper** (simulated) |
| Risk level | **Aggressive** |
| Sizing rule | **~10% of current equity per new position**, rounded to a clean notional |
| Leverage | **1x** (no leverage) by default for equity mirrors |
| Per-position cap | Never size a single new position above **12%** of equity |
| Signal types acted on | **BUY / purchase only** (sales are logged but not traded) |
| Recency window | Only act on buys **disclosed (ReportDate) within the last ~14 days** |
| Auto-place? | **No.** Human approval required in chat before any order. |
| Per-run limit | Propose **at most one** recommendation per run (most recent tradeable new buy); others are logged for later runs |

---

## 3. Data source — QuiverQuant bulk API

**Endpoint:** `https://api.quiverquant.com/beta/bulk/congresstrading`

**Auth:** Bearer/Token header using the secret in the **`QUIVER_API_KEY`**
environment variable (configured in the Claude Code environment settings — never
committed to the repo). Use:

```
curl -sS -H "Authorization: Token ${QUIVER_API_KEY}" \
  https://api.quiverquant.com/beta/bulk/congresstrading
```

If the request returns 401/403, or `QUIVER_API_KEY` is unset, log a
`DATA-ERROR` heartbeat and stop (do not fabricate signals). If `Token` auth is
rejected, retry once with `Authorization: Bearer ${QUIVER_API_KEY}`.

**Response:** a JSON array of trade objects. Expected fields (names may vary
slightly; match case-insensitively and tolerate extras):

- `Representative` / `Name` — member of Congress
- `Ticker` — stock symbol
- `Transaction` — `Purchase` (buy) or `Sale`/`Sell`
- `TransactionDate` — date of the trade
- `ReportDate` / `Filed` — disclosure date
- `Range` / `Amount` — dollar range
- `House` / `BioGuideID` — chamber / member id

Parse the array, keep only `Transaction == Purchase`, and use the actual field
names returned by this endpoint: **`Ticker`, `Traded`** (transaction date),
**`Filed`** (disclosure date), **`Transaction`, `Trade_Size_USD`, `Name`**
(member), `Party`, `Chamber`, `excess_return`. Use **`Filed`** (fall back to
`Traded`) for recency and the dedup key `Ticker|Traded|Name`. Note the feed is
large (~53 MB / 110k+ rows); filter to recent purchases before doing per-ticker
work.

---

## 4. "New buy" detection logic

The authoritative dedup store is **`trading/seen-congress-buys.json`** — a set of
`"Ticker|Traded|Name"` keys already seen (plus a `seeded_at` date). Read it at
the start of each run and add every purchase you process to it.

A disclosed trade is a **new actionable buy** when ALL of these hold:

- Transaction type is a **purchase/buy** (ignore `Sale`/`Sell`).
- The `Ticker|Traded|Name` key is **not** in `trading/seen-congress-buys.json`
  (nor already recommended in `recommendations-log.md`).
- We do **not** already hold an open position in that ticker in the paper
  portfolio.
- `Filed` (disclosure date) is within the last **~14 days**.
- **The ticker is tradeable on Liquid** (`search_markets` returns an exact
  match). If it is a valid new buy but NOT on Liquid, log it as `NOT TRADEABLE`
  and post a brief note, but do not size or propose an order.

**Baseline re-seed (first authenticated run after the 2026-07-13 source switch):**
The old baseline in `recommendations-log.md` came from the Tim-Moore-only
StockTaper feed and does NOT reflect the full-Congress bulk API. On the FIRST
successful bulk-API fetch, **seed a new baseline**: record all purchases from the
current fetch (last ~14 days) into the log's baseline as `SEEN` and do NOT fire
recommendations that run. Only buys that appear in *subsequent* fetches (i.e.
newly disclosed after the re-seed) are actionable. This prevents a flood of
recommendations for the existing backlog.

If multiple new tradeable buys appear in one run, propose only the **single most
recent** (by ReportDate) and record the rest in the log so later runs can pick
them up one at a time.

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
2. **Insider / strategy signal** — "<Representative> (Congress Buys) purchased
   $X–$Y on <transaction date>, disclosed <report date>."
3. **Why the trade matters** — sector/thesis, size, any clustering (multiple
   members or repeat buys in the same name), conviction read.
4. **Performance vs S&P 500** — how the Congress Buys strategy / that member has
   done against SPX, with a source.
5. **Suggested position size** — notional $ (~10% of equity) and approx shares,
   1x, and % of equity.
6. **Exact order** — e.g. `PAPER BUY 6 NVDA @ market, 1x, ~$1,000 notional`
   plus any TP/SL if used.

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
- Never print or commit `QUIVER_API_KEY`.
- Keep the log append-only; never rewrite history, only update status fields.

---

## 9. Scheduling (durable Routines)

The 30-minute cadence is implemented as **two offset hourly Routines** (the
scheduler's minimum interval is hourly), both firing into the owner's chat
session so the approval step lands in chat. Cron is in **UTC**; the window is
widened for DST and the routine self-gates on the real America/New_York clock.

| Routine | ID | Cron (UTC) | Fires |
|---|---|---|---|
| Congress Buys (paper) — top of hour | `trig_014xBFrK3oijua5EAMdUcPdL` | `0 13-21 * * 1-5` | :00 each hour |
| Congress Buys (paper) — half past | `trig_01GW4gi2zw4UgZLe2t3Xs8zG` | `30 12-20 * * 1-5` | :30 each hour |

Together they fire every 30 minutes across US market hours, Mon–Fri. Firings
outside the real 09:30–16:00 ET window are skipped by the routine's own
market-hours check. To pause the routine, disable/delete both triggers.

## 10. Execution prompt (what the scheduled trigger runs)

```
Run the Congress Buys paper-trading routine defined in
trading/congress-buys-routine.md.

1. Confirm it is currently US market hours (Mon–Fri 09:30–16:00
   America/New_York) and not a holiday. If not, stop quietly (no chat message).
2. Verify Liquid paper trading is ENABLED (enable if not).
3. Fetch the latest Congress buys for ALL members from the QuiverQuant bulk API
   at https://api.quiverquant.com/beta/bulk/congresstrading using the
   QUIVER_API_KEY env var (Authorization: Token). Keep only purchases. If the
   API errors or the key is missing, log a DATA-ERROR note and stop.
4. Load the baseline + prior entries from trading/recommendations-log.md and the
   current paper portfolio (get_portfolio). If no bulk-API baseline exists yet,
   RE-SEED: record current recent purchases as the baseline and stop without
   recommending. Otherwise find NEW buys (purchase, not already logged/held,
   ReportDate within ~14 days) and check tradeability on Liquid (search_markets).
5. If nothing new/tradeable: stop quietly. If a new buy is NOT on Liquid: log it
   NOT TRADEABLE and post a brief note, then stop.
6. If there is a new tradeable buy: pick the single most recent, get its price
   (analyze_market), compute the Aggressive size (~10% of equity, 1x, 12% cap),
   post the 6-point summary, append it to the log as PENDING APPROVAL, ask
   "Approve this paper trade? (approve / skip)", and STOP. Do not place the order.
7. When a recommendation, placement, or status change occurs, commit and push
   trading/recommendations-log.md to branch
   claude/liquid-paper-trading-routine-ml2ic5.
```
