# Universal MCP Skills Reference

This document provides a comprehensive technical reference for the 9 core bioinformatics tools registered on the `embpy` Model Context Protocol (MCP) server.

All tools adhere to strict **Token-Saving Guardrails**:
- Large arrays stay on disk in `.h5ad` AnnData files or memory caches.
- Tools return compact JSON summaries with array shapes, counts, and Markdown tables.
- Exceptions return descriptive diagnostic strings to enable autonomous self-correction by the AI agent.

---

## Tool Summary Table

| Tool Name | Category | Primary Function | Token-Saving Return |
| :--- | :--- | :--- | :--- |
| [`inspect_data`](#1-inspect_data) | Preprocessing | Inspects `.h5ad` structure, layers, and candidate keys | Shape `(n_obs, n_vars)`, candidate keys, layer names |
| [`resolve_identifiers`](#2-resolve_identifiers) | Preprocessing | Harmonizes IDs (Ensembl, HGNC, SMILES) | Mapped percentage, sample pairs |
| [`preprocess_counts`](#3-preprocess_counts) | Preprocessing | Normalizes (`normalize_total`, `log1p`), selects HVGs | Output path, cell/gene counts, HVG count |
| [`list_model_registry`](#4-list_model_registry) | BioEmbedder | Catalogs ~150 foundation models and hardware | Hardware device, model rosters by modality |
| [`compute_embeddings`](#5-compute_embeddings) | BioEmbedder | Extracts multi-model latent representations | Output path, embedding keys, array shapes |
| [`annotate_metadata`](#6-annotate_metadata) | Annotation | Enriches `.obs`/`.uns` with GO, pathways, disease data | Added column names, entity count |
| [`compute_umap`](#7-compute_umap) | Analysis | Computes kNN graph and 2D UMAP coordinates | Coordinate array shape `(n_obs, 2)`, umap key |
| [`benchmark_scib`](#8-benchmark_scib) | Benchmarking | Evaluates bio-conservation & batch integration | Formatted Markdown table of NMI, ARI, ASW, cLISI |
| [`benchmark_perturbations`](#9-benchmark_perturbations) | Benchmarking | Scores differential perturbation predictions | Formatted Markdown table of cell-eval metrics |

---

## 1. `inspect_data`

Profiles an AnnData `.h5ad` file or a list of items to inspect metadata, observation columns, embeddings, layers, and count distributions.

### Parameters
- `path` (*str*, required): Filesystem path to the `.h5ad` AnnData dataset.
- `max_categories` (*int*, default=10): Maximum categorical levels to list per column.
- `sample_peek` (*int*, default=5): Number of sample observation and feature names to return.

### Response Schema
```json
{
  "status": "success",
  "path": "data/pbmc3k.h5ad",
  "n_cells": 2700,
  "n_genes": 32738,
  "is_log_normalized": false,
  "obs_columns": ["n_genes", "percent_mito", "cell_type", "batch"],
  "candidate_label_columns": ["cell_type"],
  "candidate_batch_columns": ["batch"],
  "obsm_keys": ["X_pca"],
  "layers": ["counts"],
  "sample_genes": ["CD3D", "MS4A1", "GNLY", "CD14", "FCER1A"]
}
```

---

## 2. `resolve_identifiers`

Harmonizes biological entity vocabularies (Ensembl IDs <-> HGNC symbols, molecule names <-> SMILES, UniProt <-> proteins) to prevent vocabulary mismatches in downstream models.

### Parameters
- `identifiers` (*list[str] | str*, optional): List or comma-separated string of identifiers to map.
- `path` (*str*, optional): Path to AnnData `.h5ad` whose `var_names` or `.obs` column to resolve.
- `entity_type` (*str*, default="auto"): Entity class (`"gene"`, `"molecule"`, `"protein"`, `"cell_line"`).
- `target_id_type` (*str*, default="auto"): Target ID system (`"ensembl"`, `"symbol"`, `"smiles"`).
- `organism` (*str*, default="human"): Species for resolution (`"human"`, `"mouse"`).
- `update_h5ad_path` (*str*, optional): If provided, writes the updated AnnData with mapped IDs to disk.

### Response Schema
```json
{
  "status": "success",
  "entity_type": "gene",
  "total_submitted": 2000,
  "mapped_count": 1984,
  "mapping_success_rate": 99.2,
  "sample_mapping": {
    "TP53": "ENSG00000141510",
    "GAPDH": "ENSG00000111640"
  },
  "output_path": "/data/pbmc_resolved.h5ad"
}
```

---

## 3. `preprocess_counts`

Runs standard single-cell quality control, library size normalization (`sc.pp.normalize_total`), log1p transformation (`sc.pp.log1p`), and highly variable gene (HVG) selection. Preserves raw count matrices in `.layers['counts']` and normalized expression in `.layers['log_normalized']`.

### Parameters
- `path` (*str*, required): Path to the raw `.h5ad` AnnData dataset.
- `output_path` (*str*, optional): Destination path for the preprocessed dataset.
- `preprocessing` (*{'auto', 'standard', 'none'}*, default="standard"): Preprocessing strategy:
  - `"auto"`: Dynamically assesses dataset scale; scales down QC thresholds and HVG counts if dataset is small to prevent cell starvation.
  - `"standard"`: Enforces strict filtering cutoffs (`min_genes=200`, `min_cells=3`), applies `normalize_total` to 1e4, computes `log1p`, and selects `n_top_genes` HVGs.
  - `"none"`: Bypasses filtering and transformations; preserves raw counts in `.layers['counts']` for models requiring integer counts (e.g., scVI).
- `min_genes` (*int*, default=200): Minimum genes per cell.
- `min_cells` (*int*, default=3): Minimum cells expressing a gene.
- `target_sum` (*float*, default=1e4): Target library count per cell.
- `n_top_genes` (*int*, default=2000): Number of highly variable genes to select.
- `select_hvg` (*bool*, default=true): Whether to compute HVGs.
- `log_transform` (*bool*, default=true): Whether to apply `sc.pp.log1p`.
- `scale` (*bool*, default=false): Whether to scale expression to unit variance (stored in `.layers['scaled']`).
- `auto_adjust` (*bool*, default=false): Whether to automatically adapt cutoffs for small datasets.

### Response Schema
```json
{
  "status": "success",
  "input_path": "/data/raw.h5ad",
  "output_path": "/data/preprocessed.h5ad",
  "n_cells": 2650,
  "n_genes": 18200,
  "n_hvg": 2000,
  "layers": ["counts", "log_normalized"],
  "summary": "Successfully preprocessed 2650 cells and 18200 genes."
}
```

---

## 4. `list_model_registry`

Reports available biological foundation models (~150), supported modalities, and active hardware acceleration.

### Parameters
- `entity_type` (*str*, optional): Biological modality filter (`"gene"`, `"protein"`, `"molecule"`, `"dna"`, `"single_cell"`, `"morphology"`, `"text"`, `"perturbation"`).

### Response Schema
```json
{
  "status": "success",
  "total_models_available": 135,
  "compute_device": "cuda",
  "device_details": "NVIDIA GPU (Tesla V100-SXM2-16GB)",
  "supported_entities": [
    "gene",
    "protein",
    "molecule",
    "dna",
    "single_cell",
    "morphology",
    "text",
    "perturbation"
  ],
  "recommended_models": {
    "gene": ["enformer", "borzoi", "nt", "evo"],
    "protein": ["esm2_t6_8M", "esm2_t12_35M", "esm2_t30_150M", "esm2_t33_650M_UR50D", "esm1v", "prott5"],
    "molecule": ["chemberta", "molformer_base", "rdkit_2d", "rdkit_morgan"],
    "dna": ["enformer", "borzoi", "nt", "evo"],
    "single_cell": ["pca", "scvi", "scgpt", "geneformer_v2_12L", "state"],
    "morphology": ["subcell", "structure"],
    "text": ["minilm", "bge_large", "llama"],
    "perturbation": ["state", "subcell", "chemberta", "scgpt", "geneformer_v2_12L"]
  },
  "runnable_singlecell_models": ["pca", "scvi", "scanvi", "scgpt", "geneformer_v2_12L", "state"]
}
```

---

## 5. `compute_embeddings`

Wraps `BioEmbedder.embed()` to extract multi-model latent representations and stores them directly in `.obsm`, `.varm`, or `.uns` on disk.

### Supported Modalities & Model Roster

The table below catalogs the foundation models available across the 8 supported biological modalities, along with their standardized scverse AnnData destination slots:

| Modality (`entity_type`) | Foundation Models & Aliases | Key Catalog Models | Target AnnData Slot | Typical Dimensions | Description & Biological Target |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`gene`** | `enformer`, `borzoi`, `nt`, `evo`, `hyenadna`, `dnabert2` | `enformer_human_rough`, `borzoi_v0`–`v3`, `nt_500m_human_ref`, `nt_v2_500m`, `evo1_8k`, `hyenadna_tiny_1k`–`large_1m`, `caduceus_ph_131k` | `.varm` (feature vectors) or `.obsm` (`is_perturbation=True`) | `(n_genes, 256-4096)` | Gene sequence representations, regulatory genomics, transcriptional impact |
| **`protein`** | `esm2_t33_650M_UR50D`, `esm1v`, `prott5`, `esm2_t6_8M`, `esm2_t12_35M`, `esm2_t30_150M` | `esm2_8M`, `esm2_35M`, `esm2_150M`, `esm2_650M`, `esm2_3B`, `esm2_15B`, `esm1v_1`–`5`, `esm1b`, `prot_t5_xl`, `esmc_300m`, `esmc_600m`, `esm3_small/med/large`, `boltz2` | `.obsm` (perturbation) or `.varm` / `.uns` | `(n_proteins, 320-5120)` | Protein evolutionary fitness, variant effect predictions, tertiary structural geometry |
| **`molecule`** | `chemberta`, `molformer_base`, `rdkit_2d`, `rdkit_morgan` | `chemberta2MTR`, `chemberta2MLM`, `molformer_base`, `rdkit_fp`, `morgan_fp`, `morgan_count_fp`, `maccs_fp`, `atom_pair_fp`, `minimol`, `mole` | `.obsm` (perturbation) or `.uns` | `(n_molecules, 128-1024)` | Chemical latent representations, physicochemical properties, bioactivity fingerprints |
| **`dna`** | `enformer`, `borzoi`, `nt`, `evo` | `enformer_human_rough`, `borzoi_v0`–`v3`, `flashzoi_v0`–`v3`, `nt_500m_human_ref`, `nt_v2_500m`, `ntv3_650m_pos`, `evo1_8k`, `evo1_131k`, `evo2_7b`, `hyenadna_large_1m` | `.varm` (feature vectors) or `.obsm` (`is_perturbation=True`) | `(n_sequences, 256-4096)` | Regulatory sequence representations, chromatin accessibility, transcription factor binding |
| **`single_cell`** | `pca`, `scvi`, `scgpt`, `geneformer_v2_12L`, `state` | `pca`, `scvi`, `scanvi`, `totalvi`, `scgpt`, `geneformer_v2_12L`, `geneformer_v2_18L`, `state`, `stack`, `uce`, `cell2sentence_2b`, `tahoe_70m` | `.obsm['X_<model>']` | `(n_cells, 30-1024)` | Whole-cell transcriptomic states, latent manifold topology, cluster conservation |
| **`morphology`** | `subcell`, `structure` | `subcell_mae_rybg`, `subcell_vit_rybg`, `subcell_mae_rbg`, `subcell_vit_ybg`, `subcell_mae`, `subcell_contrast`, `boltz2` | `.obsm['X_<model>']` | `(n_cells, 512-1024)` | High-content imaging profiles, subcellular organelle localization, phenotypic screens |
| **`text`** | `minilm`, `bge_large`, `llama` | `minilm_l6_v2`, `bert_base_uncased`, `llama3.2_1b`, `llama3.2_3b`, `llama3.1_8b`, `openai_small`, `openai_large`, `voyage_3` | `.obsm` or `.uns` | `(n_entities, 384-4096)` | Biomedical literature embeddings, gene descriptions, clinical concept embeddings |
| **`perturbation`** | `state`, `subcell`, `chemberta`, `scgpt`, `geneformer_v2_12L` | `state`, `subcell_mae_rybg`, `chemberta2MTR`, `scgpt`, `geneformer_v2_12L`, `geneformer_v1_12L`, `pca`, `stack` | `.obsm['X_<model>']` | `(n_cells, 30-1024)` | Transcriptional and morphological perturbation responses, differential cell state shifts |

### Parameters
- `path` (*str*, required): Path to the preprocessed `.h5ad` dataset.
- `models` (*list[str] | str*, default="pca"): Model roster to compute. Call `list_model_registry` to view all ~150 available models dynamically.
  - `single_cell`: `["pca", "scvi", "scgpt", "geneformer_v2_12L", "state"]`
  - `gene`: `["hyenadna_tiny_1k", "dnabert2", "ntv2_500m", "enformer_human_rough"]`
  - `protein`: `["esm2_t33_650M_UR50D", "esm1v", "prott5"]`
  - `molecule`: `["chemberta", "molformer_base", "rdkit_2d"]`
  - `dna`: `["enformer", "borzoi", "nt", "evo"]`
  - `morphology`: `["subcell", "structure"]`
  - `text`: `["minilm", "bge_large"]`
  - `perturbation`: `["state", "subcell", "chemberta"]`
- `output_path` (*str*, optional): Filesystem path to save the embedded dataset.
- `entity_type` (*{'gene', 'protein', 'molecule', 'dna', 'single_cell', 'morphology', 'text', 'perturbation'}*, default="single_cell"): Biological entity modality.
- `batch_size` (*int*, default=32): Inference batch size.
- `device` (*str*, default="auto"): Compute device (`"auto"`, `"cuda"`, `"mps"`, `"cpu"`).
- `preprocessing` (*{'auto', 'standard', 'raw', 'none'}*, default="auto"): Single-cell preprocessing strategy.

### Response Schema
```json
{
  "status": "success",
  "output_path": "/data/pbmc_embedded.h5ad",
  "entity_type": "single_cell",
  "n_cells": 2650,
  "models_computed": ["pca", "scvi"],
  "embedding_keys": ["X_pca", "X_scvi"],
  "embedding_shapes": {
    "X_pca": [2650, 50],
    "X_scvi": [2650, 30]
  },
  "execution_time_seconds": 12.4
}
```

---

## 6. `annotate_metadata`

Enriches `.obs` or `.uns` with external biological metadata (GO terms, pathways, disease associations, protein domains, molecular interactions) based on entities present.

### Parameters
- `path` (*str*, required): Path to the AnnData `.h5ad` dataset.
- `annotation_type` (*{'go', 'pathways', 'domains', 'diseases', 'interactions', 'cell_line'}*, default="pathways"): Type of biological annotation:
  - `"go"`: Gene Ontology terms (biological process, molecular function, cellular component).
  - `"pathways"`: Biological pathways (KEGG, Reactome, WikiPathways).
  - `"domains"`: Protein functional domains and families (Pfam, InterPro).
  - `"diseases"`: Disease associations and clinical indications (DisGeNET, Open Targets).
  - `"interactions"`: Molecular and protein-protein interactions (STRING, IntAct).
  - `"cell_line"`: Cell line annotations, lineages, and dependencies (DepMap, Cellosaurus).
- `annotation_types` (*list[str] | str*, optional): Alias or multi-select options for annotation types.
- `entity_type` (*{'auto', 'gene', 'protein', 'molecule', 'cell_line'}*, default="auto"): Entity type.
- `column` (*str*, optional): Target column to use as query (or None to check obs_names/var_names).
- `obs_column` (*str*, optional): Alias for column.
- `output_path` (*str*, optional): Destination path to save the annotated dataset.
- `organism` (*str*, default="human"): Target organism (`"human"`, `"mouse"`).

### Response Schema
```json
{
  "status": "success",
  "output_path": "/data/pbmc_annotated.h5ad",
  "entity_type": "gene",
  "annotation_type": "pathways",
  "target_column": "perturbation",
  "new_columns_added": ["pathway_name", "go_biological_process"],
  "successful_count": 48,
  "failed_count": 0,
  "summary": "Annotated 48/48 gene entities using column 'perturbation'. Added 2 new metadata features to .obs."
}
```

---

## 7. `compute_umap`

Computes kNN neighborhood graphs and 2D UMAP coordinates over a selected representation in `.obsm`.

### Parameters
- `path` (*str*, required): Path to the embedded `.h5ad` AnnData dataset.
- `embedding_key` (*str*, default="X_pca"): Source embedding representation in `.obsm`.
- `output_key` (*str*, optional): Custom `.obsm` key (defaults to `X_umap_<model>`).
- `output_path` (*str*, optional): Filesystem path to write updated AnnData.
- `n_neighbors` (*int*, default=15): Number of neighbors for graph construction.
- `min_dist` (*float*, default=0.5): Minimum distance parameter for UMAP.
- `random_state` (*int*, default=42): Random seed for reproducibility.

### Response Schema
```json
{
  "status": "success",
  "embedding_key": "X_pca",
  "umap_key": "X_umap_pca",
  "n_cells": 2650,
  "umap_shape": [2650, 2],
  "output_path": "/data/pbmc_umap.h5ad"
}
```

---

## 8. `benchmark_scib`

Wraps `tl.compute_scib_metrics()` to score embeddings on biological conservation (NMI, ARI, ASW label) and batch correction (ASW batch, graph connectivity, cLISI). Returns a concise Markdown comparison table.

### Parameters
- `path` (*str*, required): Path to the `.h5ad` dataset containing embeddings in `.obsm`.
- `embedding_keys` (*list[str] | str*, required): List of representation keys to benchmark.
- `label_key` (*str*, required): Column in `.obs` holding cell type / cluster labels.
- `batch_key` (*str*, optional): Column in `.obs` holding technical batch annotations.
- `output_csv` (*str*, optional): Path to export metric results as CSV.

### Response Schema
```json
{
  "status": "success",
  "embedding_keys": ["X_pca", "X_scvi"],
  "rankings": ["X_scvi", "X_pca"],
  "markdown_table": "| | NMI | ARI | ASW_label | ASW_batch | Bio_Conservation | Total |\n|---|---|---|---|---|---|---|\n| X_scvi | 0.7421 | 0.6812 | 0.8123 | 0.7954 | 0.7452 | 0.7703 |\n| X_pca  | 0.6830 | 0.6120 | 0.7450 | 0.6210 | 0.6800 | 0.6505 |",
  "output_csv": "/data/scib_rankings.csv"
}
```

---

## 9. `benchmark_perturbations`

Wraps `tl.run_cell_eval()` across paired or split datasets, scoring differential perturbation responses against reference controls using Arc Institute `cell-eval`. Returns a concise Markdown summary table.

### Parameters
- `path` (*str*, required): Path to the real observations dataset (or split dataset).
- `control_pert` (*str*, required): Identifier corresponding to unperturbed control (e.g. `'ctrl'`).
- `pert_col` (*str*, default="perturbation"): Column identifying perturbation conditions.
- `split_col` (*str*, optional): Column indicating data splits. Requires `pred_label` and `ref_label`.
- `pred_label` (*str*, optional): Split category representing model predictions (e.g. `'test'`).
- `ref_label` (*str*, optional): Split category representing real reference observations (e.g. `'train'`).
- `path_pred` (*str*, optional): Path to predicted AnnData dataset when using paired files.
- `embedding_keys` (*list[str] | str*, optional): Target representation keys to evaluate.
- `profile` (*str*, default="minimal"): Evaluation profile (`"full"`, `"minimal"`, `"de"`, `"vcc"`, `"anndata"`).
- `output_csv` (*str*, optional): Filesystem path to export results as CSV.

### Response Schema
```json
{
  "status": "success",
  "control_pert": "ctrl",
  "pert_col": "perturbation",
  "n_perturbations": 12,
  "markdown_table": "| Perturbation | Mean_R2 | DEG_Overlap | Delta_L2 | Direction_Agreement |\n|---|---|---|---|---|\n| drug_A | 0.8421 | 0.7912 | 0.1245 | 0.9123 |\n| drug_B | 0.7910 | 0.7104 | 0.1832 | 0.8654 |",
  "output_csv": "/data/pert_metrics.csv"
}
```

---

## Supported Preprocessing Modes

`embpy` enforces a strict output contract that decouples raw integer counts from downstream transformed representations, ensuring that models requiring counts (such as scVI or TotalVI) receive uncorrupted Poisson data, while foundation models (such as scGPT or Geneformer) receive normalized and binned token inputs.

### 1. Quality Control & Filtering
- **`min_genes`**: Removes non-viable cells or empty droplets expressing fewer than a minimum threshold of detected genes (standard: 200).
- **`min_cells`**: Removes rare transcripts expressed in fewer than a specified number of cells across the cohort (standard: 3).
- **Dynamic Starvation Protection (`preprocessing="auto"`)**: In tutorial notebooks, synthetic benchmarks, or small experimental batches (`n_vars < 500` or `n_obs < 100`), cutoffs are automatically adapted (`min(min_genes, n_vars // 3)`) to prevent accidental cell starvation or empty slice crashes.

### 2. Library Size Normalization & Transformation
- **`sc.pp.normalize_total`**: Rescales each cell's total expression to a uniform target count depth (`target_sum=1e4`), eliminating sequencing depth variations across technical batches.
- **`sc.pp.log1p`**: Stabilizes high expression variance using natural logarithmic transformation:
  $$\tilde{x} = \ln(1 + x)$$
- **Layer Preservation**:
  - `adata.layers["counts"]`: Houses untouched raw integer counts.
  - `adata.layers["log_normalized"]`: Houses normalized, log-transformed continuous expression.
  - `adata.layers["scaled"]`: Houses zero-centered, unit-variance scaled data clipped at standard deviation thresholds (optional: `scale=True`).
  - `adata.X`: Restored to raw counts to honor the `embpy` output contract.

### 3. Highly Variable Gene (HVG) Selection & Tokenization
- Computes mean-variance dispersion across features using Seurat or Seurat v3 algorithms, marking top informative genes (`n_top_genes=2000`) in `adata.var["highly_variable"]`.
- For foundation transformer architectures requiring tokenization (scGPT, Geneformer), `embpy` applies rank value encoding: genes are ranked by normalized expression level within each cell, assigned discrete vocabulary tokens, and batched into padded tensor sequences with attention masks.

### 4. Summary of Strategy Profiles
- **`auto`**: Inspects dataset dimensions and current expression nature. Applies dynamic scaling on filtering cutoffs if dimensions are small; runs library normalization, log1p, and selects HVGs.
- **`standard`**: Enforces strict cutoffs (`min_genes=200`, `min_cells=3`), applies `normalize_total` (1e4), `log1p`, selects top 2,000 HVGs, and populates `.layers['counts']` and `.layers['log_normalized']`.
- **`none`**: Completely bypasses all filtering and math transformations. Copies `adata.X` into `.layers['counts']` and writes the object directly to disk, preserving pristine integer counts for models that perform their own internal stochastic variational inference.

---

## Supported Annotation Databases

The `annotate_metadata` skill enriches single-cell, genomic, proteomic, and pharmacological datasets with curated knowledge from reference biological repositories:

| Category | Supported Databases & Services | Description & Biological Content | Output Location |
| :--- | :--- | :--- | :--- |
| **Gene Ontology (GO)** | MyGene.info, Gene Ontology Consortium | Biological Process (BP), Molecular Function (MF), and Cellular Component (CC) terms, including GO IDs and descriptive tags. | `.obs` features and `.uns['gene_annotations']` |
| **Pathways** | KEGG, Reactome, WikiPathways, Enrichr | Canonical signaling networks, biochemical cascades, and metabolic pathways with enrichment statistics and gene set overlap. | `.obs['pathway_name']` and `.uns` |
| **Protein Domains** | Pfam, InterPro, UniProt Knowledgebase | Protein domain architecture, catalytic motifs, zinc fingers, transmembrane domains, and clan classifications. | `.obs` and `.uns['protein_annotations']` |
| **Disease Associations** | DisGeNET, Open Targets Platform, ClinVar | Clinically validated disease-gene associations, therapeutic indications, oncogenic driver classifications, and variant risk scores. | `.obs['disease_association']` and `.uns` |
| **Molecular Interactions** | STRING-db, IntAct, BioGRID | Physical protein-protein interaction (PPI) networks, functional association confidence scores, and interaction partner degrees. | `.obs` and `.uns['ppi_network']` |
| **Cell Line Metadata** | DepMap (Broad Institute), Cellosaurus | Cancer Dependency Map (DepMap) CRISPR-Cas9 dependency scores, RNAi gene effects, copy number alterations, and lineage classifications. | `.obs` and `.uns['cell_line_metadata']` |

---

## Analysis & Benchmarking Methods

### 1. 2D Manifold Visualization (`compute_umap`)
- Builds high-dimensional k-nearest neighbor (kNN) affinity graphs (`sc.pp.neighbors`) across latent spaces stored in `adata.obsm` (e.g., `X_pca`, `X_scvi`, `X_scgpt`).
- Computes Uniform Manifold Approximation and Projection (`sc.tl.umap`) coordinates to project representations into 2D manifolds for cell type visual inspection.
- Embeds coordinates into `adata.obsm['X_umap_<model>']` and returns array dimensions `(n_cells, 2)` to the agent without dumping raw floating-point matrices.

### 2. Single-Cell Integration Benchmarking (`benchmark_scib`)
Evaluates representations using the single-cell integration benchmarking (scIB) metric framework, providing quantitative scores across two primary axes:
- **Biological Conservation Metrics:**
  - **Normalized Mutual Information (NMI)**: Measures clustering consistency between ground-truth biological annotations and Louvain/Leiden clusters in latent space ($[0, 1]$).
  - **Adjusted Rand Index (ARI)**: Evaluates partition similarity against ground-truth labels adjusted for chance ($[0, 1]$).
  - **Average Silhouette Width - Label (`ASW_label`)**: Measures compactness of identical cell types relative to other lineages ($[0, 1]$).
  - **Isolated Label F1 & ASW**: Tests preservation of rare, clinically significant cell types against dominant populations.
  - **Cell Cycle Conservation**: Assesses variance preservation across non-cycling vs cycling phases.
- **Batch Correction Metrics:**
  - **Average Silhouette Width - Batch (`ASW_batch`)**: Quantifies batch mixing where 1.0 indicates perfect blending across donors or sequencing runs.
  - **Graph Connectivity**: Determines whether all cells of a given cell type remain connected in the kNN graph despite batch origins.
  - **Integration Local Inverse Simpson's Index (`iLISI`)**: Gauges local neighborhood diversity with respect to batch labels ($[1, n_{batches}]$).
  - **k-nearest neighbor Batch Effect Test (`kBET`)**: Tests whether batch distributions within local neighborhoods replicate the global sample distribution.
- Returns a Markdown comparison table and composite rankings to guide autonomous model selection.

### 3. Perturbation Prediction Benchmarking (`benchmark_perturbations` / `cell-eval`)
Integrates the Arc Institute `cell-eval` benchmarking harness to score predicted transcriptional response profiles against real experimental observations:
- **Profile Correlation & Mean $R^2$**: Evaluates overall variance explained across all genes and perturbations compared to unperturbed controls (`control_pert`).
- **Differentially Expressed Gene (DEG) Overlap**: Measures recall, precision, and Jaccard similarity for the top 50, 100, and 200 differentially expressed genes.
- **Directional Agreement & Delta $L_2$**: Quantifies whether predicted transcriptional shifts point in the true biological direction in high-dimensional gene expression space.
- Supports paired evaluation (`path_pred` vs `path`) and split-based evaluation (`split_col` with explicit `pred_label` and `ref_label`).

