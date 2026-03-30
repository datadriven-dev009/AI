# AlloyDB Quick Setup Lab — Concepts & Explanations
> **Codelab:** One-click AlloyDB cluster + instance setup on Google Cloud | **Duration:** Short

---

## 1. What This Codelab Builds

A fully operational **AlloyDB cluster and instance** on Google Cloud using a simplified one-click/script-based setup — ready to use as a database backend for any application or AI workload.

### Architecture Overview

```
Google Cloud Project
        │
        └── AlloyDB Cluster
                │
                └── Primary Instance
                        │
                        └── Database Tables (test data loaded)
```

---

## 2. What is AlloyDB?

**AlloyDB for PostgreSQL** is Google Cloud's fully managed, PostgreSQL-compatible database service — but fundamentally redesigned for **performance, AI integration, and enterprise scale**.

> It is NOT just a managed PostgreSQL. It's a re-engineered engine built for the AI era.

---

## 3. Key Features & Why AlloyDB Stands Out

### 3.1 HTAP — Hybrid Transactional & Analytical Processing

| Traditional DB | AlloyDB |
|---------------|---------|
| OLTP (transactions) only | OLTP + OLAP in one engine |
| Need a data warehouse for analytics | Built-in **Columnar Engine** |
| Complex ETL pipelines required | Real-time BI on operational data |
| Analytics are slow | Analytical queries up to **100x faster** than standard PostgreSQL |

The **Columnar Engine** automatically keeps frequently-queried data in a column store in memory — no manual configuration needed.

---

### 3.2 Native AI / Vertex AI Integration

- Uses the **`google_ml_integration`** PostgreSQL extension
- Lets you call **Vertex AI models (Gemini)** directly from SQL queries
- Use cases inside SQL:
  - Sentiment analysis
  - Language translation
  - Entity extraction
  - Text classification

```sql
-- Example: Call Gemini directly from SQL
SELECT google_ml.predict_row(...)
FROM my_table;
```

Benefits:
- **Data stays in the database** → no data movement, higher security
- **Minimal latency** → no round-trips to external APIs
- AI operations as standard DB transactions

---

### 3.3 Superior Vector Search (RAG-ready)

| Standard PostgreSQL | AlloyDB |
|--------------------|---------|
| pgvector extension | pgvector **+ ScaNN index** |
| HNSW indexes | **ScaNN** (Scalable Nearest Neighbors) |
| Slower at scale | Significantly faster similarity search |
| Lower recall | **Higher recall** at scale |

**ScaNN** was developed by Google Research and powers Google's own large-scale vector search systems. AlloyDB is the only managed PostgreSQL that includes it.

> Perfect for building **RAG (Retrieval Augmented Generation)** applications natively in the database.

---

### 3.4 Performance at Scale

- **4x faster** transactional performance than standard PostgreSQL
- **Compute and storage scale independently** (cloud-native architecture)
- The storage layer handles **Write-Ahead Logging (WAL) processing** — offloading work from the primary compute instance
- Result: Higher throughput with lower CPU pressure on primary

---

### 3.5 Enterprise Availability

- **99.99% uptime SLA** — including during maintenance windows
- Achieved through cloud-native architecture (not just replication)
- Rapid failure recovery
- Storage durability built into the architecture

---

## 4. AlloyDB Concepts

### 4.1 Cluster
- The **top-level container** in AlloyDB
- Holds all resources: databases, logs, backups, instances
- Has a single geographic region
- You delete a cluster to remove everything inside it

### 4.2 Primary Instance
- The **main read/write access point** for a cluster
- Every cluster has exactly one primary instance
- Applications connect through the primary instance
- Can have read pool instances added for scaling reads

### 4.3 Tables
- Standard PostgreSQL tables inside the database
- The test dataset in this lab is loaded into tables within the cluster

### 4.4 Cluster Hierarchy

```
AlloyDB Cluster
    ├── Primary Instance  ← read/write endpoint
    ├── Read Pool Instance (optional) ← scale reads
    └── Databases
            └── Tables (your data lives here)
```

---

## 5. Public IP vs Private IP

| | Private IP | Public IP |
|--|-----------|----------|
| **Default** | ✅ Yes | ❌ No |
| **Security** | More secure | Less secure |
| **Access** | Within VPC only | From anywhere internet |
| **Use case** | Production | Development/testing only |

### For Local Development Testing:
- Enable **Public IP** on the instance
- Set Authorized External Networks to `0.0.0.0/0`
- ⚠️ **Remove and disable Public IP when done** — never leave open in production

---

## 6. Setup Method — What the Script Does

The `run.sh` script automates:
1. Takes your **Project ID**, **Cluster name**, **Instance name** as input via a web UI
2. Creates the AlloyDB cluster in your project
3. Provisions the primary instance
4. Loads the test dataset into tables

This process takes several minutes (coffee break time ☕) as AlloyDB provisions the cluster and sets up the storage layer.

---

## 7. AlloyDB vs Other Databases

| Feature | Standard PostgreSQL | Cloud SQL for PostgreSQL | AlloyDB |
|---------|--------------------|-----------------------|---------|
| HTAP | ❌ | ❌ | ✅ |
| Native AI/Gemini | ❌ | ❌ | ✅ |
| ScaNN vector search | ❌ | ❌ | ✅ |
| Transactional speed | Baseline | ~1x | **4x faster** |
| Analytical speed | Slow | Slow | **100x faster** |
| Uptime SLA | — | 99.95% | **99.99%** |
| Compute/Storage split | ❌ | ❌ | ✅ |

---

## 8. Key Terms Glossary

| Term | Meaning |
|------|---------|
| **AlloyDB** | Google's fully managed, PostgreSQL-compatible database for AI/analytics |
| **Cluster** | Top-level AlloyDB container holding instances and databases |
| **Primary Instance** | Main read/write endpoint of an AlloyDB cluster |
| **Columnar Engine** | In-memory column store for fast analytical queries |
| **HTAP** | Hybrid Transactional and Analytical Processing |
| **ScaNN** | Scalable Nearest Neighbors — Google's fast vector similarity index |
| **RAG** | Retrieval Augmented Generation — AI pattern using vector search |
| **google_ml_integration** | PostgreSQL extension to call Vertex AI models from SQL |
| **WAL** | Write-Ahead Log — mechanism for data durability and replication |
| **pgvector** | PostgreSQL extension for storing and querying vector embeddings |
| **Public IP** | External network access — for dev/testing only |
