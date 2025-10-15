# Korinsic Platform — Tech Functional Diagram

This one-pager summarizes how client data flows into the Korinsic surveillance platform, how the core engines transform that data into risk intelligence, and how downstream teams consume the outputs. The layout reads left to right across three primary zones.

```mermaid
flowchart LR
    %% Left zone: external connectors feeding Korinsic
    subgraph Connectors[Client Connectors & Adaptors]
        MarketData["Market Data\n(FIX, market tapes, curves)"]
        OrderData["Order & Execution Feeds\n(OMS, EMS, blotters)"]
        TradeData["Trade & Position Data\n(ETD, OTC, commodities)"]
        CommsData["Communications & Voice\n(voice-to-text, chat, email)"]
        HRData["HR, KYC & Reference\n(staff roles, accounts, hierarchies)"]
        RefData["Reference & Config\n(limits, instruments, case tags)"]
    end

    %% Middle zone: Korinsic core services and orchestration
    subgraph Core[Korinsic Core Intelligence Layer]
        DQGate["Data Quality & Health Gate\n(completeness, sufficiency, drift)"]
        Normalizer["Event Normalizer & Feature Fabric\n(schema harmonization, enrichment)"]
        ModelService["Model Service & Signal Factory\n(Bayesian typologies, intent models)"]
        RiskSynth["Risk Synthesis Engine\n(alert clustering, scenario rollups)"]
        Explainability["Explainability & Evidence Service\n(audit trails, regulatory rationale)"]
        Orchestrator["Real-time & EOD Orchestrator\n(workflows, scheduling, escalation)"]
    end

    %% Right zone: outward facing delivery surfaces
    subgraph Delivery[API Delivery & Investigator Experience]
        APIGateway["API Gateway\n(/api/v1 REST, streaming hooks)"]
        AlertAPI["Alerting & Case APIs\n(alert payloads, case sync)"]
        ConfigAPI["Configuration APIs\n(model registry, thresholds)"]
        UI["Korinsic Surveillance UI\n(dashboards, explainability views)"]
        ClientApps["Client Applications & Integrations\n(case mgmt, downstream risk)"]
    end

    %% Connector flows into the core
    MarketData --> DQGate
    OrderData --> DQGate
    TradeData --> DQGate
    CommsData --> DQGate
    HRData --> DQGate
    RefData --> DQGate

    %% Internal sequencing within the core layer
    DQGate --> Normalizer
    Normalizer --> ModelService
    ModelService --> RiskSynth
    RiskSynth --> Explainability
    Explainability --> Orchestrator

    %% Outbound orchestration into delivery layer
    Orchestrator --> APIGateway
    Orchestrator --> AlertAPI
    Orchestrator --> ConfigAPI
    AlertAPI --> UI
    APIGateway --> UI
    ConfigAPI --> UI
    UI --> ClientApps
```

## Zone definitions

- **Client Connectors & Adaptors.** Upstream systems deliver market, order, trade, communications, and HR reference feeds through adaptable connectors so Korinsic ingests the same evidence clients already trust.
- **Korinsic Core Intelligence Layer.** Data is validated for quality, normalized into a common schema, scored by Bayesian typologies, synthesized into risk scenarios, and paired with explainable evidence before the orchestrator dispatches alerts in real time or scheduled end-of-day batches.
- **API Delivery & Investigator Experience.** REST and streaming APIs expose alerts, model configurations, and case synchronizations to Korinsic's UI and to the client's existing operational tooling.
