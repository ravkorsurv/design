# Scila Rule Engine Reference (Spoofing & Insider)

## Purpose
This document captures the deterministic rule logic the Scila-style comparator uses in the Pattern-1 MVP. It enumerates the required inputs, equity market-cap segmentation thresholds, alert triggers, and implementation notes so developers can implement or mock the comparator reliably for scenario benchmarking.

## Shared Data Inputs
| Feed | Fields | Notes |
| --- | --- | --- |
| **Orders** | order_id, account_id, instrument_id, side, price, quantity, event_type (NEW/MOD/CXL), timestamp | Must preserve parent-child linkage for modifications so outstanding depth can be reconstructed. |
| **Trades** | trade_id, order_id, account_id, instrument_id, side, price, quantity, timestamp | Used for fill ratios and detecting wash trading. |
| **Market Data (Level 1)** | instrument_id, best_bid, best_ask, mid_price, bid_depth, ask_depth, timestamp | Needed for away-from-mid checks and price gap validation. |
| **Instrument Reference** | instrument_id, liquidity_band, market_cap_usd, sector, venue | Market-cap segmentation drives dynamic thresholds. |
| **Account Metadata** | account_id, client_type (retail/institutional/prop), desk_id | Enables rule exclusions or stricter limits for certain client segments. |
| **Baselines (optional)** | cancel_ratio_baseline, trade_to_order_baseline per instrument or segment | Used when available to override static thresholds. |

## Equity Market-Cap Thresholds
Thresholds map instruments into three liquidity sensitivity tiers. Use the reference data's `market_cap_usd` snapshot at the session open.

| Segment | Market Cap (USD) | Typical Instruments | Notes |
| --- | --- | --- | --- |
| **Large Cap** | ≥ 10 B | S&P 500 constituents, Tier-1 listings | High liquidity → tolerate slightly higher cancel ratios before alerting. |
| **Mid Cap** | 2 B – < 10 B | Mid-tier exchange listings | Baseline thresholds remain unchanged. |
| **Small / Illiquid** | < 2 B or liquidity_band ∈ {“AIM”, “OTC”} | AIM, OTCBB, smaller venues | Apply stricter notional and distance-from-mid triggers due to lower depth. |

When an instrument is a bond or future, skip market-cap checks and rely on configured liquidity bands.

---

## Spoofing Model — Scila-Style (Binary, Event-Driven)
The comparator evaluates the following predicates on rolling windows. Each predicate is independent; multiple alerts may fire for the same behaviour.

### Data Inputs
- Orders (new/modify/cancel)
- Trades (fills)
- Level 1 market data (best bid/ask, mid)
- Instrument reference data (liquidity band, market cap)
- Account metadata (client type)
- Optional baselines for trade-to-order and cancel ratios

### Rule Catalogue

#### 1. High Cancel Ratio (`HighCancelRatio`)
- **Window:** Rolling 60 seconds per account × instrument × venue.
- **Trigger:**
  - Cancel ratio ≥ 80 % (`cancel_events / order_events`), *and*
  - Total order events (NEW + MOD + CXL) ≥ 10.
- **Segment adjustments:**
  - Large cap: threshold remains 80 %.
  - Mid cap: threshold 75 %.
  - Small/illiquid: threshold 65 %.
- **Output payload:** include cancel ratio, total orders, segment tag.

#### 2. Order Churn (`OrderChurn`)
- **Window:** Rolling 10 seconds per account × instrument.
- **Trigger:**
  - Order submissions ≥ 5 per second averaged over the window.
- **Segment adjustments:**
  - For small/illiquid instruments, reduce threshold to 3 per second.
- **Notes:** modifications count as submissions; cancels do not.

#### 3. Layering / Large Queue Orders (`Layering`)
- **Evaluation:** maintain live order book snapshot per account.
- **Trigger:**
  - Outstanding orders at ≥ 3 distinct price levels on the same side of the book, *and*
  - Aggregate notional ≥ threshold by segment:
    - Large cap: ≥ £1 000 000
    - Mid cap: ≥ £500 000
    - Small/illiquid: ≥ £250 000
- **Reset:** clear once order depth falls below 3 levels or notional drops below threshold.

#### 4. Large Orders Away From Mid (`AwayFromMidCancel`)
- **Trigger:**
  - New order posts at distance from mid-price ≥ δ, where δ defaults to 0.5 % for large/mid caps and 0.25 % for small/illiquid instruments.
  - Order cancelled (no fill) within 5 seconds of placement.
- **Payload:** record distance from mid, lifetime, notional.

#### 5. Wash Trade Check (`WashTradePattern`)
- **Trigger:**
  - Trade event where buyer and seller account IDs match (self-match) or share the same beneficial owner tag.
  - Optional enhancement: limit detection to trades within ±2 ticks of mid.
- **Payload:** include both order IDs, trade price, and venue.

#### 6. Trade-to-Order Ratio (`LowTradeToOrderRatio`)
- **Window:** Rolling 5 minutes per account × instrument.
- **Trigger:**
  - `executed_orders / total_orders ≤ 5 %` (or comparator baseline if supplied).
- **Segment adjustments:**
  - Large cap: keep at 5 %.
  - Mid cap: 4 %.
  - Small/illiquid: 3 %.
- **Notes:** resets when volume-weighted execution ratio recovers above threshold for two consecutive windows.

### Implementation Notes
- Compute rolling metrics using tumbling windows aligned to UTC timestamps to mimic the deterministic Scila implementation.
- Each alert should include `rule_name`, `trigger_timestamp`, `account_id`, `instrument_id`, `segment`, and relevant metrics.
- No correlation is performed between rules; duplicates are expected for single behaviours (e.g., layering + high cancel ratio).

---

## Insider Dealing Model — Scila-Style (Binary, Event Proximity)
The insider comparator keys off corporate event calendars and trade activity. It does not perform probabilistic fusion.

### Data Inputs
- Executed trades (price, quantity, direction)
- Corporate event timestamps (earnings, M&A, ratings changes, guidance, regulatory news)
- Instrument reference data (sector, liquidity, market cap)
- Optional account metadata (client type, desk)
- Optional holdings snapshot (positions before/after event)

### Rule Catalogue

#### 1. Pre-Event Trading (`PreEventTrade`)
- **Window:** trades within configurable pre-event horizon (default 120 minutes for equities, 240 minutes for bonds).
- **Trigger:**
  - Trade direction matches post-event price move sign (buy before positive move or sell before negative move).
  - Trade notional ≥ min threshold (see rule 2).
- **Segment adjustments:**
  - Equity large cap: 60–120 minute window.
  - Equity small cap / illiquid: extend to 240 minutes due to thinner news flow.
  - Bonds: use credit event calendar; 240 minute default.

#### 2. Large Position Relative to Normal (`LargeTradeBeforeEvent`)
- **Trigger:** trade notional exceeds the greater of:
  - Fixed floor (e.g., £500 000 large cap, £250 000 mid cap, £100 000 small cap), *or*
  - `X ×` trailing 30-day average trade size for the account on that instrument (default X = 3).
- **Notes:** This predicate is evaluated for all trades, not just pre-event ones; it feeds rule 1 as a helper.

#### 3. High Turnover Account (`HighTurnoverEventProximity`)
- **Window:** per account daily volume tally.
- **Trigger:**
  - Account executes > N trades per day (default N = 50 for retail, 200 for institutional), *and*
  - At least one trade occurs inside the pre-event window.

#### 4. Concentration (`HighParticipation`)
- **Window:** event day session.
- **Trigger:**
  - Account participation ≥ 10 % of total on-book volume for the instrument on the event day.
  - For small-cap equities or illiquid bonds, lower threshold to 5 %.

#### 5. News Reaction Check (`PriceMoveAfterTrade`)
- **Trigger:**
  - Trade occurs before an absolute price gap ≥ X % (default X = 2 % for large cap equities, 1 % for small cap/bonds) measured within 30 minutes after news release.
  - No requirement for direction confirmation beyond matching sign; comparator simply checks magnitude.

### Implementation Notes
- Each alert carries `event_id`, `event_type`, `trade_timestamp`, `direction`, `notional`, and `price_gap` metrics.
- When multiple rules trigger for the same account/instrument/event, the comparator emits separate alerts; it does not deduplicate or link them.
- Ground truth labels for Pattern-1 scenarios (S1–S8) should map to these rule outputs to compute comparison metrics like Precision@k.

---

## Output Schema Summary
```json
{
  "alert_id": "SCILA-<uuid>",
  "rule_name": "HighCancelRatio",
  "account_id": "<acct>",
  "instrument_id": "<isin or symbol>",
  "segment": "LARGE_CAP",
  "trigger_timestamp": "2024-06-20T09:31:12.345Z",
  "metrics": {
    "cancel_ratio": 0.82,
    "order_count": 19,
    "distance_from_mid": 0.006
  },
  "scenario_id": "S1",
  "ground_truth": true
}
```
Use this structure (or an equivalent protobuf) so downstream benchmarking services and dashboard exports can join Scila alerts with Korinsic fused cases by `scenario_id`.
