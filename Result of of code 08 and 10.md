# Drug Repurposing and Pathway Prediction via Knowledge Graphs and Network Embedding

This project systematically predicts missing molecular pathway associations and potential therapeutic targets by integrating **protein‑protein interaction networks**, **heterogeneous knowledge graphs**, and **Boolean network simulation**. Multiple link prediction algorithms and drug‑disease association models are benchmarked to provide high‑confidence drug candidates for breast cancer, validated by PubMed literature.

## Core Technical Pipeline

1. **Network Construction**  
   - Parsing a KEGG XML file to build a directed protein‑protein interaction network (826 nodes, 1,107 edges).  
   - Integrating Open Targets gene‑disease associations and ChEMBL drug‑target data to build a heterogeneous network of proteins, drugs, and diseases.

2. **Link Prediction (Missing PPI Prediction)**  
   - Comparing **topological features (Adamic‑Adar + Jaccard)**, **Node2Vec (undirected/directed) + Random Forest**, and **Logistic Regression**.  
   - Evaluation metrics: AUC‑ROC, Average Precision (AP), Precision@K, and STRING database verification.

3. **Drug Repurposing (Breast Cancer)**  
   - Supervised learning: Node2Vec embeddings + Logistic Regression / Random Forest / Relational Graph Convolutional Network (RGCN).  
   - Unsupervised baselines: Path counting, weighted PageRank.  
   - Outputting candidate drugs with PubMed literature validation.

4. **Boolean Network Attractor Analysis**  
   - Generating Boolean logic rules from a subgraph, simulating attractor states, analyzing proliferation/apoptosis/viral phenotypes, and testing perturbation robustness.

## Key Results

### Link Prediction Performance

| Method                          | AUC‑ROC | AP     | Precision@50 | Precision@100 | STRING Verified@50 |
|--------------------------------|---------|--------|--------------|---------------|--------------------|
| Node2Vec Undirected + RF        | **0.9895** | **0.9801** | 0.98         | 0.98          | 0.30               |
| Node2Vec Directed + RF          | 0.9559 | 0.9507 | **1.00**     | 0.97          | 0.44               |
| Logistic Regression (topology) | 0.7636 | 0.7283 | 0.86         | 0.78          | 0.40               |
| Adamic‑Adar + Jaccard           | 0.5610 | 0.5507 | 0.88         | 0.93          | **0.48**           |

- **Node2Vec Undirected + RF** achieves the best overall discriminative power (AUC ≈ 0.99) and 98% precision among high‑confidence predictions.
- Directed Node2Vec reaches **1.00 Precision@50**, indicating extremely reliable top predictions.

### Supervised Drug Repositioning Performance (Breast Cancer)

| Method                     | AUC    | AUPR   |
|----------------------------|--------|--------|
| Random Forest              | **0.9823** | **0.9965** |
| RGCN                       | 0.9556 | 0.9907 |
| Logistic Regression        | 0.9180 | 0.9840 |

- **Random Forest** achieves the best AUC (0.982) and AUPR (0.997), demonstrating that Node2Vec‑based edge features effectively capture drug‑disease associations.

### Candidate Drug Validation (PubMed)

Among 46 candidate drugs recommended by five methods, **28 drugs** have at least one PubMed publication related to breast cancer. Representative validated drugs:

- **LAPATINIB** (Random Forest): Approved for HER2+ breast cancer.
- **PANOBINOSTAT** (Random Forest / PageRank): Multiple studies support anti‑breast cancer activity.
- **CUDC‑101** (Path counting / PageRank): Three articles confirm multi‑target inhibition.
- **VENETOCLAX** (RGCN): Active in breast cancer models.
- **OBLIMERSEN** (RGCN): Entered clinical trials for breast cancer preoperative chemotherapy.

Full validation results are available in `pubmed_validation_breast_cancer.csv`.

### Boolean Network Attractor Analysis

- 124 Boolean rules generated from the PPI subgraph; a **single stable attractor** (39 active nodes) was identified.  
- Perturbation experiments (fixing PI3K=0, RAS=0, TP53=1) did not alter the attractor, indicating high network robustness with no single‑node control point.

## Environment & Dependencies

- Python 3.12, NetworkX, PyTorch, DGL, Node2Vec, scikit-learn, Biopython, etc.  
- See `environment.yml` for detailed installation.

## Reproducibility

1. Prepare data: KEGG XML, Open Targets, ChEMBL (see project structure).  
2. Run scripts in `scripts/` sequentially.  
3. Results are automatically saved in `results/`.

## Value

- **Fill knowledge gaps**: Predict missing PPIs and drug‑disease associations, providing high‑confidence candidates for experimental validation.  
- **Accelerate drug repurposing**: Quickly propose new uses for existing drugs without high‑throughput screening.  
- **Mechanistic insights**: Boolean network simulation reveals state switching and robustness.

## Copyright and Usage Restrictions

**Copyright (c) [2026] [Zeyuan Ma]. All Rights Reserved.**

The code and documentation in this repository are provided for **research, learning, and non‑commercial purposes only**.

**Commercial use is strictly prohibited**, including but not limited to:
- Integrating this code or its derivatives into any commercial product or service.
- Charging fees or deriving commercial benefit from this code or any derivative work.

For commercial licensing, please contact the author.
