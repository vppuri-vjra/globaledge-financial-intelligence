# GlobalEdge Financial Intelligence Assistant

A no-code RAG-based market intelligence system built with **n8n**, **Pinecone**, and **OpenAI** for GlobalEdge Brokerage — enabling 400 equity brokers to query financial news, stock prices, and SEC filings in plain English before client calls.

UT Austin "AI Agents and Generative AI for Business Applications" (Great Learning) final submission — graded PDF in `docs/`.

## Business Context

GlobalEdge Brokerage is a mid-sized retail and institutional brokerage firm operating across 12 countries, serving ~180,000 active clients through 400 equity brokers. Each broker handles 30–50 client interactions daily but can only read 20–30% of the 60–80 morning news articles before the first call — creating a structural intelligence gap at the moment of client contact.

**Problem:** Brokers give recommendations based on general knowledge rather than today's actual signals. They can't cite sources when challenged, creating trust erosion and compliance exposure.

**Solution:** A natural language market intelligence system that ingests financial news, stock prices, and SEC filings into a vector store, making them queryable through plain-language questions with sourced, grounded answers and automated quality evaluation.

## Architecture

### Workflow 1: Data Ingestion (`VP_GlobalEdge Data Ingestion`)

```
Manual Trigger
    ├── Financial Market News (global_news.csv)
    ├── SEC Filing (sec_filings_10q.pdf)
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

**Actual execution:** 3 documents processed → 1,001 chunks generated → 2,444 embeddings created.

### Workflow 2: QA RAG with LLM-as-Judge — 3 Configurations Tested

```
Chat Trigger (user question)
    │
    ├──► Question and Answer Chain
    │     ├── Chat Model (GPT-4.1, temp 0)
    │     ├── Retriever (Top-K per config)
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

The assignment held chunk size (500), chunk overlap (50), temperature (0), and top-p (1) constant, and varied only **Top-K retrieval** and **Max Tokens** across three configurations — evaluated across 5 real financial test cases (15 total LLM-as-a-Judge runs):

| Config | Top-K | Max Tokens | Avg Overall Score | Test Cases Won |
|--------|-------|-----------|-------------------|-----------------|
| **A — Recommended** | 8 | 1000 | **8.96 / 10** | 3 of 5 |
| B | 5 | 800 | 8.76 / 10 | 1 of 5 |
| C | 10 | 1500 | 8.64 / 10 | 1 of 5 |

## Test Cases (real broker questions)

1. Based on this week's news sentiment around Amazon, would you classify the current signal as bullish or bearish, and does the price data support that?
2. Should I invest in Apple right now? I heard Tim Cook is stepping down — is that a red flag? How's the stock been doing, and is there anything I should be worried about with the company's financials or risks?
3. Is now a good time to buy Bitcoin? News is saying it just crossed $78k — am I too late? What's the price trend looked like over the last few months, and what's driving the current rally?
4. J.P. Morgan is calling for the S&P 500 to hit 7,600 on AI earnings. Is that realistic? Should I buy in now or wait?
5. I read in the news that Nvidia is leading the AI rally. How has NVDA's stock been performing compared to the news sentiment?

## Key Findings

- Configuration A (top-k=8, 1,000 tokens) delivered the strongest overall performance in Test Cases 1–3 and the highest average score (8.96/10) across all 15 runs.
- Configuration C (top-k=10, 1,500 tokens) won only Test Case 4; its extra retrieval depth hurt Test Case 5 enough to trigger a "Needs Improvement" verdict (7.4) for omitting risk discussion despite good relevance/accuracy.
- Configuration B (top-k=5, 800 tokens) produced the best balance of quality and efficiency for Test Case 5, after the QA system prompt was refined based on initial judge feedback.
- **Increasing retrieval depth and response budget alone did not reliably improve answer quality** — in several cases it slightly reduced groundedness and accuracy by introducing unnecessary context. Prompt engineering and evaluation criteria had a greater measured impact than retrieval depth.

## Data Sources

| # | File | Type | Description |
|---|------|------|-------------|
| 1 | `global_news.csv` | CSV | 500 news articles — stock markets, Fed policy, oil, crypto, tech earnings |
| 2 | `stock_price_details.csv` | CSV | 500 daily price records — stocks/ETFs/FX/crypto (AAPL, TSLA, BTC-USD, S&P 500, etc.) |
| 3 | `sec_filings_10q.pdf` | PDF | 126-page SEC 10-Q filing — Alphabet, Apple, Tesla financial statements, risk factors, legal disclosures |

## Technical Stack

| Component | Technology | Details |
|-----------|-----------|---------|
| Orchestration | n8n | No-code workflow automation |
| Vector DB | Pinecone | Index: `global-edge`, Namespace: `globaledge-rag` |
| Embeddings | OpenAI `text-embedding-3-large` | 1024 dimensions |
| Chunking | Recursive Text Splitter | 500 chars, 50 overlap |
| LLM (QA) | GPT-4.1 | temp 0, 3 configs (Top-K 5/8/10, Max Tokens 800/1000/1500) |
| LLM (Judge) | GPT-4.1 | temp 0, 5-metric evaluation |
| Retrieval | Config A (recommended) | Top-K=8, cosine similarity |

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
│   ├── GlobalEdge Business Context.pdf
│   └── Vipin_Puri_GlobalEdge Financial Intelligence Assistant.pdf  (graded submission)
├── workflows/
│   ├── VP_GlobalEdge Data Ingestion.json
│   ├── Config_A_VP_GlobalEdge_QA_RAG.json   (top-k=8, 1000 tokens — recommended)
│   ├── Config_B_VP_GlobalEdge_QA_RAG.json   (top-k=5, 800 tokens)
│   └── Config_C_VP_GlobalEdge_QA_RAG.json   (top-k=10, 1500 tokens)
└── data/
    ├── global_news.csv
    ├── stock_price_details.csv
    └── sec_filings_10q.pdf
```

## Live Demo

**[vjra.us/globaledge.html](https://vjra.us/globaledge.html)**
