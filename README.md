# Enterprise Real-Time Power AI Intelligence Platform

A single Colab notebook that runs a full support-intelligence pipeline end to end on a real
Kaggle dataset: **ingest → validate → quality-gate → Delta Lakehouse (Bronze/Silver/Gold) →
embed & index → real retrieval → Groq LLM answers → evaluation → real-time dashboard** — with
**labeled output and row counts printed after every stage**.


---

## Table of contents

1. [What it does](#what-it-does)
2. [Dataset](#dataset)
3. [Quick start](#quick-start)
4. [Pipeline stages & the output you'll see](#pipeline-stages--the-output-youll-see)
5. [Configuration](#configuration)
6. [Deliverables mapping](#deliverables-mapping)


---

## What it does

Support teams face a flood of repetitive tickets. This platform ingests tickets, checks their
quality, stores them in a medallion lakehouse, indexes past agent answers into a vector store,
and then — for any new question — retrieves the most relevant knowledge and drafts a **cited
answer** using a fast Groq-hosted LLM, while monitoring latency and answer quality in real time.

The emphasis of this build is **visibility**: you can watch the data transform at each step,
with row counts and samples printed after ingestion, quality gate, Bronze, Silver, Gold, and
indexing.

---

## Dataset

**Name:** Multilingual Customer Support Tickets (Synthetic)
**Source:** Kaggle — `tobiasbueck/multilingual-customer-support-tickets`
**Loaded via:** `kagglehub` (no API token needed in Colab)

**Raw columns:** `subject`, `body`, `answer`, `type`, `queue`, `priority`, `language`, `tags`,
plus a few others.

**Normalization done in the notebook:**
- `body` -> `description`
- `answer` -> `agent_response` (this becomes the RAG knowledge source)
- `queue` -> `section`
- Missing values filled with `"unknown"`
- Filtered to English and capped at 50 rows for a fast, readable demo (raise the cap for
  larger runs)

The dataset has no `ticket_id` or timestamp; the pipeline works from the natural fields and
synthesizes what it needs.

---

## Quick start

1. Open **`REGENERATED_ENTERPRISE_POWER_AI_FIXED.ipynb`** in Google Colab.
2. (Recommended) `Runtime -> Change runtime type -> T4 GPU`.
3. (Optional, for real LLM answers) add your Groq key in the Colab **Secrets** panel
   (key icon) as `GROQ_API_KEY`. Without it, the notebook runs in offline mode with extractive
   answers — it still completes fully.
4. `Runtime -> Run all`.

Get a free Groq key at console.groq.com -> API Keys.

---

## Pipeline stages & the output you'll see

Each stage prints a labeled box with a **row count** and a **data sample**, so you can see the
result of every transformation. The main run ends with a consolidated summary table.

| Stage | What prints |
|---|---|
| **1 — Ingestion (Kafka)** | count produced, count valid (schema pass), count sent to DLQ, DLQ error sample |
| **2 — Quality Gate** | count passed, count quarantined, quarantined subjects + their scores |
| **3 — Bronze** | row count + sample of raw immutable records (`subject`, `priority`, `language`) |
| **4 — Silver** | row count + sample with enriched columns (`is_resolved`, `desc_len`) |
| **5 — Gold** | the analytics aggregate (tickets by priority) + total across groups |
| **6 — Vector index** | vectors indexed + embedding dimension + sample IDs |
| **Row-count summary** | one table showing rows at every stage from raw -> indexed |

Example of the summary table you'll see:

```
  PIPELINE ROW-COUNT SUMMARY
====================================================================
                  stage  rows
      1. Produced (raw)    52
 2. Valid (schema pass)    51
   -> DLQ (schema fail)     1
      3. Quality passed    50
         -> Quarantined     1
              4. Bronze    50
              5. Silver    50
6. Gold (priority grps)     3
     7. Vectors indexed    50
```

After the pipeline, the notebook also prints:
- **Retrieval + Groq answers** for demo queries (top chunks with real cosine scores, the cited
  answer, and per-query `precision@3` / `grounded` checks).
- A **real-time dashboard**: queries served, average & p95 latency, grounded-rate, and SLA
  alerts.
- The **lineage** and **audit** trail (events grouped by stage).

---


## Configuration

Edit these near the top of the notebook:

- **Sample size / language** — in the dataset cell, change the `.head(50)` cap or remove the
  English filter to process more rows.
- **`GROQ_MODEL`** — defaults to `llama-3.1-8b-instant`; a fallback model is used automatically
  if that one is unavailable.
- **`temperature` / `max_tokens`** — set to `0` / `150` for deterministic, concise support
  answers.
- **Dashboard thresholds** — `RealTimeIntelligence(p95_sla_ms=3000, grounded_floor=0.6)`
  controls when alerts fire.

---

## Deliverables mapping

| Deliverable | Where in the notebook |
|---|---|
| 1 — Ingestion (Kafka producer/consumer, schema validation, DLQ, retry) | Sections 5-6, run in Section 13 |
| 2 — Delta Lakehouse (Bronze/Silver/Gold) | Section 8, run + printed in Section 13 |
| 3 — RAG (embed, index, real retrieval, citations) | Section 9, demoed in Section 13.1 |
| 5 — Quality Gate (scoring, quarantine) | Section 7, printed in Section 13 |
| Governance (OpenLineage-style lineage + audit) | Section 10, trail in Section 14.1 |
| Groq LLM answers | Section 11 |
| Evaluation (precision@k, groundedness) | Section 12, applied in Section 13.1 |
| Real-time intelligence dashboard | Section 14 |

