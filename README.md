# Hybrid Provenance Graph Detection

## Overview

This project implements an offline, end-to-end hybrid threat detection system based on **Provenance Graphs**. It is designed to analyze system audit logs (Auditbeat) to detect complex cyber threats, such as privilege escalation and data theft.

The system addresses the "dependency explosion" problem in provenance analysis by combining two methods:
1.  **Graph Neural Networks (GNN):** An unsupervised AutoEncoder to calculate structural anomaly scores ($S_{GNN}$) for noise reduction.
2.  **Heuristic Rules:** Expert knowledge-based rules to filter for known suspicious behaviors.

This "Dual-Filter" approach effectively extracts high-confidence attack chains from noisy graph data.

## System Architecture

The analysis pipeline consists of three stages:

1.  **Data Layer (`log_parser.py`):**
    * Parses raw `ndjson` audit logs.
    * Constructs a Provenance Graph in Neo4j with deep semantic features (e.g., `executable` paths, command line `args`).

2.  **Feature Layer (`feature_extractor.py`):**
    * Pre-computes structural features (In-degree/Out-degree) directly within the database.
    * Applies baseline heuristic rules to generate semantic labels (`is_suspicious`).

3.  **Analysis Layer (`gnn_trainer.py` & Cypher Query):**
    * Trains a GNN AutoEncoder to learn "normal" graph structures.
    * Writes anomaly embeddings and scores back to the database.
    * Executes a hybrid Cypher query to extract the final attack chain.

## Prerequisites

* **Database:** Neo4j Community Edition (v4.x or v5.x)
* **Language:** Python 3.8+
* **Dependencies:**
    * `py2neo`
    * `torch`
    * `torch_geometric`

## Installation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/Zhanke422/CS5231-Final-Project
    cd CS5231-Final-Project
    ```

2.  **Install Python dependencies:**
    ```bash
    pip install py2neo torch torch_geometric
    ```

3.  **Configure Database:**
    * Ensure your Neo4j database is running at `bolt://localhost:7687`.
    * Update the `NEO4J_PASSWORD` variable in all three Python scripts (`log_parser.py`, `feature_extractor.py`, `gnn_trainer.py`) to match your local configuration.

## Usage

Run the scripts in the following order to execute the full pipeline:

### Step 1: Parse Logs and Build Graph
Import the audit logs into Neo4j. *Note: Ensure the `log_file_path` in the script points to your actual `.ndjson` file.*

```bash
python3 log_parser.py
```

### Step 2: Extract Features
Compute structural degrees and apply initial semantic tagging in the database. This step prepares the numerical features required by the GNN.

```
python3 feature_extractor.py
```

### Step 3: Train GNN and Score Anomalies
Train the AutoEncoder model. The script will automatically fetch graph data, train the model, and write the learned embeddings back to the Process nodes in Neo4j.

```
python3 gnn_trainer.py
```

### Step 4: Final Hybrid Detection
Open your Neo4j Browser (http://localhost:7474) and run the following Dual-Filter Query to visualize the attack chain. This query finds nodes that are both structurally anomalous (high GNN score) and behaviorally suspicious.

```
MATCH (f1:File {path: '/home/student/secret/secret.txt'})
MATCH (f2:File {path: '/home/attacker/output.txt'})
MATCH (p:Process)-[r1:ACCESSED_FILE]->(f1)
MATCH (p)-[r2:ACCESSED_FILE]->(f2)
WHERE p.is_suspicious = 1 AND p.gnn_embedding IS NOT NULL
MATCH (parent:Process)-[r_created:CREATED]->(p)
RETURN parent, r_created, p, r1, f1, r2, f2
```
