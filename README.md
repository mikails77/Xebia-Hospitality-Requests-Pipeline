# LLM-Based Information Extraction from Hospitality Request Emails
### A Proof-of-Concept Pipeline for Xebia's Back-Office Department

> **BSc Business Analytics Thesis, University of Amsterdam, 2025–2026**  
> **Author:** Mikail Imran Shaukat | **Company:** Xebia Nederland B.V.  
> **Supervisor:** Adia Lumadjeng (UvA) | **Company Contact:** Yannick Bosch (Xebia)

---

## Overview

This repository contains all files, notebooks, and data associated with the thesis "LLM-Based Information Extraction for Hospitality Request Emails: A Prompt Engineering Approach".

The project investigates to what extent an LLM-based information extraction pipeline can reliably structure incoming hospitality request emails at Xebia.

The pipeline was built using the OpenAI API and evaluated across a 3×3 experimental matrix using three prompt variants and three model variants, assessed using field-level precision, recall, and F1 score against a manually annotated ground truth dataset of 50 synthetic emails.

---

## Repository Structure

```
Xebia_Hospitality_Pipeline/
│
├── Stage 1 - Real Email Handling
│   └── Extraction Schema & Hospitality Emails.xlsx
│
├── Stage 2 - Data Generation
│   ├── Synthetic Data Generation.ipynb
│   └── Synthetic Data.csv
│
├── Stage 3 - Extraction 
│   ├── Final Extraction Pipeline.ipynb
│   └── Extraction Results.csv
│
├── Stage 4 - Evaluation 
│   ├── Final Evaluation Pipeline.ipynb
│   ├── Ground Truth Annotations for Evaluation.xlsx
│   ├── Overall Evaluation.csv
│   └── Field Level Evaluation.csv
│ 
│── Stage 5 - Proof-of-Concept Tool
│   └── Xebia Hospitality POC Tool.html
│   └── Xebia POC Tool Demo Video.mov
│
└── README.md
```

---

## Prerequisites

To run the notebooks you will need:

- Python 3.10+ and a Jupyter environment (Google Colab, JupyterLab, or VS Code)
- The following packages: `openai`, `pandas`, `numpy`, `openpyxl`
- An OpenAI API key with access to `gpt-4o`, `gpt-4o-mini`, and `gpt-4.1-nano`

> Note: Stages 2 and 3 load the API key via Colab secrets (`google.colab.userdata`). If running outside Colab, replace this with `os.environ['OPENAI_API_KEY']` instead.

---

## Stage 1 — Schema Definition & Real Emails

> **No notebook required.** This stage documents the foundation of the project: the extraction schema and the 17 real emails used as the basis for everything that follows.

### What happened in this stage

Xebia's hospitality back-office staff were interviewed to understand what information is required to process an incoming event request. Based on these interviews, a **12-field extraction schema** was co-defined with the hospitality team, with field names and definitions agreed upon together.

After access to the Xebia hospitality Outlook inbox was received, 17 emails were selected with the hospitality team to represent the natural variation present in real incoming requests. These emails varied in writing style and level of detail. Dutch emails were translated using Outlook and verified. Only the body section of each email was retained.

Each of the 17 emails was manually annotated against the 12 fields of the extraction schema. A field was only given a value if it was explicitly stated in the email. Fields that could be inferred but were not written were marked as *Not mentioned*. The frequency with which each field appeared across the 17 emails was calculated and used to guide synthetic data generation in Stage 2.

### Files

| File | Description |
|------|-------------|
| `Stage 1 - Extraction Schema & Hospitality Emails.xlsx` | Four sheets: the 12-field extraction schema with definitions, all 17 translated emails, the full field-level annotations, and the field frequency rate used to guide synthetic data generation. |

---

## Stage 2 — Synthetic Data Generation

> Generates 50 synthetic hospitality request emails using GPT-4o, with the 17 real emails as stylistic references. Produces the evaluation dataset used in Stages 3 and 4.

### What happened in this stage

The 17 real emails alone were insufficient to produce statistically meaningful field-level evaluation scores. Synthetic data generation was therefore used to create a dataset of 50 emails. An approach supported by growing literature on LLM-based synthetic data generation in low-resource settings.

GPT-4o was used for generation, with all 17 real emails included directly in the prompt as stylistic references. The field frequency percentages calculated in Stage 1 were included in the prompt to ensure the synthetic emails matched the field presence patterns observed in the real data. A temperature of 0.8 was used to encourage stylistic variation.

Initial attempts to generate all 50 emails in a single API call produced structurally repetitive outputs that followed a fixed template. This was resolved by restructuring generation into batches of 5 emails per API call across 10 iterations. Each batch independently referenced the 17 real emails and produced noticeably more varied outputs consistent with the real emails.

The notebook also handles a known OpenAI API behaviour where the model occasionally wraps JSON output in markdown code fences despite explicit instructions not to. So a cleaning function was added that strips these before parsing.

### How to run

1. Upload `Stage 1 - Extraction Schema & Hospitality Emails.xlsx` to your Colab session
2. Add your OpenAI API key to Colab secrets as `OPENAI_API_KEY`
3. Run all cells in order
4. The notebook generates 50 emails across 10 batches and saves them to `Stage 2 - Synthetic Data.csv`

### Files

| File | Description |
|------|-------------|
| `Stage 2 - Synthetic Data Generation.ipynb` | Generates 50 synthetic emails using GPT-4o with the 17 real emails as stylistic references. Batches of 5 per API call to ensure variation. |
| `Stage 2 - Synthetic Data.csv` | The 50 synthetic emails used as the evaluation dataset in Stages 3 and 4. Columns: `email_id`, `email_text`. |

---

## Stage 3 — Extraction Pipeline

> Runs all 9 pipeline configurations (3 prompt variants × 3 model variants) across the 50 synthetic emails and saves the extracted field values to CSV.

### What happened in this stage

The extraction pipeline was built using the OpenAI API and structured into three steps for each email. First, the email text is passed to an LLM with a system prompt instructing it to extract the 12 fields and return a raw JSON object. Second, the JSON is parsed and any null fields are identified as missing. Third, if any fields are missing, a second LLM call generates a draft follow-up email requesting only the missing information.

Three prompt variants were designed to test the impact of increasing schema guidance:

| Prompt | Description |
|--------|-------------|
| **Prompt 1 — Zero-shot baseline** | Field names only, no definitions or examples |
| **Prompt 2 — Schema-guided** | Contextual introduction and field definitions |
| **Prompt 3 — Few-shot** | Same as Prompt 2 plus two annotated real email–output pairs as in-context examples |

Three model variants were selected to explore the performance-cost tradeoff:

| Model | Description |
|-------|-------------|
| `gpt-4o` | Flagship general-purpose model, used as the reference point |
| `gpt-4o-mini` | Cost-optimised variant, ~15–20× cheaper per token than GPT-4o |
| `gpt-4.1-nano` | Lowest cost model, optimised for structured extraction and low-latency tasks |

All 9 configurations were run across all 50 emails at temperature 0, producing 450 individual extraction results. Draft replies were generated using GPT-4o and Prompt 3 only, as reply quality was not part of the evaluation.

If a sender explicitly states they will provide a value later (e.g. *"I will send dietary requirements by Thursday"*), the model was instructed to extract the value as `"To be confirmed"` rather than null which is consistent with the ground truth annotation rules used in Stage 4.

### How to run

1. Upload `Stage 2 - Synthetic Data.csv` to your Colab session
2. Add your OpenAI API key to Colab secrets as `OPENAI_API_KEY`
3. Run all cells in order
4. Cell 9 runs a test across all 9 configurations on one email before the full run
5. Cell 10 processes all 450 combinations and saves to `Stage 3 - Extraction Results.csv`

### Files

| File | Description |
|------|-------------|
| `Stage 3 - Final Extraction Pipeline.ipynb` | Defines all 3 prompts and 3 models, runs the full 3×3 extraction matrix across 50 emails, detects missing fields, and generates draft follow-up replies. |
| `Stage 3 - Extraction Results.csv` | Output of the extraction pipeline. One row per email per configuration (450 rows). Columns: `prompt_version`, `model_name`, `email_id`, [12 field columns], `missing_fields`, `draft_reply`. |

---

## Stage 4 — Evaluation

> Evaluates all 9 pipeline configurations against the manually annotated ground truth using precision, recall, and F1 score.

### What happened in this stage

The 50 synthetic emails were manually annotated against the 12 extraction schema fields using the same annotation principles applied to the real emails. Ground truth annotations were stored in a flat long-format file with one row per field per email (600 rows total).

Two matching strategies were considered. Exact matching was explored first but rejected because slight differences between extracted values and ground truth such as verbosity differences or minor phrasing variations would cause semantically correct extractions to be matched as incorrect. A token-level overlap matching function was utilized instead, which removes stop words and scores a match as correct if the meaningful word overlap covers at least 50% of the ground truth words. A standard stop word library such as NLTK was not used as common stop word lists include semantically meaningful words in the hospitality context. For example *"not"* which appears in dietary requirements such as *"does not eat pork"* would incorrectly alter the token overlap calculation.

The evaluation follows these rules for each field-email pair:

- **GT has value, model extracted matching value** → True Positive (TP)
- **GT has value, model extracted non-matching value** → False Positive (FP)
- **GT has value, model returned null** → False Negative (FN)
- **GT is null, model returned null** → True Negative — skipped
- **GT is null, model extracted something** → False Positive (FP) 

True negatives are excluded to avoid artificially inflating scores by rewarding the model for correctly identifying absent fields. However cases where the GT is null but the model extracted a value are counted as false positives, as the model should not be extracting values for fields the email does not mention.

### How to run

1. Upload `Stage 3 - Extraction Results.csv` and `Stage 4 - Ground Truth Annotations for Evaluation.xlsx` to your Colab session
2. Run all cells in order
3. Cell 4 tests one configuration before the full evaluation
4. Cell 5 runs the full evaluation across all 9 configurations
5. Cell 6 saves `Stage 4 - Overall Evaluation.csv` and `Stage 4 - Field Level Evaluation.csv` and prints the 3×3 F1 matrix
6. Cell 7 runs a mismatch analysis showing false positive and false negative examples per field for further evaluation

### Files

| File | Description |
|------|-------------|
| `Stage 4 - Final Evaluation Pipeline.ipynb` | Loads extraction results and ground truth, applies token-level overlap matching, computes field-level and overall precision, recall, and F1 for all 9 configurations, and produces a mismatch analysis. |
| `Stage 4 - Ground Truth Annotations for Evaluation.xlsx` | Manually annotated ground truth for all 50 synthetic emails. Long format: one row per field per email (600 rows). Read with `header=3`, `sheet_name='Ground Truth Annotations'`. |
| `Stage 4 - Overall Evaluation.csv` | Overall precision, recall, and F1 for each of the 9 prompt-model configurations (9 rows). |
| `Stage 4 - Field Level Evaluation.csv` | Field-level precision, recall, and F1 for each field within each configuration (108 rows). |

---

## Stage 5 — Proof-of-Concept Tool

> This tool was built as an interactive demo of the pipeline. No installation required, just open the file in any modern browser.

### What this is

This is a HTML file that brings the extraction pipeline to life as something non-technical stakeholders can click through and test directly, rather than reading notebook output. It uses the best-performing configuration identified in Stage 4 (few-shot prompt + GPT-4o) and calls the OpenAI API directly from the browser.

The tool allows a user to:

1. Paste in a hospitality request email
2. Run extraction on the 12 structured fields in real time
3. View each field as colour-coded (extracted, missing, or "to be confirmed")
4. Automatically receive a draft follow-up email requesting only the missing fields
5. Browse a "History" tab showing results from the original 50-email experiment and filterable by how many fields are missing

### How to run

> **Note:** Clicking the file on GitHub displays the raw HTML source code, not the rendered tool. To use it, click the **download icon** (top right of the code view) to download the raw file, then open the downloaded file in your browser.

1. Open `Xebia Hospitality POC Tool.html` in any modern browser
2. Enter an OpenAI API key with access to `gpt-4o` in the top-right input field
3. Paste a hospitality request email into the text box and click Run extraction
4. Switch to the History tab to browse results from the original experiment

| File | Description |
|------|-------------|
| `Xebia Hospitality POC Tool.html` | Interactive proof-of-concept demo. Lets the user run a live extraction demonstrating the pipeline built visually. |
| `Xebia POC Tool Demo Video.mov` | Recording of proof-of-concept demo. Included to cater to readers without access to OpenAI API key. |


---

*BSc Business Analytics Thesis — Mikail Imran Shaukat*
