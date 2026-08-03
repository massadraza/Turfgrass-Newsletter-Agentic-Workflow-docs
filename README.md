# Turfgrass Weekly — Architecture

An AI-generated weekly newsletter for golf course superintendents and turf managers. A LangGraph pipeline scrapes weather, disease/pest dashboards, academic research, industry news, and social content — classifies and routes it through parallel GPT‑4o writing agents, fact-checks and assembles a personalized per-ZIP HTML newsletter, gates it behind human review, and delivers it by email.

A companion FastAPI app handles subscriber signup/unsubscribe, feedback collection, human-in-the-loop (HITL) approval, and serves a React admin dashboard.

> This repo documents the system design of a private production codebase. It contains no source code — just the architecture writeup below.

## Architecture

The pipeline is a [LangGraph](https://langchain-ai.github.io/langgraph/) `StateGraph` over a single shared state object, run in six layers.

```mermaid
flowchart TD
    subgraph L1["Layer 1 — Parallel Ingestion"]
        direction LR
        A1[weather_pull]
        A2[content_aggregator]
        A3[local_reference]
        A4["academic_research (OpenAlex)"]
        A5[twitter_scrape]
        A6[transcript_fetcher]
        A7[podcast_transcriber]
        A8["cornell_dashboard (vision)"]
        A9["msu_gdd (vision)"]
    end

    L1 --> M[merge_ingestion]
    M --> T["content_triage (Layer 1.5)<br/>7 topic buckets"]
    T --> O["orchestrator<br/>decides sections + fan-out via Send()"]

    O --> L2Z["Layer 2 — per-ZIP agents<br/>weather_agent · disease_agent · pest_agent"]
    O --> L3S["Layer 3 — shared agents<br/>herbicide · industry_news · academic<br/>superintendent · gdd_phenology · podcast_video<br/>condensed_reports · meme"]

    L2Z --> E[evaluator<br/>re-dispatch if empty/short]
    L3S --> E

    E --> PG["plagiarism_guard"]
    PG -- flagged --> RW[rewrite] --> PG
    PG -- clean --> CC["crosscheck_agent"]
    CC -- failed --> RV[revise] --> CC
    CC -- passed --> NA["newsletter_assembly"]

    NA --> SD[source_dedup]
    SD --> WCA[works_cited_audit]
    WCA --> HG{{"hitl_gate<br/>(graph interrupts, waits for admin)"}}

    HG -- approved --> DEL[delivery]
    DEL --> RAG["rag_update<br/>embeds into pgvector"]

    HG -- rejected --> RR["re-run flagged section's agent"]
    RR --> NA
```

`rag_update` embeds the finished newsletter, reviewer corrections, disease/pest trends, and academic papers into pgvector tables that later runs' agents query for grounding.

### Content triage buckets

Triage sorts scraped content into buckets that map 1:1 to newsletter sections.

| Bucket | Populated by | Section |
|---|---|---|
| `weather` | LLM classification | Weather |
| `disease` | LLM classification (also covers fungicide application/timing/resistance content) | Disease |
| `pest` | LLM classification (bypassed when MSU GDD Tracker data is available for the zip) | Pest |
| `herbicide` | LLM classification | Herbicide / Weed |
| `industry` | LLM classification (default bucket for anything unclassified) | Industry News |
| `academic` | Direct DB insert from OpenAlex, not LLM-classified — fallback bucket only | Academic Spotlight |
| `podcast_video` | Routed by content type, no LLM call | Podcast & Video |
| `superintendent` | Routed by known X/Twitter handles, no LLM call | Superintendent Voices |

`gdd_phenology`, `condensed_reports`, and `meme_of_the_week` pull directly from weather data or the raw article pool and don't use triage buckets.

### Zip-personalization

Weather, disease, pest, and GDD phenology are zip-dependent — the orchestrator fans out one `Send()` per unique subscriber zip so each gets locally-relevant content. Shared/national sections (herbicide, industry news, academic, podcast/video, superintendent) run once.

### Quality gates

- **Plagiarism guard** — flags copied phrasing, retries the section's writer with a rewrite instruction (max 2 retries).
- **Crosscheck agent** — a Claude-based fact-checker that verifies claims in each section's HTML against its cited source articles and any explicitly trusted structured data (e.g. weather API numbers), stripping or correcting unsupported claims.
- **Works cited audit** — verifies the final works-cited list matches in-body citations.
- **HITL gate** — the graph interrupts and waits for an admin to approve or reject via email or the admin dashboard before sending. Rejection re-runs just the flagged section and loops back through assembly.

## System components

- **Pipeline** — LangGraph graph definition, orchestration, and per-section writing agents (GPT‑4o + Claude).
- **Web app** — FastAPI service handling subscriber management, feedback, and HITL approval endpoints.
- **Admin dashboard** — React + TypeScript (Vite) frontend for reviewing/approving newsletters and managing subscribers.
- **RAG store** — Postgres + pgvector, indexed with prior newsletters, reviewer corrections, disease/pest trends, and academic papers, queried by writing agents for grounding.
- **Evals** — a harness for scoring pipeline output quality against reference criteria.

## Stack

Python, LangGraph, FastAPI, Postgres/pgvector (Supabase), React/TypeScript, GPT-4o, Claude, Playwright (dashboard scraping), OpenAlex API, Twitter/YouTube APIs.
