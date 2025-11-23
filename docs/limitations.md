# Limitations (Condensed Version)

This system demonstrates a complete Spark-based recommender pipeline, but several structural limitations remain due to the nature of arXiv metadata and the constraints of scalable, single-machine execution.

---

## 1. Dataset-Level Limitations

### 1.1 Metadata-only text (no full paper content)
The system relies solely on titles, abstracts, and categories because the dataset does not include PDF full text.  
This limits semantic depth and restricts the recommender to high-level lexical similarity rather than full scientific understanding.

### 1.2 Inconsistent or incomplete metadata
Some records contain missing authors, categories, or malformed abstracts (LaTeX artifacts), which can distort category trends, co-occurrence graphs, and author-based analytics.

---

## 2. Methodological Limitations

### 2.1 TF-IDF is non-semantic
TF-IDF cannot model synonyms, context, or scientific terminology relationships.  
As a result, similarity is strictly vocabulary-driven and may miss deeper conceptual connections.

### 2.2 Exact cosine similarity is computationally expensive
Even with CSR acceleration, exact Top-K similarity scales linearly with dataset size and does not use Approximate Nearest Neighbor (ANN) indexing.  
This approach is suitable for demonstration but not for large-scale real-time deployment.

---

## 3. Pipeline & System Limitations

### 3.1 Single-machine resource constraints
Training TF-IDF and running complex SQL analyses on the full arXiv dataset requires significant memory and disk spill space.  
These limitations reflect hardware constraints rather than Spark’s capability on distributed clusters.

### 3.2 Streaming is a controlled simulation
The streaming workflow processes synthetic weekly batches rather than live external data, which limits realism but satisfies the course requirement to demonstrate Structured Streaming.

---

## 4. Evaluation Limitations

### 4.1 No quantitative evaluation
The system does not include precision/recall, user feedback loops, or comparison to embedding-based baselines.  
Similarity quality is therefore demonstrated only through case studies, not measurable benchmarks.

---

## Summary
These limitations arise from dataset constraints, the lexical modeling choice, and the single-machine execution environment.  
Despite them, the system fulfills the intended purpose: demonstrating an end-to-end Spark pipeline combining ingestion, preprocessing, analytics, MLlib, streaming, and search at scale.
