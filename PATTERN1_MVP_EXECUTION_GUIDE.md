# Pattern-1 MVP Execution Guide

## Purpose and Scope
The Pattern-1 minimum viable product (MVP) hardens the current pre-MVP codebase so it can ingest real-time US equity market data, blend it with mocked adjunct data sources, and drive an eight-scenario storyboard (the original six core flows plus two cross-venue/cross-product extensions) end-to-end through the existing analysis stack. This guide summarises what the MVP delivers, why it matters, and the concrete development tasks required to tailor the baseline system to the client brief.

The existing implementation is still a pre-MVP baseline; every workstream below focuses on maturing that code into a client-ready demonstrator without assuming production fitness today.

## Business Outcomes
- **Live market credibility.** Show stakeholders that Pattern-1 processes genuine exchange data (AAPL.O, TSLA.O) with the same controls planned for production while preserving deterministic mocked fixtures for other feeds.
- **Analyst-ready alerts.** Deliver HIGH/CRITICAL real-time alerts enriched with motif, anomaly, and graph evidence, accompanied by human-readable narratives, Evidence Sufficiency Index (ESI) banners, and recommended actions.
- **Comparative positioning.** Quantify how Pattern-1’s Bayesian fusion suppresses noise versus rule-based systems (e.g., Scila), highlighting the value of CPT-driven enrichment.
- **Operational readiness.** Provide a documented runbook covering provider credentials, ingestion toggles (live/replay/blended), troubleshooting, and nightly capture routines.

## Functional Capabilities
1. **Market Data Ingestion**
   - Primary live feed streaming US equities over websocket/SSE.
   - Synthetic fixtures for non-market channels (orders, HR, comms, PnL) to keep scenario playback deterministic.
   - Micro-batching (≤250 ms) with back-pressure, exponential backoff, and circuit breaker safeguards.
2. **Data Shaping & Feature Alignment**
   - Reuse `DataProcessor.process_realtime` for latency-critical metrics until Pattern-1 rolling windows are warm.
   - Maintain parity between live ticks and synthetic fixtures.
3. **Layered Evidence Pipeline**
   - **Layer A:** YAML-defined fast-path rules (Spoof_Burst, Quote_Stuffing, PreNews_Accum) wrapping `AnalysisService.analyze_realtime_data` and emitting via `AlertService.generate_realtime_alerts`.
   - **Layer B:** Anomaly evidence with z-scores and feature contribution rankings.
   - **Layer C1:** Sequence motifs (2–10 s windows) with rarity scores and excerpt snippets.
   - **Layer C2:** Graph coordination signals refreshing every 5 s with community IDs and collusion scores.
   - **Layer D:** Bayesian fusion using production-targeted CPTs and `ComplexRiskAggregator` for tiering, penalties, and DQ modifiers.
   - **Evidence Sufficiency:** Apply fallback evidence paths, compute ESI, and surface DQSI dampeners in payloads.
4. **Scenario Replay & Blending**
   - Toggle between live-only, replay-only, and blended feeds per scenario.
   - Record nightly 15–30 minute live segments for regression playback.
5. **UI & Explainability**
   - Persist enriched alerts with scenario IDs, narratives, timelines, recommended actions, motif excerpts, anomaly summaries, graph enrichment, naive vs fused comparisons, and workflow states (e.g., Analyst Review → Escalated to Manager → Closed).
   - Expose alert-level tables/cards in the Pattern-1 UI so analysts can inspect each fused case alongside the dashboard aggregates, update status, capture notes, and prepare exports for stakeholder briefings.
   - Attach deterministic natural-language rationales and synthetic PnL summaries to every alert so the analyst-facing payload already contains the content required for presentation decks or external dashboards.
6. **Observability & Runbook**
   - Metrics for connection health, throttling, per-instrument event rates, and alert latencies.
   - Document API key handling, reconnect policy, demo-safe usage, and CLI commands.

## Demo Dashboard Overview
The Pattern-1 dashboard that accompanies this MVP is **not** a production monitoring screen. It is a proof-point comparison
panel that GTN stakeholders can use during demos to contrast:

- The Korinsic Pattern-1 MVP (probabilistic fusion with Bayesian explainability), and
- A Scila-style binary rule engine that represents the incumbent baseline.

For the same synthetic feed, the dashboard visualises:

- Alerts fired by each engine.
- How many unique “episodes” of behaviour those alerts represent.
- Precision@k so stakeholders can see alert quality, not just volume.
- Top contributors that drove Korinsic’s fused risk score.

Think of it as a side-by-side storytelling layer that proves Pattern-1 delivers fewer, smarter, explainable alerts.

### Data Basis: Synthetic Orders per 1,000 Events
- The kor-gen scripts synthesise ≈1,000 order events (NEW/MOD/CXL/FILL) spanning the eight scenarios below. No live GTN order
  stream is required.
- Metrics are normalised per 1,000 orders so relative noise is comparable even if the generator is scaled (e.g., 5,000 orders).
- Example: if Scila fires 120 alerts and Korinsic fires 6 on the same 1,000-event pack, the dashboard presents “120 vs 6 alerts
  per 1k orders.”

### Dashboard Metrics
1. **Alerts per 1,000 Orders**
   - Measures alert noise relative to trading flow.
   - Formula: `alerts_per_1k = total_alerts / total_orders * 1000`.
   - Interpretation: “Korinsic reduces alert noise by ~95% for the same trading flow.”
2. **Unique Episodes**
   - Measures how many distinct abusive behaviours occurred vs alerts fired.
   - Each alert carries `episode_id`; Scila emits one per rule, Pattern-1 fuses to one per episode.
   - Interpretation: “Scila produced 120 alerts for eight behaviours; Korinsic produced six alerts for six behaviours.”
3. **Precision@k**
   - Measures alert quality using synthetic ground-truth labels (S1, S3–S7 abusive; S2, S8 benign).
   - Alerts are ranked by risk score (Korinsic) or severity (Scila) and evaluated for top *k* hits.
   - Example: Precision@10 of 0.80 vs 0.30 demonstrates 2.6× higher fidelity.
4. **Top Contributor Bars**
   - Aggregates `top_contributors` from Pattern-1 alerts to show which signals drove the Bayesian score (e.g., fast-path rules,
     sequence motifs, market impact, graph collusion, data-quality dampeners).
   - Visualised as horizontal bars (positive vs negative contributions) to reinforce explainability.

### Example Dashboard Snapshot (text format)

| Metric | Scila | Korinsic | Comment |
| --- | --- | --- | --- |
| Total Orders | 1,000 | 1,000 | Same synthetic dataset |
| Total Alerts | 120 | 6 | 95% reduction |
| Alerts per 1k Orders | 120 | 6 | Noise comparison |
| Unique Episodes | 8 | 6 | Many-to-one collapse |
| Precision@10 | 0.30 | 0.80 | 2.6× higher precision |
| Top Contributors (Korinsic) | – | Rule_Spoof_Burst 58%, Seq_Spoof 42%, Graph 22%, DQSI −15% | Explainable drivers |

### Dashboard Implementation Plan
1. **Synthetic generator.** Produce the eight scenarios below (~1,000 orders) with scenario IDs and abusive/benign labels.
2. **Dual engine run.** Execute the Scila comparator (rule outputs with `alert_id`, `rule_name`, `scenario_id`) and the Pattern-1
   pipeline (fused `alert_id`, `episode_id`, `risk_score`, `top_contributors`).
3. **Metrics collector & storage.** Aggregate alerts-per-1k, distinct episodes, Precision@k, contributor averages, synthetic PnL buckets, and rationale audit records, and persist both the aggregated metrics and the underlying tagged alert rows in an accessible store (e.g., Postgres table + CSV export job) so GTN teams can lift the raw numbers into PowerPoint or BI tooling without rerunning the pipeline.
4. **Renderer.** Build a simple HTML/Plotly panel with:
   - Bar chart: alerts per 1k (Scila vs Korinsic).
   - Table: unique episodes, Precision@k, supporting commentary.
   - Alert table/cards showing fused Pattern-1 cases with scenario tags, risk scores, contributor summaries, synthetic PnL, deterministic rationale text, escalation workflow fields, and ESI banners.
   - Horizontal bar: top contributors (positive vs dampeners).
   - Optional scatter: risk score vs ground truth for additional storytelling.

### Demo Narrative
- “Scila flagged 120 alerts per 1k orders across eight behaviours.”
- “Those 120 alerts boil down to eight unique patterns; Pattern-1 produces one alert per true episode.”
- “Precision@10 is 0.80 for Pattern-1 vs 0.30 for Scila.”
- “Top contributors show cancel bursts, sequence motifs, and graph linkage—transparent, explainable signals.”

## Scenario Coverage (Eight-Scenario Pack)
| # | Typology | Market/Product | Scenario | Expected Scila Behaviour | Expected Korinsic Behaviour |
| --- | --- | --- | --- | --- | --- |
| S1 | Spoofing | Equity – Continuous | Top-of-book microburst | May miss intent, may fire HighCancelRatio | Fast-path Spoof_Burst hard stop, fused alert, spoof ≈ 0.85 |
| S2 | Spoofing | Equity – Liquid | Benign market-maker churn | Multiple FP alerts (cancel ratio, low trade) | Suppressed (score < 0.4), anomaly context only |
| S3 | Spoofing | Futures – Cross-venue | Venue layering/arbitrage drift | Separate venue alerts, no linkage | Single fused alert; venue_hop + motif score ≈ 0.7 |
| S4 | Insider | Equity – Corporate action | Pre-earnings accumulation | PreEventTrade + LargeTradeBeforeEvent | One fused alert; insider ≈ 0.8 with news timeline |
| S5 | Insider | Bond – Rating downgrade | Pre-event sell-down | May miss (equity-tuned thresholds) | Insider ≈ 0.75; directional match + graph link |
| S6 | Insider | Cross-product | Equity front-run of bond news | No cross-product coverage | Insider ≈ 0.8; cross_product + news correlation |
| S7 | Spoofing | Bond futures – Illiquid | Depth layering / small-cap bias | Potential low-confidence alerts | Spoof ≈ 0.7; market_impact_high + anomaly_high |
| S8 | Control | Multi-venue – Mixed flow | Normal retail activity | 10–20 rule alerts (cancel ratios, event proximity) | No fused alerts; all scores < 0.4 |

### Product / Venue Coverage Matrix
| Product Type | Venue Example | Covered Scenarios |
| --- | --- | --- |
| Equities (large-cap) | LSE / NASDAQ | S1, S2, S4 |
| Equities (small-cap/illiquid) | AIM / OTC | S7 |
| Corporate bonds | OTC / MTF | S5 |
| Futures (bond futures) | CME / ICE | S3, S7 |
| Cross-product (bond → equity) | Multi-venue | S6 |
| Retail mixed flow | GTN retail router | S8 |

### Why Eight Scenarios
| Goal | Coverage | Achieved By |
| --- | --- | --- |
| Highlight Scila noise | High FP volume | S2, S8 |
| Expose missed intent | Spoofing/insider gaps | S1, S3, S6 |
| Demonstrate bond relevance | Bond-specific flows | S5, S7 |
| Surface cross-product intelligence | Equity ↔ bond linkage | S6 |
| Showcase Korinsic explainability | Narrative + contributors | All, especially S4–S6 |

Eight scenarios keep demo runtime manageable (<1 day replay) while spanning all GTN-relevant market types.

## Alert Workflow & Data Export Expectations
- **Alert cards/table.** Present each Pattern-1 alert as both a table row and an expanded card so analysts can see fused score, contributing signals, DQ/ESI banners, recommended actions, cross-product links, and workflow status in a single view before escalating to managers.
- **Workflow metadata.** Capture triage timestamps, assigned analyst/manager, disposition notes, and escalation outcome so the same payload powers analyst hand-offs and retrospective reviews.
- **Raw data retention.** Store the enriched alert payloads and comparison metrics in a retrievable format (database table + CSV/Parquet exports) so product, sales, or compliance leads can assemble bespoke dashboards or slide decks without engineering support.
- **Dashboard snapshotting.** Provide a simple export script (CLI or API) that packages the aggregated metrics, raw alert rows, and top-contributor breakdown into a bundle that non-engineers can pull into presentation tooling.
- **Rationale & PnL audit.** Version the rationale templates and PnL configuration so exported data always references the template version and threshold set used to generate the analyst-facing copy.

## Add-On Spec: Natural-Language Rationale & Synthetic PnL Reasoner
### Goals
- Every alert carries a deterministic, human-readable rationale composed from existing evidence; no generative models or hallucinated facts.
- Each episode records a synthetic, mark-to-market PnL summary computed from structured trade and market data.
- Outputs are idempotent, configuration-driven, and remain part of the same alert payload without spawning duplicate alerts.

### Synthetic PnL Reasoner (Feature + Optional Bayesian Node)
**Inputs**
- Episode-scoped trades (account × instrument × `episode_id`) with side, quantity, price, and timestamp.
- Mid-price snapshots around the episode window (`mid(t)`), with configurable post-episode and entry smoothing windows.
- Episode window `[t_start, t_end]` sourced from the stitching logic.
- Configurable thresholds (example):
  ```yaml
  pnl:
    post_window_seconds: 900      # 15 minutes after last trade
    baseline_window_seconds: 60   # smoothing window for entry VWAP
    use_mid_price: true
    fill_slippage_bps: 0
    high_gain_bps: 50
    med_gain_bps: 20
    loss_bps: -10
  ```

**Computation (per episode × instrument)**
1. Compute volume-weighted entry price (`VWAP_entry`).
2. Identify exit mid price `mid_exit = mid(t_end + post_window_seconds)`.
3. Derive directional `pnl_bps`:
   - BUY: `(mid_exit / VWAP_entry − 1) * 10,000`
   - SELL: `(VWAP_entry / mid_exit − 1) * 10,000`
4. Bucket into `HighGain`, `MedGain`, `Neutral`, or `Loss` per thresholds (missing data → `Unknown`).
5. Optionally compute realised PnL if exit trades exist; otherwise rely on the mark-to-market proxy.

**Output**
Persist a feature record similar to:
```json
{
  "episode_id": "ep-123",
  "instrument": "AAPL.O",
  "pnl_bps": 62.4,
  "pnl_bucket": "HighGain",
  "entry_vwap": 189.31,
  "exit_mid": 190.49,
  "post_window_s": 900
}
```

If hooked into the Bayesian network, map the bucket to an observed node (e.g., `Pnl_Gain_High`), otherwise keep it explainer-only with `pnl_feed_bn: false`.

**Acceptance Criteria**
- Deterministic unit tests: fixed trades + mid series → exact `pnl_bps` and bucket.
- Performance: <5 ms per episode in memory.
- Missing mid prices mark the bucket `Unknown`, skip BN update, and append a DQ note.

### Deterministic Natural-Language Rationale Generator
**Principles**
- Template-driven sentences, each mapped to concrete evidence (contributors, timeline events, or PnL feature).
- No external services; re-render deterministically when evidence updates.

**Inputs**
- Typology focus (`spoofing`, `insider`), risk scores, top contributors (name + weight), rule hits, timeline facts, DQSI bucket, scenario metadata (motif ID, graph neighbours, news proximity), and synthetic PnL output.

**Template Structure**
1. **Header**: emphasise dominant typology with score (e.g., “Likely spoofing pattern (score 0.82).”).
2. **Body Clauses** (select top 3 based on available contributors/facts):
   - Fast-path burst (“Cancelled {cancel_ratio}% of orders …”).
   - Sequence motif with `motif_id`.
   - Graph alignment with neighbour list.
   - News proximity with minutes before event.
   - Market impact magnitude.
   - Synthetic PnL summary (`pnl_bps` and window).
   - DQSI caution if `Low`/`Moderate`.
3. **Closer**: recommended next step aligned to typology (device review vs information access/community check).

**Rendering Function**
- Implement in Layer E when assembling the alert payload; enforce max three sentences / 350 characters.
- Use a deterministic helper similar to:
  ```python
  def render_rationale(alert):
      parts = []
      spoof = alert.scores.get("spoofing", 0.0)
      insider = alert.scores.get("insider", 0.0)
      parts.append(
          f"Likely {'spoofing' if spoof >= insider else 'insider'} pattern (score {max(spoof, insider):.2f})."
      )

      contribs = {c["name"]: c for c in alert.top_contributors}
      meta = alert.meta

      if "Rule_Spoof_Burst" in contribs and meta.get("cancel_ratio") is not None:
          parts.append(
              f"Cancelled {meta['cancel_ratio']:.0f}% of orders at top of book within {meta.get('burst_window_s', 1)}s, then traded the opposite side."
          )
      if "Seq_Spoof_Score" in contribs and meta.get("motif_id"):
          parts.append(f"Order sequence matches known micro-burst motif ({meta['motif_id']}).")
      if "Graph_Collusion_Score" in contribs and meta.get("top_neighbors"):
          neighbours = ", ".join(meta["top_neighbors"][:3])
          parts.append(f"Trading aligned with linked accounts {neighbours} within {meta.get('graph_window_s', 30)}s.")
      if "PreNews_Flag" in contribs and meta.get("news_minutes_before") is not None:
          parts.append(f"Built position {meta['news_minutes_before']} minutes before material news.")
      if "Market_Impact_High" in contribs and meta.get("pull_bps") is not None:
          parts.append(f"Mid-price moved {meta['pull_bps']:.0f} bps shortly after activity.")
      if meta.get("pnl_bps") is not None and meta.get("post_window_s") is not None:
          gain_loss = "gain" if meta["pnl_bps"] >= 0 else "loss"
          parts.append(
              f"Episode marked to a {abs(meta['pnl_bps']):.0f} bps {gain_loss} over {meta['post_window_s']}s."
          )
      if alert.dqsi_trust_bucket in {"Low", "Moderate"}:
          parts.append(f"Data quality is {alert.dqsi_trust_bucket}; interpret with caution.")

      if spoof >= insider:
          parts.append("Next: review device metadata and ladder depth at event time.")
      elif meta.get("community_id"):
          parts.append(f"Next: assess information access; inspect community {meta['community_id']}.")
      else:
          parts.append("Next: assess information access and communications context.")

      return " ".join(parts[:3 if len(parts) > 3 else len(parts)])
  ```
- Include `rationale_text` and `rationale_version` fields in the final payload for auditability.

**Acceptance Criteria**
- Snapshot tests assert deterministic output for canonical alert payloads.
- Update tests confirm rationale re-renders when contributors/timeline change (e.g., late graph enrichment).
- Combined PnL + rationale computation adds <10 ms per alert.
- No sentence introduces facts absent from structured evidence.

### Integration Points
- **PnL computation**: run after Layer D fusion (episode windows known) and persist to an `episode_metrics` store keyed by `episode_id`.
- **Rationale generation**: execute in Layer E before emission and on any subsequent enrichment updates to keep `alert_id` payloads consistent.
- **Payload additions**:
  ```json
  {
    "meta": {
      "pnl_bps": 62.4,
      "post_window_s": 900,
      "pull_bps": 8.0,
      "motif_id": "S3",
      "top_neighbors": ["acct-9", "acct-12"],
      "community_id": "comm-17",
      "news_minutes_before": 22
    },
    "rationale_text": "Likely insider pattern (score 0.78). Built position 22 minutes before material news. Episode marked to a 62 bps gain over 900s. Next: assess information access; inspect community comm-17.",
    "rationale_version": "1.0.0"
  }
  ```
- **Config toggles**:
  ```yaml
  features:
    compute_pnl: true
    pnl_feed_bn: false
  rationale:
    enabled: true
    max_sentences: 3
  ```

### Testing & Audit Expectations
- Unit tests for PnL calculation and rationale rendering (golden files).
- Enrichment update test ensuring rationale refreshes when new contributors (e.g., graph signals) arrive.
- Performance instrumentation verifying combined overhead stays within 10 ms per alert.
- Template and config versioning recorded in exports so dashboards and slide decks cite the correct rationale/PnL logic.

### Demo Talk Track
“Each Korinsic alert now carries a short, plain-English rationale and a directional PnL summary for the episode. This proves motive alignment and provides analyst-ready narration without extra workload. Both outputs are deterministic, auditable, and update as additional evidence arrives.”

## Development Workstreams
### 1. Ingestion Service
- Implement `src/pattern1/ingest/market_stream.py` to manage provider auth, session lifecycle, reconnect/backoff, and micro-batching.
- Extend FastAPI ingress (`/ingest/market`) to accept normalised payloads for manual tests while the stream publishes to an in-process queue (asyncio or Redis Streams).
- Configure environment variables (`PATTERN1_MARKET_API_KEY`, endpoints) and document them in the runbook.

### 2. Data Processor Integration
- Route live ticks through `DataProcessor.process_realtime` to reuse existing metric shaping.
- Preserve compatibility with current downstream consumers until dedicated Pattern-1 feature windows are populated.

### 3. Layer A Fast-Path Rules
- Capture YAML definitions for Spoof_Burst, Quote_Stuffing, and PreNews_Accum.
- Wrap `AnalysisService.analyze_realtime_data` with a predicate runner that evaluates the rules per burst and forwards qualifying events to `AlertService.generate_realtime_alerts`.
- Ensure alert IDs persist for downstream enrichment layers.

### 4. Layer B Anomaly Evidence
- Feed processed snapshots into the anomaly mapper, compute robust z-scores, and rank feature contributions for inclusion in the alert payload.
- Validate that Scenarios 2 and 6 display anomaly context without triggering fused alerts.

### 5. Layer C1 Motif Enrichment
- Tokenise sequences (2–10 s windows), match against motif catalogues, and attach `motif_id`, rarity scores, and excerpts to the alert timeline within 2 s of Layer A trigger.

### 6. Layer C2 Graph Coordination
- Reuse cross-desk collusion processing to compute coordination metrics every 5 s and asynchronously update existing alert IDs with `community_id`, neighbour list, and `graph_collusion_score`.

### 7. Layer D Bayesian Fusion & Evidence Sufficiency
- Feed evidence bundles into the `BayesianEngine`, apply `ComplexRiskAggregator`, and record signal contributions, priors, and leak probabilities for UI explainers.
- Execute `apply_fallback_evidence`, compute ESI, and set DQSI dampeners for low-quality feeds (Scenario 5).

### 8. Scenario Tagging & Replay Control
- Annotate events with scenario IDs via feature thresholds to differentiate the eight-scenario pack (core MVP plus cross-venue/cross-product extensions).
- Maintain CLI controls to toggle live/replay/blended modes and ingest nightly capture files.

### 9. UI Payload & Comparison Chips
- Extend alert payloads with scenario metadata, naive rule comparison stats (alerts per 1k orders, overlap), motif excerpts, anomaly highlights, graph context, and ESI banners.
- Ensure `/api/v1/alerts/history` exposes the enriched payload expected by the Pattern-1 dashboard.

### 10. Observability & Runbook
- Instrument ingestion and processing latency, provider throttling, reconnection counts, and alert emission timings.
- Draft `docs/pattern1/RUNBOOK.md` covering setup, environment variables, CLI commands, troubleshooting, compliance considerations, and nightly capture processes.

### 11. Testing Strategy
- Expand synthetic CLI regression to include all eight scenarios with assertions on fused tiers, evidence payloads, and naive comparison outputs.
- Add smoke tests for live ingestion (15-minute stream) verifying alert generation, latency SLOs (<150 ms), and resilience under reconnect conditions.
- Record acceptance results in CI and document `make`/CLI commands in the runbook.

## Developer Onboarding Checklist
1. Provision provider API keys and configure environment variables.
2. Build and test the market stream adapter against the provider sandbox.
3. Wire the adapter into the ingestion queue and confirm alerts through `/api/v1/alerts/history`.
4. Implement YAML fast-path rules and verify Scenario 1 hard-stop behaviour.
5. Populate anomaly, motif, graph, and Bayesian enrichment paths; validate Scenario 2–5 payloads.
6. Integrate naive comparison metrics and confirm Scenario 6 suppression narrative.
7. Complete observability instrumentation and runbook documentation.
8. Execute full regression (live + synthetic) before demo sign-off.

## Deliverables Summary
- `src/pattern1/ingest/market_stream.py` (live adapter with resilience controls).
- YAML rule configuration for Layer A predicates.
- Updated evidence mapping, anomaly/motif/graph modules, and Bayesian CPT configuration.
- Enriched alert payloads with scenario IDs, narratives, comparison metrics, ESI banners.
- CLI/regression scripts for eight scenarios and live ingestion smoke test.
- `docs/pattern1/RUNBOOK.md` detailing operations and troubleshooting.

## Success Criteria
- Live stream operates for ≥15 minutes on AAPL.O and TSLA.O with fast-path alerts visible in the UI.
- All eight scenarios hit their checkpoints (fast-path demo, anomaly context, motif surfaced, graph enrichment, fused tiers, confidence banner, comparison narrative).
- CI regression green with deterministic synthetic fixtures.
- Runbook enables “three commands” setup for demos.

