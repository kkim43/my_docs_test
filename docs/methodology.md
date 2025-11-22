# Methodology

# 1. System Architecture Overview

## 1.1 High-Level Architecture
![High-Level Architecture](./Sparxiv_SysArchitecture_Simple.webp)

## 1.2 Detailed System Architecture
![Detailed System Architecture](./Sparxiv_SysArchitecture.webp)

---

# 2. Data Ingestion & Preprocessing (Structured APIs)

## 2.1 Ingestion Workflow
```python
df_raw = spark.read.json(input_path)
```

## 2.2 Transformations
```python
df = df.withColumn("title_clean", clean_text(F.col("title")))
```

## 2.3 Storage Format
```python
df.write.mode("overwrite").parquet(out_path)
```

---

# 3. Spark SQL Analytics

Below are the full-analysis outputs.

### Abstract Length vs Versions (Deciles)
![Abstract Length Decile](../reports/analysis_full/complex_abstractlen_versions_by_decile.png)

### Abstract Length vs Versions (Correlation)
![Correlation](../reports/analysis_full/complex_abstractlen_versions_correlation.png)

### Author Category Migration
![Author Category Migration](../reports/analysis_full/complex_author_category_migration_top20.png)

### Author Collaboration Over Time
![Author Collaboration](../reports/analysis_full/complex_author_collab_over_time_simple.png)

### Author Lifecycle
![Author Lifecycle](../reports/analysis_full/complex_author_lifecycle_scatter.png)

### Avg Token Count by Year
![Avg Token Count](../reports/analysis_full/complex_avg_token_count_by_year.png)

### Category Cooccurrence
![Category Cooccurrence](../reports/analysis_full/complex_category_cooccurrence_top.png)

### Category Versions Avg
![Category Versions](../reports/analysis_full/complex_category_versions_avg_top30.png)

### Declining Topics
![Declining Topics](../reports/analysis_full/complex_declining_topics_top20.png)

### DOI Versions Correlation
![DOI Versions](../reports/analysis_full/complex_doi_versions_correlation.png)

### DOI vs Versions Group
![DOI vs Versions](../reports/analysis_full/complex_doi_vs_versions_group.png)

### Lexical Richness by Year
![Lexical Richness](../reports/analysis_full/complex_lexical_richness_by_year.png)

### Rising Topics
![Rising Topics](../reports/analysis_full/complex_rising_topics_top20.png)

---

# 4. Standard Queries (Full Dataset)

### Abstract Length Histogram
![Abstract Length Hist](../reports/standard_queries_full/abstract_length_hist.png)

### Category Pareto
![Category Pareto](../reports/standard_queries_full/category_pareto.png)

### Category-Year Heatmap
![Category-Year Matrix](../reports/standard_queries_full/heatmap_category_year.png)

### DOI Rate by Year
![DOI Rate](../reports/standard_queries_full/doi_rate_by_year.png)

### Papers per Year
![Papers per Year](../reports/standard_queries_full/papers_per_year.png)

### Top Authors
![Top Authors](../reports/standard_queries_full/top_authors.png)

### Top Categories
![Top Categories](../reports/standard_queries_full/top_categories.png)

### Version Count Histogram
![Version Count](../reports/standard_queries_full/version_count_hist.png)

---

# 5. ML Pipeline

```python
pipeline = Pipeline(stages=[tokenizer, stopwords, vectorizer, idf, normalizer])
```

---

# 6. Similarity Search

### Sample Mode
```python
score = dot(q.toArray(), v.toArray())
```

### Full Mode (CSR)
```python
scores = csr_matrix @ query_dense
```

---

# 7. Streaming

```python
stream_df = spark.readStream.schema(schema).json(stream_path)
```

Example:
![Streaming Example](../reports/streaming_sample/20251212/papers_per_year.png)
