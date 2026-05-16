# PIFLLM: Prediction Instruction Fine-Tuned Large Language Model for Predicting CircRNA-Disease Associations

PIFLLM is an end-to-end framework for predicting circRNA-disease associations. The project starts from raw Excel files, builds reproducible circRNA and disease node mappings, learns graph structural representations with LightGCN, injects these graph features into prediction-oriented instructions, and finally fine-tunes a large language model with LoRA for binary association prediction.

## Overview

The overall workflow is shown below:

```text
Raw Excel files
      ↓
Node mapping and edge construction
      ↓
LightGCN-based graph feature learning
      ↓
Instruction-style JSON data generation
      ↓
LoRA fine-tuning of Llama-3.2-3B-Instruct
      ↓
Evaluation and prediction
```

## Project Structure

```text
/root/autodl-tmp
├── data_new/
│   ├── 去重+名称标准化后的最终数据库.xlsx
│   ├── associationMatrix.xlsx
│   ├── circRNA序列.xlsx
│   ├── Mesh_ID.xlsx
│   ├── Integrated_sqe_fun_circRNA_similarity_2738.xlsx
│   └── 275GS_DMS.xlsx
│
├── src/
│   ├── 1_build_circRNA_disease_edges.py
│   ├── 2_process_data.py
│   ├── 3_fine_tune.py
│   ├── 4_evaluate.py
│   └── 5_predict.py
│
├── train_data.json
├── val_data.json
├── test_data.json
└── llama_rna_disease_3b/
    ├── final_model/
    └── evaluation_metrics.csv
```

## Data Description

The input files are stored in the `data_new/` directory.

| File | Description |
|---|---|
| `去重+名称标准化后的最终数据库.xlsx` | Deduplicated and standardized circRNA-disease association pairs, used as positive samples. |
| `associationMatrix.xlsx` | Association matrix between circRNAs and diseases. It defines the fixed node order and node IDs. |
| `circRNA序列.xlsx` | circRNA sequence information, including circRNA names, sequences, sequence lengths, and GC content. |
| `Mesh_ID.xlsx` | Mapping between disease names and MeSH IDs. |
| `Integrated_sqe_fun_circRNA_similarity_2738.xlsx` | Optional circRNA similarity matrix, used for order consistency checking. |
| `275GS_DMS.xlsx` | Optional disease similarity matrix, used for order consistency checking. |

## Main Outputs

After running the data processing pipeline, the following JSON files are generated in the project root directory:

```text
train_data.json
val_data.json
test_data.json
```

Each sample contains two fields:

```json
{
  "Human": "Instruction prompt containing circRNA information, disease information, and graph structural features.",
  "GPT": "0 or 1"
}
```

The label definition is:

- `1`: the circRNA and disease are associated;
- `0`: the circRNA and disease are not associated.

## Graph Feature Injection

PIFLLM injects LightGCN-derived graph structural features into the instruction prompt. Each circRNA and disease node is represented by a 16-dimensional vector.

The circRNA graph feature is formatted as:

```text
<R_start>0.1234,-0.2451,...<R_end>
```

The disease graph feature is formatted as:

```text
<D_start>0.0921,0.3812,...<D_end>
```

In this way, the final prompt contains both biological textual information and graph topology-aware structural information.

## Method

### Step 1: Node Mapping and Edge Construction

Run:

```bash
python src/1_build_circRNA_disease_edges.py
```

This script performs the following operations:

1. Reads the fixed circRNA and disease order from `associationMatrix.xlsx`;
2. Assigns a unique `circRNA_ID` to each circRNA;
3. Assigns a unique `Disease_ID` to each disease;
4. Converts circRNA-disease association pairs into integer ID-based edges;
5. Removes duplicate edges and sorts them;
6. Generates node mapping files, edge files, and statistics.

Generated files:

```text
data_new/circRNA_nodes.csv
data_new/Disease_nodes.csv
data_new/circRNA_Disease_edges.csv
data_new/circRNA_Disease_edges_stats.txt
```

### Step 2: Training Data Generation

Run:

```bash
python src/2_process_data.py
```

This script performs the following operations:

1. Loads positive circRNA-disease association pairs;
2. Matches circRNA names with sequence information;
3. Matches disease names with MeSH IDs;
4. Builds a circRNA-disease bipartite graph;
5. Learns 16-dimensional graph structural features for circRNAs and diseases using LightGCN;
6. Constructs positive and negative samples;
7. Generates instruction-style JSON files for fine-tuning.

The data split ratio is:

```text
train : validation : test = 6 : 2 : 2
```

### Step 3: LoRA Fine-Tuning

Run:

```bash
python src/3_fine_tune.py
```

The base model is loaded from the local directory:

```text
Llama-3.2-3B-Instruct/
```

LoRA configuration:

| Parameter | Value |
|---|---|
| `task_type` | `CAUSAL_LM` |
| `r` | `8` |
| `lora_alpha` | `32` |
| `lora_dropout` | `0.15` |
| `target_modules` | `q_proj`, `k_proj`, `v_proj`, `o_proj` |

Training configuration:

| Parameter | Value |
|---|---|
| `num_train_epochs` | `3` |
| `learning_rate` | `2e-5` |
| `max_length` | `512` |
| `per_device_train_batch_size` | `4` |
| `eval_strategy` | `steps` |
| `eval_steps` | `200` |

The fine-tuned model is saved to:

```text
llama_rna_disease_3b/final_model/
```

### Step 4: Evaluation

Run:

```bash
python src/4_evaluate.py
```

The evaluation script reads:

```text
val_data.json
test_data.json
```

The evaluation results are saved to:

```text
llama_rna_disease_3b/evaluation_metrics.csv
```

The currently recorded evaluation results are:

| Metric | Value |
|---|---:|
| `n_total` | `200` |
| `n_used` | `16` |
| `coverage` | `0.08` |
| `threshold` | `0.294921875` |
| `accuracy` | `0.9375` |
| `precision` | `0.9375` |
| `recall` | `1.0` |
| `f1_score` | `0.967741935483871` |
| `roc_auc` | `0.8666666666666667` |

Note that `n_total`, `n_used`, and `coverage` are affected by the confidence filtering strategy in the evaluation script. The values above only report the currently saved results.

### Step 5: Prediction

To perform prediction with the fine-tuned model, run:

```bash
python src/5_predict.py
```

This script loads the fine-tuned model and predicts whether a given circRNA-disease pair is associated.

## One-Command Reproduction

Run the following commands sequentially in the project root directory `/root/autodl-tmp`:

```bash
python src/1_build_circRNA_disease_edges.py
python src/2_process_data.py
python src/3_fine_tune.py
python src/4_evaluate.py
```

## Environment Requirements

Python 3.9 or later is recommended.

Main dependencies:

```text
torch
pandas
numpy
scikit-learn
transformers
datasets
peft
accelerate
openpyxl
```

Install the dependencies with:

```bash
pip install torch pandas numpy scikit-learn transformers datasets peft accelerate openpyxl
```

If GPU training is required, install the PyTorch version that matches your CUDA environment.

## Notes

1. `associationMatrix.xlsx` is used as the unique source for node ordering. Do not randomly change its row or column order.
2. Graph features are strictly aligned with node IDs. If node ordering is changed, the injected `<R_start>` and `<D_start>` features may become mismatched.
3. Long circRNA sequences are truncated by preserving the beginning and ending segments, with `...` inserted in the middle to reduce tokenizer truncation risk.
4. Before fine-tuning, make sure the local base model directory `Llama-3.2-3B-Instruct/` exists.
5. If the dataset is replaced, the node mapping, graph feature learning, data generation, and fine-tuning steps should be rerun.

## Highlights

- Defines reproducible circRNA and disease node IDs using `associationMatrix.xlsx`;
- Learns graph topology-aware representations with LightGCN;
- Injects 16-dimensional graph structural vectors into prediction-oriented instructions;
- Fine-tunes an instruction-following LLM with LoRA at low computational cost;
- Provides a complete pipeline for data generation, model training, evaluation, and prediction.
