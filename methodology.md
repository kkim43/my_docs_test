# Methodology

This document describes the end-to-end methodology used to build **Sparxiv**, a scalable Spark-based recommender and analytics system for the arXiv metadata dataset.  
The goal is to provide a clear explanation of the architectural design, data processing workflow, model training strategy, analytics pipeline, streaming simulation, and system integration.

---

# 1. System Architecture Overview

Sparxiv consists of five major components:

1. **Batch Ingestion Pipeline**  
   Converts raw JSONL metadata into cleaned and partitioned Parquet.

2. **Feature Engineering & Model Training**  
   Generates TF-IDF vectors using Spark ML for paper similarity search.

3. **Complex Analytics Engine**  
   Executes domain-specific Spark SQL analyses (categories, DOI, authors, trends).

4. **Streaming Simulation**  
   Processes synthetic weekly metadata drops using Structured Streaming.

5. **Web Application Layer**  
   Provides a live search UI and analytics browser via Flask + Spark backend.

All components operate on top of a shared SparkSession optimized for local execution.

---

# 2. Batch Ingestion Pipeline

The ingestion stage processes the raw Kaggle arXiv JSONL file into a clean analytical dataset.

### 2.1 Input
- `arxiv-metadata-oai-snapshot.json` (full mode)
- `arxiv-sample.jsonl` generated head-N sample (sample mode)

### 2.2 Processing Steps

1. **JSONL Parsing**  
   Loads records as Spark DataFrames using schema inference.

2. **Field Normalization**
   - Standardize whitespace  
   - Remove problematic LaTeX markup patterns  
   - Lowercase categories  
   - Extract primary category (first token)

3. **Filtering**
   - Remove papers with empty or extremely short abstracts  
   - Drop malformed or incomplete entries  
   - Optional: enforce valid timestamps where present

4. **Date & Version Handling**
   - Convert `update_date` to Spark `date` type  
   - Extract year used for partitioning & analytics  

5. **Output Structure**
   Writes output Parquet with:
   - `id_base`, `paper_id`
   - `title`
   - `abstract`
   - `categories`
   - `primary_category`
   - `year`
   - `authors`, `doi`, etc.

### 2.3 Output
- Sample: `data/processed/arxiv_sample/`
- Full: `data/processed/arxiv_full/`

The ingestion pipeline ensures a consistent and clean foundation for downstream feature extraction and analytics.

---

# 3. Feature Engineering & Model Training

We use Spark ML to build **TF-IDF vector representations** for abstracts.

### 3.1 Tokenization
- `RegexTokenizer` with punctuation removal  
- Lowercasing and minimal normalization  
- No stemming or lemmatization (for scalability)

### 3.2 Stopword Filtering
Two-level strategy:
- Spark’s default English stopwords  
- *Extra high-frequency domain tokens* automatically removed based on document frequency (`extra_stopwords_topdf`)

### 3.3 Vectorization
- **Term Frequency** using `CountVectorizer`  
- **Inverse Document Frequency** using `IDF`
- Output vectors stored as **Spark ML Sparse Vectors**

### 3.4 Normalization
- L2 normalization applied for cosine similarity

### 3.5 Training Configurations

| Mode | Vocab Size | min_df | Extra Stopwords | Notes |
|------|-------------|--------|------------------|--------|
| Sample | 80k | 3 | 200 | Fast experimentation |
| Full | 250k | 5 | 500 | High-quality model |

### 3.6 Outputs
- Trained model: `data/models/tfidf_{sample,full}/`  
- Feature parquet: `data/processed/features_{sample,full}/`

These vectors serve as the foundation for the similarity search engine and are also used to enrich analytics.

---

# 4. Paper Similarity Search Engine

The similarity engine is built using:

- Precomputed TF-IDF sparse vectors  
- Exact cosine similarity  
- Distributed Spark operations

### 4.1 Query Flow
1. User enters text or selects an existing paper  
2. Query text → vectorized using trained TF-IDF model  
3. Compute cosine similarity against all vectors  
4. Return Top-K results with:
   - arXiv ID  
   - Title  
   - Categories  
   - Year  
   - Similarity score  

### 4.2 Design Decision
We use **exact** similarity rather than ANN to ensure:
- Deterministic results  
- Simple Spark implementation  
- Reproducible evaluation  

---

# 5. Complex Analytics

We implement 10 domain-specific Spark SQL & DataFrame analyses.  
Each analysis builds an aggregated CSV + visualization (PNG) stored under `reports/`.

### Major analyses include:
- **Category co-occurrence matrix**  
- **Temporal category trends**  
- **Author collaboration patterns**  
- **DOI coverage over time**  
- **Abstract length & readability patterns**  
- **Submission weekday distributions**  
- **Version stability and category migration**

These analyses demonstrate the use of Spark SQL, window functions, and aggregations at scale.

---

# 6. Streaming Pipeline (Structured Streaming)

The streaming module simulates weekly arXiv updates.

### 6.1 Batch Generator (Sample Mode)
`sample_prepare_batches.py` creates synthetic weekly JSONL drops:
- Each drop includes increasingly larger slices (10k → 20k → …)
- File naming pattern:  
  `arxiv-sample-YYYYMMDD.jsonl`

### 6.2 Streaming Engine
`sample_stream.py` monitors the incoming directory and processes each new batch:

1. Read new JSONL drop  
2. Apply same transformations as batch ingestion  
3. Compute:
   - Papers per year  
   - Top categories  
   - DOI rate over time  
4. Write results to:  
   `reports/streaming_sample/YYYYMMDD/`

### 6.3 Goals
- Demonstrate incremental ingestion  
- Produce real-time analytics  
- Validate pipeline stability under continuous updates  

---

# 7. Web Application Layer

A lightweight Flask app integrates search and analytics browsing.

### 7.1 SparkSession Integration
The backend uses a shared SparkSession with:
- Tuned shuffle partitions  
- AQE enabled  
- Local disk spill optimization  

### 7.2 Features
- **Real-time Search UI**  
  - Free text query  
  - Top-K parameter  
  - Sample/full model selection  

- **Analytics Browser**  
  - Loads CSV reports  
  - Renders HTML tables with pagination  

### 7.3 Services
- `search_service.py`: wraps cosine similarity engine  
- `filters_service.py`: list categories and metadata  
- `complex_service.py`: load analysis reports  

---

# 8. Summary of Methodological Choices

| Component | Methodological Choice | Reason |
|-----------|------------------------|--------|
| TF-IDF | Classical sparse vectors | Scalable, reproducible, Spark-friendly |
| No embeddings | Avoid heavy GPU models | Course environment & resource limits |
| Exact cosine | Deterministic Top-K | Transparency and correctness |
| Spark SQL analytics | Distributed DataFrame engine | Handles millions of records |
| Streaming simulation | Weekly-dated drops | Demonstrates real-time architecture |
| Flask UI | Lightweight UI layer | Simple integration over Spark backend |

Overall, the methodology prioritizes **scalability**, **clarity**, **reproducibility**, and **Spark-native design**, while demonstrating end-to-end data engineering and model serving in a unified system.
