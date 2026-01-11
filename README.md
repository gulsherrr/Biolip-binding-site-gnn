# BioLiP Binding-Residue Prediction with a Lightweight GNN (Baseline + Feature Ablations)

This repository contains a small, honest baseline project that tries to predict **ligand-binding residues** on protein chains using **graph neural networks (GNNs)** built from protein 3D structure contacts.

It is not “state-of-the-art” and it does not claim to solve binding-site prediction. The goal is to build a **reproducible pipeline**, test a few **cheap features**, and learn what helps / what fails.

---

## What problem am I trying to solve?

Given a protein structure (PDB chain), I:
1) converted the chain into a **graph**
2) labeled each residue as **binding (1)** or **non-binding (0)** using BioLiP annotations
3) trained a **node classifier** (GraphSAGE) to predict which residues bind the ligand.

Binding residues tend to be rare → this is an **imbalanced classification** problem.

---

## Dataset and inputs (manageable subset)

Source: **BioLiP non-redundant (nr)** files + corresponding PDB structures.

I curated a laptop-friendly subset from BioLiP:
- subset_1200 (initial)
- then expanded/filtered labeled set used for training/evaluation:
  - **~837 graphs** successfully parsed + labeled (varies slightly by filtering)

Structures are downloaded as `.cif.gz` from RCSB PDB and parsed to extract Cα (alpha carbon) coordinates per residue.

---

## Graph construction

For each protein chain:

### Nodes
- **Residues** (one node per residue)

### Edges
- **Contact edges:** between residues with Cα distance < 8Å  
- **Sequence edges:** connect i ↔ i±1 (keeps chain continuity)

Optional: store edge distance.

---

## Labels (ground truth)

Each residue/node gets a binary label:
- `y[i] = 1` if residue i is annotated as binding-site in BioLiP  
- `y[i] = 0` otherwise

**Important note:** BioLiP labels can be tricky because residue numbering (PDB `resseq`) and sequence positions can differ. I had to debug label parsing and alignment; incorrect labeling can make the model learn nonsense. This repo includes the corrected labeling strategy used for the “fixed” dataset.

---

## Models

Baseline model:
- **GraphSAGE** node classifier
- **Loss:** `BCEWithLogitsLoss` with `pos_weight` to handle class imbalance
- Node prediction produces a probability per residue

---

## Features used

### Baseline features
- **AA identity** (amino acid index → embedding)
- **Degree** (number of neighbors / contacts)

### Day 16 feature set (feat_v1)
I added a small set of **computationally cheap** node features:

1) **Degree** (already present)  
2) **Mean neighbor distance** (average distance to connected neighbors)  
3) **Sequence position**: `i / (L-1)` where i is residue index and L is chain length  
4) **Physicochemical properties (tiny lookup table):**
   - Hydrophobicity (Kyte–Doolittle)
   - Charge (-1, 0, +1)
   - Polarity (0/1)

So the node feature vector becomes approximately:

`[aa_idx, degree, mean_dist, seq_pos, hydro, charge, polar]`

These features are meant to provide the model with:
- a hint about local geometry (mean_dist)
- a stabilization signal (seq_pos)
- chemistry bias (hydrophobicity/charge/polarity)

---

## Evaluation (what the metrics mean)

Because binding residues are rare, accuracy is not helpful. I used:

### Precision
Out of residues predicted as binding, how many are truly binding?
- High precision = few false positives

### Recall
Out of truly binding residues, how many did I catch?
- High recall = fewer missed binding residues

### F1 score
A balance between precision and recall:
- `F1 = 2 * (precision * recall) / (precision + recall)`
- F1 is useful when you want a single “balanced” number.

### AUPRC
Area Under the Precision–Recall Curve (Average Precision).
- Best overall metric for imbalanced problems.
- Higher AUPRC means the model ranks true binding residues above non-binding residues more consistently.

---

## Thresholding (maxF1, p15, p20)

The model outputs probabilities. To turn probabilities into 0/1 predictions, we must choose a threshold.

Threshold = 0.5 is arbitrary and usually wrong for imbalanced problems, so I used validation-driven rules:

### maxF1 threshold
Choose the threshold on the validation set that maximizes F1.
- Good when you want a balance of precision and recall.

### p15 / p20 thresholds (precision-target thresholds)
Pick the threshold that achieves at least:
- **p15:** precision ≥ 0.15
- **p20:** precision ≥ 0.20

While keeping recall as high as possible (subject to hitting that precision).

**Interpretation:**  
- p20 is stricter: fewer false positives, but usually lower recall.
- p15 is a softer compromise.

Reported test precision/recall/F1 at:
- maxF1 threshold
- p15 threshold
- p20 threshold

---

## Honest status / current shortcomings

This project is a functional baseline, not a finished predictor.

What works:
- We can reliably build contact graphs from mmCIF
- Labels are aligned after fixes
- The model learns non-random structure (predictions often cluster spatially)
- Adding cheap features (Day 16 feat_v1) improves some metrics modestly

Main limitations:
- **Binding labels are noisy/partial**: BioLiP marks specific contact residues, but a binding pocket is a region; the model may predict “near-pocket” residues that aren’t labeled as positives.
- **Imbalance is severe**: positives are a small fraction of residues; precision remains difficult.
- **Edge distances aren’t fully exploited** in the simplest GraphSAGE baseline; distance-aware layers may help.
- Performance is still far from “useful for real prediction” without further work.

---

## Reproducibility

Evaluated using multiple random splits (seeds) to reduce “lucky split” effects.

Seeds used commonly:
`[1, 7, 42, 123, 999]`

---

## Next steps (future improvements)
- Distance-aware message passing (edge weights / distance buckets)
- Better “surface/pocket” features (exposure proxies, local curvature)
- Hard-negative sampling (near-pocket decoys) to reduce false positives
- Top-K evaluation (Precision@K) as a more practical metric for pocket discovery

---

## Disclaimer
This is an educational/research prototype. Must not be used for real biological conclusions without validation.
This project was made with the help of Artifical Intelligence softwares.
