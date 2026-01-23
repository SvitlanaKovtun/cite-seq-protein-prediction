# cite-seq-protein-prediction

This project predicts surface protein expression from single-cell RNA-seq data (CITE-seq).  
It implements a global modeling pipeline that combines biological priors, dimensionality reduction, and donor-aware validation to achieve robust generalization across heterogeneous cell populations.

---

## Biological Context

CITE-seq (Cellular Indexing of Transcriptomes and Epitopes by sequencing) simultaneously measures gene expression (RNA-seq) and surface protein abundance in individual cells using antibody-derived tags. While RNA and protein levels are related, the relationship is indirect and noisy due to post-transcriptional regulation, protein turnover, and technical effects.

Several properties of CITE-seq data make RNA → protein prediction non-trivial:

- **Protein-specific signal strength**  
  Some proteins show strong correspondence with their transcripts, while others are weakly or inconsistently correlated. Targets differ widely in variance, mean expression, and predictability.

- **High-dimensional transcriptomic input**  
  Each cell is characterized by thousands of genes, many of which are irrelevant for a given surface protein and primarily contribute noise.

- **Cellular heterogeneity**  
  The dataset contains multiple hematopoietic cell types. Although biological intuition suggests cell-type–specific regulation, splitting the data by cell type substantially reduces sample size and increases variance.

- **Donor effects**  
  Cells originate from multiple donors. Donor-specific expression shifts can inflate performance if not explicitly controlled, leading to models that fail to generalize.

---

## Pipeline Overview

**Goal:** predict CITE-seq surface protein abundance from single-cell RNA-seq data while remaining robust to donor effects and high dimensionality.

---

### 1. Input data

- Single-cell RNA-seq gene expression levels  
  - library-size normalized  
  - log1p transformed  
- Surface protein measurements for the same cells  
  - dsb normalized  
- Cell metadata  
  - donor  
  - cell type  
  - day retained as a proxy for biological differentiation state, distinct from technical batch effects.  

---

### 2. Feature construction

- **RNA-derived features**
  - Target-specific relevant genes (BioLegend TotalSeq annotations), kept uncompressed
  - Remaining transcriptome standardized and compressed using PCA (used here as a regularization and compression step for machine-learning models, not for biological interpretation. RNA features are standardized prior to PCA to avoid dominance by high-variance genes and to improve stability and generalization across donors).

  - Final RNA feature block:  
    `[relevant genes] + [PCA components]`

- **Metadata features**
  - `cell_type` encoded as one-hot dummy variables (`ct_*`)
  - `day` included as a numeric covariate

- **Leakage control**
  - `donor` used exclusively for donor-held-out splitting
  - `donor` removed from the feature matrix before model training

---

### 3. Modeling

- Single **global XGBoost regression model** configured with deep trees (depth 6) to capture non-linear interactions, balanced by strong L2 regularization (lambda 15) to reject donor-specific noise.
- All cell types are merged after empirical evaluation showed that cell-type–specific models underperformed
- Multi-protein prediction handled explicitly

---

### 4. Hyperparameter tuning

- A subset of **representative protein targets** is selected:
  - stratified by difficulty to predict (hard / easy) based on column-wise Pearson correlation,
  - and signal strength (low / high variance)
  - Manual biological overrides: Specific targets (e.g., Isotype controls, rare cell markers) forced into the tuning set to prevent overfitting to dominant cell types.

- Hyperparameters are tuned using **donor-held-out validation**
  - one donor fully excluded from training
  - parameters selected based on performance on the unseen donor

---

### 5. Evaluation

- Metric: **mean cell-wise Pearson correlation**
- Correlation computed per cell across all predicted proteins
- Designed to assess recovery of relative protein expression profiles per cell

---

### Final training run (refit on all labeled data)

After selecting hyperparameters using donor-held-out validation, a final model is trained on **all available labeled cells**:


The refit model is then used to generate predictions for the evaluation/test set.

---

### 6. Design principles

- Evidence-driven modeling decisions
- Use of biological priors where available
- Regularization through dimensionality reduction
- Explicit control of donor leakage
- Memory-efficient execution
