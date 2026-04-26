# KEGG Transporter Signaling Network Analysis

A computational biology pipeline that constructs a signaling network from KEGG transporter gene families, annotates it using an LLM, and performs multi-layered network analysis — including hub identification, pathway enrichment, link prediction, drug-disease integration, and Boolean network simulation.

---

## Project Overview

The pipeline begins with **46 transporter genes** spanning ABC transporters, SLC family carriers, ion channels, aquaporins, and ion pumps. From these seed genes, KEGG pathway data is collected and structured into a directed protein-protein interaction network via GPT-4o annotation. The resulting network (826 nodes, 1107 edges) then serves as the foundation for all downstream analyses.

```
KEGG API (transporter genes)
        ↓
  01: Data Collection
        ↓
  02: LLM Annotation (GPT-4o) → oncogene_pathways.xml
        ↓
  ┌─────────────────────────────────────────────────┐
  │                 Downstream Analysis              │
  ├──────────────┬──────────────┬───────────────────┤
  03: Hub &    04-07: Link    08: Drug-Disease
  Enrichment   Prediction     Heterogeneous
  Analysis     (4 methods)    Network
                    ↓
              09: Boolean Network
              Simulation
  └─────────────────────────────────────────────────┘
```

---

## Repository Structure

```
kegg-transporter-signaling-network/
│
├── README.md
├── requirements.txt
│
├── notebooks/
│   ├── 01_data_collection_kegg_api.ipynb
│   ├── 02_network_construction_llm_annotation.ipynb
│   ├── 03_hub_identification_pathway_enrichment.ipynb
│   ├── 04_link_prediction_topology_adamic_jaccard.ipynb
│   ├── 05_link_prediction_node2vec_undirected.ipynb
│   ├── 06_link_prediction_logistic_regression.ipynb
│   ├── 07_link_prediction_node2vec_directed.ipynb
│   ├── 08_drug_disease_heterogeneous_network.ipynb
│   └── 09_boolean_network_attractor_simulation.ipynb
│
├── data/
│   ├── raw/                        # KEGG API raw JSON outputs
│   │   └── oncogene_ppi.json
│   ├── processed/                  # Intermediate network files
│   │   └── oncogene_pathways.xml   # Core network (826 nodes, 1107 edges)
│   └── results/                    # Analysis outputs
│       ├── link_prediction_topological.csv
│       ├── link_predictions_node2vec.csv
│       ├── directed_link_predictions_manual.csv
│       ├── directed_network_with_predictions.graphml
│       └── kegg_enrichment.csv
│
└── figures/                        # Network visualizations
```

---

## Notebooks

### 01 · Data Collection — KEGG API
**`01_data_collection_kegg_api.ipynb`**

Queries the KEGG REST API starting from 46 manually curated transporter genes. For each gene, retrieves associated pathways, network elements, and KEGG Network Element (NE) references. Traverses each pathway to collect additional element IDs, then fetches name and definition for every element. Outputs a JSON dictionary of protein interaction elements.

- Input: gene symbol → KEGG ID dictionary (hardcoded)
- Output: `data/raw/oncogene_ppi.json`

---

### 02 · Network Construction — LLM Annotation
**`02_network_construction_llm_annotation.ipynb`**

Feeds each KEGG element's name and definition to GPT-4o-mini in parallel (ThreadPoolExecutor, 10 workers). The model is prompted to return a structured XML block encoding nodes (with cellular location and molecular function) and edges (with interaction type: activation, inhibition, indirect effect, etc.). Output XML fragments are cleaned, parsed, and assembled into a single NetworkX `DiGraph`. The final graph is serialized as `oncogene_pathways.xml`.

- Input: `data/raw/oncogene_ppi.json`
- Output: `data/processed/oncogene_pathways.xml`
- External dependency: OpenAI API key

---

### 03 · Hub Identification & Pathway Enrichment
**`03_hub_identification_pathway_enrichment.ipynb`**

Loads the network from XML and computes multiple centrality metrics (degree, betweenness, PageRank). The top 200 PageRank nodes are filtered for valid gene symbols and submitted to Enrichr (KEGG_2021_Human gene set) for over-representation analysis. Enriched pathways are then mapped to broad disease categories (cancer, neurodegeneration, viral infection, metabolic disease, etc.) using keyword matching.

- Input: `data/processed/oncogene_pathways.xml`
- Output: `data/results/kegg_enrichment.csv`

---

### 04 · Link Prediction — Topological Indices
**`04_link_prediction_topology_adamic_jaccard.ipynb`**

Converts the directed graph to undirected and computes Adamic-Adar and Jaccard coefficients for all non-adjacent node pairs. Scores are min-max normalized and combined into a single ranking. Top predictions are optionally validated against the STRING protein interaction database (score threshold: 0.4). Results are visualized as a subgraph with predicted edges highlighted in red.

- Input: `data/processed/oncogene_pathways.xml`
- Output: `data/results/link_prediction_topological.csv`

---

### 05 · Link Prediction — Node2Vec (Undirected)
**`05_link_prediction_node2vec_undirected.ipynb`**

Uses the `node2vec` library to learn 64-dimensional node embeddings on the undirected graph (walk length 30, 200 walks per node, p=1, q=1). Edge features are constructed via **Hadamard product** of node embeddings. A Random Forest classifier (100 trees) is trained on positive edges and randomly sampled negative edges, then used to score all non-adjacent pairs.

- Input: `data/processed/oncogene_pathways.xml`
- Output: `data/results/link_predictions_node2vec.csv`
- Test AUC: reported per run (~0.90+)

---

### 06 · Link Prediction — Logistic Regression
**`06_link_prediction_logistic_regression.ipynb`**

A baseline link prediction model using hand-crafted topological features (common neighbours, degree product, shortest path, etc.) with Logistic Regression. Serves as a comparator to the Node2Vec-based approaches.

- Input: `data/processed/oncogene_pathways.xml`

---

### 07 · Link Prediction — Node2Vec (Directed)
**`07_link_prediction_node2vec_directed.ipynb`**

Implements biased random walks directly on the directed graph (following only out-edges), without relying on the `node2vec` library. Edge features are constructed by **concatenating** source and target embeddings (preserving directionality). A Random Forest classifier scores all directed non-adjacent pairs. High-confidence predictions (≥ 0.95) are added to the graph and exported as GraphML.

- Input: `data/processed/oncogene_pathways.xml`
- Output: `data/results/directed_link_predictions_manual.csv`, `directed_network_with_predictions.graphml`
- Test AUC: 0.9559

---

### 08 · Drug-Disease Heterogeneous Network
**`08_drug_disease_heterogeneous_network.ipynb`**

Extends the signaling network into a heterogeneous knowledge graph by integrating two external databases. OpenTargets association scores link network proteins to diseases. ChEMBL mechanism-of-action data links drugs to their protein targets. The resulting graph adds drug nodes and disease nodes to the existing protein interaction graph, with typed edges (`targeted_by`, `associated_with`).

- Input: `data/processed/oncogene_pathways.xml`, OpenTargets parquet files (local)
- External data: OpenTargets `target/`, `disease/`, `association_overall_direct/`, `drug_molecule/`, `drug_mechanism_of_action/`

---

### 09 · Boolean Network Simulation
**`09_boolean_network_attractor_simulation.ipynb`**

Extracts a subnetwork seeded from high-PageRank nodes. Assigns Boolean update rules based on edge types (UREG/LIG → activation; DREGC/DREGNC → inhibition). Runs stochastic attractor search (500 random initial states, max 100 steps per trajectory). Identifies steady-state attractors and their basins of attraction, then performs systematic single-node perturbation experiments to identify nodes whose knockout or forced activation shifts the system to an alternative attractor.

- Input: `data/processed/oncogene_pathways.xml`
- Dependency: `BooleanNetwork` class (defined in notebook)

---

## Key Data File

**`data/processed/oncogene_pathways.xml`** is the central artifact of this project. All downstream notebooks (03–09) load from this file. It encodes:
- Nodes with attributes: `name`, `cellular_location`, `function`
- Edges with attributes: `type` (UREG, DREGC, LIG, INTNSB, etc.)
- Multi-pathway structure (one `<pathway>` block per KEGG element)

---

## Setup

```bash
git clone https://github.com/<your-username>/kegg-transporter-signaling-network.git
cd kegg-transporter-signaling-network
pip install -r requirements.txt
```

For notebook 02, set your OpenAI API key:
```bash
export OPENAI_API_KEY="sk-..."
```

For notebook 08, download OpenTargets data files from https://platform.opentargets.org/downloads and place them in `data/external/`.

---

## Gene Families Covered

| Family | Genes |
|---|---|
| ABC Transporters | ABCB1, ABCC1–6, ABCG2, CFTR, ABCA1, ABCG5/8 |
| SLC Carriers | SLC2A1, SLC5A1, SLC6A4, SLC12A2, SLC15A1, SLC22A4–8, SLC25A4, SLC47A1 |
| Ion Channels | SCN1A, SCN5A, KCNJ11, CACNA1C, CLCN1/2, TRPV1, TRPM6 |
| Aquaporins | AQP1–4 |
| Ion Pumps | ATP1A1, ATP4A |

---

## Methods Summary

| Notebook | Method | AUC / Notes |
|---|---|---|
| 04 | Adamic-Adar + Jaccard | Topological baseline |
| 05 | Node2Vec + RF (undirected) | ~0.90+ |
| 06 | Logistic Regression | Baseline comparator |
| 07 | Node2Vec + RF (directed) | **0.9559** |
