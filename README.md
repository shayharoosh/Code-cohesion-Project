# Code Cohesion Supply-Chain Attack Detection

This repository contains the code and experimental pipeline for measuring code functional cohesion and detecting supply-chain attacks (injected/malicious code changes) across software repositories.

The core hypothesis is that malicious code injections disrupt a function's semantic cohesion—the alignment between its implementation, name, and comments. By extracting function histories, predicting semantic names using LLMs, and scoring cohesion deltas over version transitions, we train tree-based machine learning detectors (LightGBM) to pinpoint supply-chain anomalies.

---

## 🛠 Project Architecture & Pipeline Overview

The research workflow is structured into four main Jupyter Notebooks:

```
                          ┌────────────────────────────────────────┐
                          │    repositories.csv                    │
                          └───────────────────┬────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. download_github_reposetories.ipynb                                                   │
│    • Clones target repos via pygit2                                                      │
│    • Extracts function history over time using Tree-Sitter (C, C++, JS, Python)          │
│    • Exports parquet dataset partitions of function evolution                            │
└─────────────────────────────────────┬────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 2. creating_names_for_function.ipynb                                                     │
│    • Loads functions_over_time.jsonl / parquet history                                   │
│    • Uses Code LLMs (transformers + 4-bit bitsandbytes) to generate zero-shot function   │
│      names from implementation code                                                      │
│    • Saves predictions to predicted_names_data.jsonl / predicted_names_malware.jsonl     │
└─────────────────────────────────────┬────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 3. creating_values_from_names.ipynb                                                      │
│    • Computes semantic embeddings using sentence-transformers                            │
│    • Measures cohesion metrics (cosine similarity of predicted vs. actual names/comments)│
│    • Generates dataset metrics (e.g., cohesion_metrics_clean.csv, js_metrics.csv)       │
└─────────────────────────────────────┬────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ 4. CodeCohesionExperiment.ipynb                                                          │
│    • Part 1: Builds paired feature matrix (v_i, v_{i+1}) with project-local z-scores    │
│    • Part 2: Feature validation & Logistic Regression baseline                           │
│    • Part 3: GroupKFold cross-validation (by repository) for LightGBM model selection    │
│    • Part 4: Trains final LightGBM model & evaluates on held-out test projects           │
│    • Part 5: Paper simulation under class imbalance (1:100, 1:1,000, 1:10,000) (P@100/AP)   │
│    • Part 6: Cohesion-level breakdown (low, mid, high cohesion codebases)               │
│    • Part 7: Feature explainability (Permutation Importance & SHAP)                     │
│    • Parts 8 & 9: Scoring JS pairs & evaluating P@100 on known malware (CodeQL dataset)  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Setup & Installation

### 1. Prerequisites & Environment
Ensure you have Python 3.9+ installed along with GPU support (e.g., NVIDIA RTX 3090 / CUDA) for LLM inference and model training.

Install all required Python dependencies:
```bash
pip install -r requirements.txt
```

### 2. Data Preparation
1. Create a `data/` folder in the project root if it does not exist:
   ```bash
   mkdir -p data
   ```
2. Download the pre-extracted `functions_over_time.jsonl` dataset from [Google Drive](https://drive.google.com/file/d/1YgyyMseE4-QWdaxNmFWM5al0rat_WzoV/view?usp=drive_link).
3. Place `functions_over_time.jsonl` into the `data/` directory.

---

## 📖 How to Run the Experiments

### Quick Start: Reproduce Paper Experiments
If you already have `functions_over_time.jsonl` or pre-computed cohesion metrics in `data/`:
1. Open [`CodeCohesionExperiment.ipynb`](file:///Users/nachmias/code-cohesion/CodeCohesionExperiment.ipynb).
2. Execute all cells to run the full machine learning training, cross-validation, paper simulations, explainability analysis, and JS malware evaluation.

### End-to-End Pipeline Execution
To execute the pipeline from scratch:

1. **Repository & Function Extraction**:
   Run [`download_github_reposetories.ipynb`](file:///Users/nachmias/code-cohesion/download_github_reposetories.ipynb) to fetch GitHub repositories listed in [`repositories.csv`](file:///Users/nachmias/code-cohesion/repositories.csv) and extract historical function states using Tree-Sitter.

2. **LLM Function Name Generation**:
   Run [`creating_names_for_function.ipynb`](file:///Users/nachmias/code-cohesion/creating_names_for_function.ipynb) to load function histories and generate predicted function names via quantized Code LLMs.

3. **Cohesion Metric Calculation**:
   Run [`creating_values_from_names.ipynb`](file:///Users/nachmias/code-cohesion/creating_values_from_names.ipynb) to compute semantic embeddings and export metric CSV files to `data/`.

4. **Model Training & Evaluation**:
   Run [`CodeCohesionExperiment.ipynb`](file:///Users/nachmias/code-cohesion/CodeCohesionExperiment.ipynb) to train the LightGBM detector and evaluate supply-chain attack detection performance.

---

## 💻 Experimental Setup & Hardware
- **GPU Setup**: Experiments were executed on an NVIDIA RTX 3090 GPU (24GB VRAM).
- **LLM Quantization**: 4-bit (`bitsandbytes`) quantization used for fast local LLM inference.
- **Model Evaluation**: GroupKFold cross-validation grouped by repository to prevent cross-project data leakage.
