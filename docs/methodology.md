# Methodology

# 1. System Architecture Overview

## 1.1 High-Level Architecture

![High-Level Architecture](../docs/Sparxiv_SysArchitecture_Simple.webp)

---

## 1.2 Detailed System Architecture

![Detailed System Architecture](../docs/Sparxiv_SysArchitecture.webp)

---

# 2. Data Ingestion & Preprocessing (Structured APIs)

## 2.1 Ingestion Workflow

Raw arXiv metadata (JSON/JSONL) is loaded with Spark DataFrame APIs:

```python
df_raw = spark.read.json(input_path)
```

SparkSession is created using a tuned helper:

```python
spark = get_spark("ccda_project")
```

The ingestion stage performs:

- Schema inference  
- Multi-line JSON handling  
- Extraction of relevant fields  
- Initial cleanup and normalization  

---

## 2.2 Preprocessing & Transformations

Representative transformation:

```python
df = df.withColumn("title_clean", clean_text(F.col("title")))
```

Transformations include:

- **Text Normalization:** lowercase, punctuation cleanup  
- **Category Extraction:** primary + category list  
- **Author Normalization**  
- **Temporal Parsing:** derive year  
- **Feature Augmentation:** abstract length, author count, DOI flag  
- **Filtering of low-quality entries**

---

## 2.3 Storage Format

Outputs are stored as:

- **Parquet**, partitioned by `year`  
- **Zstandard (ZSTD)** compression  

```python
df.write.mode("overwrite").parquet(out_path)
```

---

# 3. Exploratory Analysis & Spark SQL Layer

Representative SQL computation:

```python
df.createOrReplaceTempView("papers")
spark.sql("SELECT year, COUNT(*) FROM papers GROUP BY year")
```

Analytical modules include:

- Category co-occurrence  
- Author collaboration  
- Topic rise/decline  
- Lexical richness  
- DOI/version analysis  
- Abstract length trends  
- Category stability  

Example figure:

![Category Co-occurrence](../reports/analysis_full/complex_category_cooccurrence_top.png)

---

# 4. Machine Learning Pipeline (Spark MLlib TF-IDF)

## 4.1 Pipeline Stages

```python
pipeline = Pipeline(stages=[tokenizer, stopwords, vectorizer, idf, normalizer])
```

Stages include:

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

Vocabulary size and stopword policies are dataset-dependent.

Example training snippet:

```python
model = pipeline.fit(df)
```

---

# 5. Similarity Search Methodology

## 5.1 Query Vectorization

```python
query_vec = pipeline_model.transform(query_df).select("features")
```

---

## 5.2 Sample Mode (In-Memory Search)

Representative similarity computation:

```python
score = dot(q.toArray(), v.toArray())
```

- Load TF-IDF vectors into memory  
- Compute cosine similarity in Python  
- No Spark during inference  

---

## 5.3 Full Mode (CSR-Based Offline Index)

### Offline Phase

```python
csr_data.append((indices, values))
```

Steps:

- Read Parquet via PyArrow  
- Reconstruct SparseVectors  
- Build CSR matrix  
- Store metadata arrays  

### Query Phase

```python
scores = csr_matrix @ query_dense
```

Fast inference using matrix–vector multiplication.

---

# 6. Structured Streaming Methodology

```python
stream_df = spark.readStream.schema(schema).json(stream_path)
```

Workflow:

1. Watch directory for new JSONL  
2. Apply `transform_all()`  
3. Compute stats (year, categories, DOI)  
4. Write per-microbatch results  

Example visualization:

![Papers per Year](../reports/streaming_sample/20251212/papers_per_year.png)

---

# 7. End-to-End Pipeline Integration

Executed with:

```bash
bash run.sh
```

Pipeline includes:

1. Ingestion  
2. Transformation  
3. TF-IDF training  
4. SQL analytics  
5. Streaming  
6. CSR index building  
7. Web search interface  

---

# 8. Summary

Sparxiv implements:

- **Structured APIs** for ingestion  
- **Spark SQL** for analytics  
- **MLlib TF-IDF** for modeling  
- **Structured Streaming** for real-time ingestion  
- **CSR & in-memory indices** for efficient search  

This version adds representative code snippets directly tied to the actual implementation.
