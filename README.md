# GlobalEdge Financial Intelligence Assistant

A no-code RAG-based market intelligence system built with **n8n**, **Pinecone**, and **OpenAI** for GlobalEdge Brokerage — enabling 400 equity brokers to query financial news, stock prices, and SEC filings in plain English before client calls.

## Business Context

GlobalEdge Brokerage is a mid-sized retail and institutional brokerage firm operating across 12 countries, serving ~180,000 active clients through 400 equity brokers. Each broker handles 30–50 client interactions daily but can only read 20–30% of the 60–80 morning news articles before the first call — creating a structural intelligence gap at the moment of client contact.

**Problem:** Brokers give recommendations based on general knowledge rather than today's actual signals. They can't cite sources when challenged, creating trust erosion and compliance exposure.

**Solution:** A natural language market intelligence system that ingests financial news, stock prices, and SEC filings into a vector store, making them queryable through plain-language questions with sourced, grounded answers and automated quality evaluation.

## Architecture

### Workflow 1: Data Ingestion (`VP_GlobalEdge Data Ingestion`)

```
Manual Trigger
    ├── Financial Market News (global_news.csv)
    ├── SEC Filing (Apple Q2 2026 Form 10Q) (sec_filings_10q.pdf)
    └── Stock Market Data (stock_price_details.csv)
            │
            ▼
    Document Loader (binary mode)
            │
            ▼
    Recursive Text Splitter (500 chars, 50 overlap)
            │
            ▼
    OpenAI Embeddings (text-embedding-3-large, 1024-dim)
            │
            ▼
    Pinecone Vector Store
      Index: global-edge
      Namespace: globaledge-rag
```

### Workflow 2: QA RAG with LLM-as-Judge (`VP_GlobalEdge_QA_RAG`)

```
Chat Trigger (user question)
    │
    ├──► Question and Answer Chain
    │     ├── Chat Model (GPT-4.1, temp 0.3, max 1000 tokens)
    │     ├── Retriever (Top-K=8)
    │     │     └── Pinecone Vector Store (global-edge / globaledge-rag)
    │     │           └── OpenAI Embeddings (text-embedding-3-large, 1024-dim)
    │     └── System Prompt (grounded answers only, cite evidence, confidence level)
    │
    ├──► Pinecone Search (Get Many) — parallel retrieval for evaluation context
    │
    └──► Merge Results
              │
              ▼
         LLM-as-a-Judge Evaluation (GPT-4.1)
         Scores: Relevance, Groundedness, Accuracy, Completeness, Risk Awareness
              │
              ▼
         Extract Evaluation Metrics (regex parsing)
              │
              ▼
         Insert row → n8n DataTable (RAG_Evaluation_Results)
              │
         Chat Output → User sees answer
```

## Data Sources

| # | File | Type | Description |
|---|------|------|-------------|
| 1 | `global_news.csv` | CSV | News articles from 114 sources — stock markets, Fed, oil, crypto |
| 2 | `stock_price_details.csv` | CSV | Daily prices for 23 stocks/ETFs/FX/crypto (AAPL, TSLA, BTC-USD, Gold, etc.) |
| 3 | `sec_filings_10q.pdf` | PDF | 126-page SEC 10-Q filing (Q1 2026) — Alphabet, Apple, Tesla financial statements, risk factors, legal disclosures |

## Technical Stack

| Component | Technology | Details |
|-----------|-----------|---------|
| Orchestration | n8n | No-code workflow automation |
| Vector DB | Pinecone | Index: `global-edge`, Namespace: `globaledge-rag` |
| Embeddings | OpenAI `text-embedding-3-large` | 1024 dimensions |
| Chunking | Recursive Text Splitter | 500 chars, 50 overlap |
| LLM (QA) | GPT-4.1 | temp 0.3, max 1000 tokens |
| LLM (Judge) | GPT-4.1 | 5-metric evaluation |
| Retrieval | Top-K=8 | Cosine similarity |

## LLM-as-a-Judge Evaluation Metrics

| Metric | Description |
|--------|-------------|
| Relevance | Is the answer relevant to the question? |
| Groundedness | Is the answer grounded in retrieved context? |
| Accuracy | Are the facts correct? |
| Completeness | Does it cover all aspects of the question? |
| Risk Awareness | Does it highlight risks and uncertainties? |
| Overall Score | Average of all 5 metrics (1-10) |
| Verdict | Pass / Needs Improvement |

## System Prompt

```
You are the GlobalEdge Financial Intelligence Assistant.
Answer questions using only the retrieved context.
- Do not make up facts, assumptions, or conclusions
- If info not available, state clearly
- Highlight risks, uncertainties, and assumptions
- Cite supporting evidence
- Provide: Answer, Evidence, Confidence Level
```

## File Structure

```
globaledge-financial-intelligence/
├── README.md
├── docs/
│   └── GlobalEdge Business Context.pdf
├── workflows/
│   ├── VP_GlobalEdge Data Ingestion.json
│   └── VP_GlobalEdge_QA_RAG.json
└── data/
    ├── global_news.csv
    ├── stock_price_details.csv
    └── sec_filings_10q.pdf
```

## Live Demo

**[vjra.us/globaledge.html](https://vjra.us/globaledge.html)**
