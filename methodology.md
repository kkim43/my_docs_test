# Sparxiv Methodology
## Cloud Computing for Data Analysis — ITCS 6190/8190, Fall 2025

Sparxiv is a large-scale scientific document analysis and recommendation system built using **Apache Spark**.  
The project integrates **Structured APIs, Spark SQL, Streaming, and MLlib**, satisfying all requirements of the ITCS 6190/8190 course project.  
It processes arXiv metadata, builds TF-IDF representations, generates analytical output, and supports efficient similarity-based search through both in-memory and CSR-based indices.

---

# 1. System Architecture Overview

## 1.1 High-Level Architecture

![High-Level Architecture](./Sparxiv_SysArchitecture_Simple.webp)

The system is divided into four major layers:

- **Ingestion Layer** – raw JSON → Spark DataFrame → cleaned Parquet  
- **Analytics Layer** – Spark SQL for complex analytical modules  
- **ML Layer** – TF-IDF modeling using Spark MLlib  
- **Search Layer** – similarity search via in-memory & CSR-based indices  
- **Streaming Layer** – Structured Streaming processing simulated weekly updates  
- **Application Layer** – lightweight search and analytics interface

---

## 1.2 Detailed System Architecture

![Detailed System Architecture](./Sparxiv_SysArchitecture.webp)

This diagram expands the core pipeline:

- Shared SparkSession  
- Batch and streaming transformations via `transform_all()`  
- ML training pipeline  
- Offline CSR index builder  
- Flask-based UI  
- Result reporting infrastructure  

---

# 2. Data Ingestion & Preprocessing (Structured APIs)

## 2.1 Ingestion Workflow

Raw arXiv metadata (JSON/JSONL) is loaded with Spark DataFrame APIs:

```python
spark.read.json(path)
```

The ingestion stage performs:

- Schema inference  
- Multi-line JSON support  
- Extraction of relevant fields  
- Initial cleanup and normalization  

---

## 2.2 Preprocessing & Transformations

Transformations include:

- **Text Normalization:** lowercase, punctuation cleanup  
- **Category Extraction:** primary + category list  
- **Author Normalization**  
- **Temporal Parsing:** derive year  
- **Feature Augmentation:** abstract length, author count, DOI flag  
- **Filtering of low-quality entries**

---

## 2.3 Storage Format

Cleaned outputs stored as:

- **Parquet**, partitioned by `year`  
- **Zstandard (ZSTD)** compression  

---

# 3. Exploratory Analysis & Spark SQL Layer

Analytical modules use Spark SQL for:

- Category co-occurrence  
- Author collaboration  
- Topic rise/decline  
- Lexical richness  
- DOI/version analysis  
- Abstract length patterns  
- Category stability  

Spark SQL is used for clarity, reproducibility, and integration with the execution engine.

---

# 4. Machine Learning Pipeline (Spark MLlib TF-IDF)

## 4.1 Pipeline Stages

- RegexTokenizer  
- StopWordsRemover  
- CountVectorizer  
- IDF  
- L2 Normalizer  

---

## 4.2 Rationale for TF-IDF

TF-IDF is:

- Deterministic  
- Interpretable  
- CPU-friendly  
- Fully supported by MLlib  
- Highly reproducible  

---

## 4.3 Parameter Choices

Vocabulary size, min_df, and stopword policies are dataset-driven and tailored to sample vs full pipelines.

---

# 5. Similarity Search Methodology

## 5.1 Query Vectorization

Query text is transformed via the same MLlib pipeline used during training to ensure consistency.

---

## 5.2 Sample Mode (In-Memory Search)

- Load all TF-IDF vectors into memory  
- Compute cosine similarity via sparse dot product  
- No Spark execution at inference time  

---

## 5.3 Full Mode (CSR-Based Offline Index)

### Offline Phase
1. Read TF-IDF features via PyArrow  
2. Reconstruct SparseVectors  
3. Build global CSR matrix  
4. Store metadata arrays  

### Query Phase
1. Spark vectorizes query  
2. Convert to dense Python vector  
3. CSR matrix–vector multiply  
4. Select Top-K  

Designed to replace expensive Spark cross-joins and ensure scalable, deterministic search.

---

# 6. Structured Streaming Methodology

Streaming pipeline:

1. Watches directory for new JSONL files  
2. Applies preprocessing with `transform_all()`  
3. Computes:
   - Yearly counts  
   - Category stats  
   - DOI rates  
4. Writes results per micro-batch  

Structured Streaming ensures real-time ingestion consistency.

---

# 7. End-to-End Pipeline Integration

Pipeline stages:

1. Ingestion  
2. Transformation  
3. TF-IDF training  
4. SQL analytics  
5. Streaming (optional)  
6. CSR index building  
7. Web-based search  

Designed for reproducibility with a single-command execution approach.

---

# 8. Limitations

- TF-IDF lacks deep semantic modeling  
- CSR index requires memory proportional to dataset size  
- Streaming pipeline is stateless  
- Version history not deeply modeled  
- Scientific vocabulary may dominate token statistics  

---

# 9. Summary

Sparxiv demonstrates a complete Spark-based workflow:

- **Structured APIs** for ETL  
- **Spark SQL** for analysis  
- **MLlib** TF-IDF modeling  
- **Structured Streaming** for real-time processing  
- **Optimized similarity search** with CSR and in-memory indices  

It highlights Spark’s unified engine for large-scale scientific text analytics.

