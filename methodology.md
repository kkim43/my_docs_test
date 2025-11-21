# Methodology

This document explains the full methodology underlying **Sparxiv**, an end‑to‑end scalable ingestion, transformation, feature engineering, search, and analytics pipeline built on the arXiv metadata corpus.  
All components described here come directly from the project’s actual codebase (`engine/`, `pipelines/`, `streaming/`) and reflect the real execution flow.

---

# 1. System Architecture Overview

Sparxiv follows a classic multi‑stage data/ML pipeline:

1. **Raw Data → Ingestion → Parquet**
   - `engine/data/ingestion.py`
   - `engine/data/transformations.py`
   - Reads Kaggle JSON/JSONL
   - Cleans, normalizes, filters, and extracts metadata
   - Writes partitioned Parquet

2. **Parquet → TF‑IDF Model Training**
   - `engine/ml/train.py`
   - `engine/ml/featurization.py`
   - Produces:
     - Spark ML PipelineModel (`data/models/...`)
     - Normalized feature vectors (`data/processed/features_...`)

3. **Search**
   - `engine/search/vectorize.py`
   - `engine/search/similarity.py`
   - `engine/search/search_engine.py`
   - Vectorizes query → computes exact cosine similarity → returns top‑K

4. **Complex Analytics**
   - `engine/complex/complex_queries.py`
   - 10 advanced SQL/Spark analyses (topic dynamics, DOI trends, lexical richness, etc.)

5. **Streaming Analytics (Optional)**
   - `streaming/sample_stream.py`
   - `streaming/full_stream.py`
   - Auto‑process incoming daily/weekly JSON drops

---

# 2. Ingestion Pipeline

## 2.1 Reading Raw JSON
```
read_arxiv_json → spark.read.json
```
Supports JSON / JSONL (`multiLine=true|false`).

## 2.2 `transform_all()` Overview  
Found in `engine/data/transformations.py`.

### Major transformations performed:
| Task | Code |
|------|------|
| Clean text (title/abstract) | `clean_text()` |
| Lowercase + normalize | `lower()` |
| Extract primary category | `extract_primary_category()` |
| Split categories → array | `split_categories()` |
| Generate year | `parse_year_from_datestr()` |
| Normalize authors | `normalize_authors()` |
| Compute lengths | `abstract_len`, `title_len` |
| Extract DOI presence | `has_doi` |
| Filter quality | abstract length ≥ 40 |

### Sample output schema (simplified)

```
arxiv_id
title_clean
abstract_clean
primary_category
category_list (array)
authors_list (array)
year
abstract_len
title_len
n_authors
n_categories
has_doi
versions
...
```

## 2.3 Partitioning & Output
Ingestion writes:

```
data/processed/arxiv_full/
data/processed/arxiv_sample/
```

Partitioning can be:
- by year (default)
- by primary_category
- or none

Compression: **Zstandard**, optimized block/page sizes, max 50k records per file.

---

# 3. Feature Engineering (TF‑IDF Pipeline)

Defined in `engine/ml/featurization.py`.

## 3.1 Pipeline Components
1. **RegexTokenizer**
2. **StopWordsRemover**
   - default English stopwords
   - domain stopwords (arxiv, propose, results…)
   - optional extra stopwords (most frequent tokens)
3. **Optional bigrams**
4. **CountVectorizer**
5. **IDF**
6. **Normalizer (L2)**

Output column:
```
features_norm : SparseVector
```

## 3.2 Training Settings

### Sample model (`pipelines/train_sample.py`)
```
vocab_size=80000
min_df=3
use_bigrams=False
extra_stopwords_topdf=200
```

### Full model (`pipelines/train_full.py`)
```
vocab_size=120000
min_df=10
use_bigrams=False
extra_stopwords_topdf=0   # disabled (heavy)
```

---

# 4. Query Vectorization & Search

## 4.1 Query Vectorization
`engine/search/vectorize.py`

Steps:
1. Combine title + abstract
2. Run through the same TF‑IDF pipeline
3. Produce a single L2‑normalized SparseVector

## 4.2 Exact Cosine Similarity
From `engine/search/similarity.py`.

```
score = dot(query_vector, candidate_vector)
```

Implements:
- brute‑force crossJoin
- Fast UDF dot-product on SparseVector
- Window partition with row_number() for Top‑K

## 4.3 High‑Level Search API
`engine/search/search_engine.py`

```
SearchEngine(mode="sample"|"full")
  -> vectorize query
  -> compute topk neighbors
  -> join metadata (title, abstract, year, categories)
  -> return ranked list
```

---

# 5. Complex Analytics Layer (10 Modules)

Found in `engine/complex/complex_queries.py`.

### Complete list:
1. Category co‑occurrence  
2. Author collaboration over time  
3. Rising & declining topics  
4. Lexical richness trends  
5. DOI vs versions correlation  
6. Author productivity lifecycle  
7. Author category migration  
8. Abstract length vs popularity  
9. Weekday submission patterns  
10. Category stability via versions  

Each produces:
- CSV summaries
- Matplotlib PNG charts  
Output saved under:

```
reports/analysis_full/
reports/analysis_sample/
```

---

# 6. Streaming Workflow (Optional)

`streaming/sample_stream.py` & `full_stream.py`:

- Watches incoming `arxiv-*.jsonl`
- Performs:
  - transform_all()
  - yearly counts
  - category distribution
  - DOI rate trends
- Saves incremental reports to:

```
reports/streaming_sample/YYYYMMDD/
```

---

# 7. End‑to‑End Pipeline (run.sh)

```
1. Download dataset (if missing)
2. Ingest full dataset
3. Train TF-IDF model
4. Run complex analytics
5. (Optional) Run streaming processors
```

Outputs:

```
data/processed/arxiv_full/
data/models/tfidf_full/
data/processed/features_full/
reports/analysis_full/
```

---

# 8. Key Design Choices

### ✔ Spark-based fully distributed pipeline  
Allows scaling to 1.7M+ papers, suitable for full arXiv.

### ✔ Pure TF‑IDF over heavy embedding models  
Chosen for:
- transparency  
- speed  
- ability to run on CPU clusters  
- interpretable vector space

### ✔ Exact cosine over approximate ANN  
Provides deterministic reproducible results for academic evaluation.

### ✔ Clean separation of ingestion, ML, search, analysis  
Each layer is independently testable.

---

# 9. Limitations

1. **Brute‑force search** does not scale to billions of vectors.  
2. **TF‑IDF semantic limits** (no embeddings, no semantic similarity).  
3. Ingestion depends on Kaggle's snapshot schema—changes upstream may break parsers.  
4. Some metadata fields (e.g., version history) are not deeply exploited.

---

# 10. Summary

Sparxiv implements a complete Spark‑powered pipeline:

- cleans & standardizes raw arXiv JSON  
- transforms it into structured Parquet  
- trains TF‑IDF features  
- enables fast exact cosine retrieval  
- performs deep SQL/analytics over the metadata  
- can operate in streaming or batch mode  

This methodology reflects the **actual implementation** of the system with no hypothetical components or missing steps.

