[README.md](https://github.com/user-attachments/files/32292419/README.md)
# De-Novo-Gen# AI-Driven De Novo Drug Design Against TAK1

A reproducible, Colab-oriented computational workflow for **de novo small-molecule generation, chemical-space analysis, candidate prioritization, and structure-based docking against Transforming Growth Factor Beta-Activated Kinase 1 (TAK1/MAP3K7)**.

The project is organized as a sequential five-notebook pipeline. The workflow deliberately separates:

1. **Reference-data preparation**
2. **Chemical characterization of known TAK1 inhibitors**
3. **Molecular generation**
4. **Chemical-space triage and candidate prioritization**
5. **Structure-guided docking**

A key design principle is that **generation and chemical-space triage do not use the TAK1 protein structure**. The protein structure and experimentally observed binding pocket are introduced only in Notebook 5 for structure-guided evaluation.

---

## Workflow

```text
ChEMBL TAK1 bioactivity data
          │
          ▼
01. Preprocess ChEMBL TAK1 bioactivity data
          │
          ▼
02. Exploratory analysis of known TAK1 inhibitors
          │
          ├── physicochemical characterization
          ├── Bemis–Murcko scaffolds
          ├── Morgan fingerprints
          └── chemical-space analysis
          │
          ▼
03. De novo molecular generation
          │
          ├── Baseline A: unconditional MolGPT
          ├── Experiment 1: scaffold-seeded MolGPT
          └── Experiment 2: attachment-aware scaffold decoration
          │
          ▼
04. Chemical-space analysis & candidate prioritization
          │
          ├── quality filtering
          ├── exact novelty
          ├── TAK1 chemical-space similarity
          ├── PCA
          ├── diversity analysis
          └── diversity-aware candidate selection
          │
          ▼
05. TAK1 structure-guided docking
          │
          ├── TAK1 PDB 4O91 preparation
          ├── NG2 redocking control
          ├── ligand preparation
          └── AutoDock Vina docking
          │
          ▼
Pose / interaction analysis
```

---

# Project Objective

The objective is to explore whether pretrained molecular-generation methods can produce **new chemical structures that remain chemically connected to known TAK1 inhibitor space**, while providing a practical computational route toward structure-based evaluation.

The workflow is not intended to claim biological activity from generation, chemical similarity, or docking alone. Each stage has a distinct role:

| Stage | Main question |
|---|---|
| Notebook 1 | How can the TAK1 bioactivity data be cleaned into a usable reference set? |
| Notebook 2 | What does the known TAK1 inhibitor chemical space look like? |
| Notebook 3 | Can different molecular-generation strategies produce valid novel structures? |
| Notebook 4 | Which generated molecules are chemically suitable and diverse enough to evaluate further? |
| Notebook 5 | How do the shortlisted molecules interact with the experimentally observed TAK1 binding pocket in docking? |

---

# Notebook 1 — Preprocess ChEMBL TAK1 Bioactivity Data

**File:** `01_Preprocess_ChEMBL_TAK1_Bioactivity_Data.ipynb`

This notebook converts the raw ChEMBL TAK1 bioactivity table into a standardized reference dataset for subsequent cheminformatics and molecular-generation steps.

### Processing

The workflow:

- loads the raw ChEMBL dataset;
- inspects bioactivity types, relations, units, missing values, and SMILES;
- retains **IC50 measurements ≤ 1000 nM** with:
  - `Standard Type = IC50`
  - `Standard Units = nM`
  - `Standard Relation` equal to `=` or `<`;
- removes records without SMILES;
- removes duplicate SMILES;
- canonicalizes SMILES using RDKit;
- removes molecules that cannot be parsed by RDKit;
- removes duplicate canonical molecules.

### Main output

```text
TAK1_clean.csv
TAK1_clean_canonical.csv
```

`TAK1_clean_canonical.csv` is the principal reference dataset used by the downstream notebooks.

---

# Notebook 2 — Exploratory Analysis of TAK1 Inhibitors

**File:** `02_Exploratory_Analysis_of_TAK1_Inhibitors.ipynb`

This notebook characterizes the cleaned TAK1 inhibitor reference space before molecular generation.

### Molecular descriptors

RDKit is used to calculate:

- Molecular Weight (MW)
- LogP
- Topological Polar Surface Area (TPSA)
- Hydrogen-bond donors (HBD)
- Hydrogen-bond acceptors (HBA)
- Rotatable bonds
- Aromatic rings
- Heavy atoms
- Quantitative Estimate of Drug-likeness (QED)

### Structural analysis

The notebook also performs:

- Bemis–Murcko scaffold extraction;
- scaffold-frequency analysis;
- Morgan fingerprint generation;
- pairwise Tanimoto similarity analysis;
- PCA-based visualization of descriptor-defined chemical space.

This reference-space characterization provides the chemical context used later for scaffold selection, similarity analysis, and candidate prioritization.

---

# Notebook 3 — De Novo Molecular Generation

**File:** `03_DeNovo_Generation.ipynb`

Notebook 3 explores **three complementary generation strategies**.

## Generation strategy A — Unconditional MolGPT

A pretrained MolGPT model is used without supplying a TAK1-derived scaffold.

**Model checkpoint:**

```text
jonghyunlee/MolGPT_pretrained-by-ZINC15
```

The model is used for inference only; no fine-tuning is performed.

Current generation settings include:

- 100 unconditional samples
- temperature = 1.0
- top-k = 50
- top-p = 0.95
- maximum generation length = 64 new tokens

Generated SMILES are parsed and canonicalized with RDKit.

---

## Generation strategy B — BM-scaffold seeded MolGPT

The cleaned TAK1 inhibitor dataset is decomposed into **Bemis–Murcko scaffolds**.

In the current run:

- 104 unique BM scaffolds were identified;
- 20 structurally diverse scaffolds were selected using Morgan fingerprints and a deterministic MaxMin-style selection procedure.

Each selected scaffold is then supplied to MolGPT as a SMILES prefix.

Current settings:

- 20 selected scaffolds
- 10 generations per scaffold
- 200 raw seeded generations

Importantly, this is **ordinary autoregressive SMILES continuation** rather than explicit atom-by-atom substituent placement.

Scaffold retention is subsequently measured rather than assumed.

---

## Generation strategy C — Attachment-aware scaffold decoration

The third strategy uses the same 20 BM scaffolds as fixed cores.

Instead of asking MolGPT to generate incomplete fragments, RDKit is used to:

1. identify known TAK1 molecules containing each selected scaffold;
2. perform R-group decomposition;
3. identify observed attachment positions;
4. label those positions;
5. collect substituents separately for each attachment label;
6. recombine substituents only at matching labelled positions;
7. sanitize and deduplicate the resulting products.

This produces new **scaffold–substituent combinations** while maintaining chemically explicit attachment relationships.

Theoretical combinations can exceed the number actually generated; the notebook limits generation to a maximum of 50 combinations per scaffold.

### Generation-stage results from the current notebook run

| Population | Raw | Valid unique |
|---|---:|---:|
| Unconditional | 100 | 100 |
| Experiment 1 — Seeded | 200 | 34 |
| Experiment 2 — Decorated | 68 | 68 |

The current seeded run showed a measured scaffold-retention rate of **1.0** for the valid generated molecules.

### Outputs

```text
MolGPT_baseline_unconditional.csv
MolGPT_experiment1_seeded.csv
MolGPT_experiment2_decorated.csv
TAK1_diverse_20_BM_scaffolds.csv
TAK1_scaffold_attachment_summary.csv
TAK1_labelled_rgroup_library.csv
```

---

# Notebook 4 — Chemical-Space Analysis & Candidate Prioritization

**File:** `04_Chemical_Space_Analysis_and_Docking_Prioritization.ipynb`

Notebook 4 does **not** perform docking.

Its purpose is to determine which generated molecules are sufficiently:

- chemically valid;
- drug-like by simple heuristic criteria;
- novel relative to the known TAK1 reference set;
- chemically connected to TAK1 inhibitor space;
- diverse enough to avoid selecting redundant analogues.

## Quality filtering

The notebook applies the following hard filters:

```text
MW   ≤ 500
LogP ≤ 5
HBD  ≤ 5
HBA  ≤ 10
PAINS = absent
```

These are generation-stage triage criteria, not predictors of TAK1 activity.

---

## Exact novelty

A generated molecule is considered exactly novel when its canonical SMILES is absent from the known TAK1 reference set.

This measures **exact molecular novelty**, not scaffold novelty or synthetic novelty.

---

## Chemical similarity to known TAK1 inhibitors

Morgan fingerprints are used to calculate Tanimoto similarity between each generated molecule and the known TAK1 inhibitor reference set.

For every generated molecule, the **maximum Tanimoto similarity** to the reference set is retained.

This metric is interpreted as proximity to known TAK1 chemical space rather than as an affinity prediction.

---

## Chemical-space visualization

Morgan fingerprints are projected into two dimensions using PCA.

The analysis compares:

- known TAK1 inhibitors;
- unconditional MolGPT molecules;
- scaffold-seeded molecules;
- decorated molecules.

Additional analyses include:

- physicochemical-property distributions;
- scaffold contribution;
- internal structural diversity.

---

## Candidate prioritization

A transparent heuristic priority score combines:

- exact novelty;
- similarity to known TAK1 chemical space;
- QED;
- preference for moderate molecular weight;
- preference for moderate LogP.

The weighting used in the notebook is:

```text
25%  Novelty
25%  TAK1-space similarity
25%  QED
12.5% MW preference
12.5% LogP preference
```

Quality filtering acts as a **gate**, rather than contributing an additional reward.

The notebook then applies diversity-aware selection using:

```text
10 candidates per generation category
Tanimoto similarity cutoff = 0.80
```

The target docking panel therefore contains up to:

```text
10 unconditional
10 seeded
10 decorated
----------------
30 candidates
```

### Current Notebook 4 run

| Population | Valid unique | Quality pass | Exact novel | Mean max TAK1 similarity |
|---|---:|---:|---:|---:|
| Unconditional | 100 | 100 | 100 | 0.194 |
| Experiment 1 — Seeded | 34 | 16 | 34 | 0.449 |
| Experiment 2 — Decorated | 68 | 63 | 57 | 0.629 |

These values describe the generated chemical populations and **should not be interpreted as evidence of TAK1 activity**.

### Outputs

```text
Notebook4_all_generated_with_analysis.csv
Notebook4_docking_candidates.csv
Notebook4_population_summary.csv
```

---

# Notebook 5 — TAK1 Structure-Guided Docking

**File:** `05_TAK1_Docking_4O91_AutoDockVina.ipynb`

Notebook 5 introduces the protein structure and experimentally observed binding site.

The selected receptor is:

```text
TAK1 — PDB 4O91
```

The structure contains the crystallographic inhibitor **NG2/NG25** in the **DFG-out** conformation.

## Docking workflow

```text
Notebook 4 candidate set
        ↓
TAK1 4O91 structure
        ↓
protein-chain identification
        ↓
crystallographic NG2 identification
        ↓
protein-only receptor preparation
        ↓
binding-box definition from NG2 coordinates
        ↓
NG2 redocking validation control
        ↓
generated-ligand 3D preparation
        ↓
PDBQT preparation
        ↓
AutoDock Vina 1.2.7
        ↓
docking scores + poses
```

---

## Receptor preparation

The notebook:

- downloads the experimental 4O91 structure;
- identifies protein chain A;
- identifies NG2 as the crystallographic ligand;
- separates the protein receptor from the ligand;
- prepares the receptor using Meeko.

The docking box is calculated from the heavy-atom coordinates of the crystallographic NG2 ligand rather than from arbitrary hard-coded coordinates.

Current box:

```text
center_x = -5.774
center_y = -49.551
center_z = -16.372

size_x = 31.031
size_y = 18.926
size_z = 20.844
```

---

## NG2 redocking control

The crystallographic ligand NG2 is redocked before evaluating generated molecules.

This provides a protocol-level control: if the known ligand cannot reproduce a plausible pose in its experimentally observed pocket, generated-molecule docking should not be interpreted without investigating the setup.

Current redocking output:

```text
Best Vina affinity = -12.36 kcal/mol
```

The notebook intentionally does not claim a quantitative redocking RMSD because reliable RMSD requires defensible atom-to-atom correspondence. The redocked pose is exported for direct visual inspection.

---

## Generated-ligand preparation

Each candidate follows:

```text
SMILES
  ↓
RDKit molecule
  ↓
add hydrogens
  ↓
ETKDG 3D embedding
  ↓
light UFF optimization
  ↓
SDF
  ↓
Meeko
  ↓
PDBQT
```

This is a practical ligand-preparation step. It is **not molecular dynamics** and does not calculate a physical binding free energy.

---

## AutoDock Vina docking

All candidates are evaluated using the same:

- TAK1 receptor;
- docking box;
- Vina scoring protocol;
- exhaustiveness;
- number of output poses.

Within this experiment, a more negative Vina affinity represents a more favorable **predicted docking score**.

A Vina score should not be interpreted as an experimentally measured binding free energy.

### Current docking run

Notebook 4 supplied **30 candidates**.

The current Notebook 5 run produced:

```text
29 successful docking runs
1 ligand-preparation/docking failure
```

The failed candidate was `Dock_002`, due to 3D embedding / ligand preparation failure.

The successful candidates produced docking poses and Vina scores for subsequent pose and interaction analysis.

### Main outputs

```text
Notebook5_docking_results.csv
Notebook5_successful_docking_results.csv
Notebook5_ligand_preparation.csv
Notebook5_redocking_summary.csv

NG2_redocked.pdbqt
NG2_redocked.sdf

docking_results/
    *.pdbqt
    *.log
```

---

# Why the Workflow Is Structured This Way

A central methodological distinction in this project is:

### Generation

Notebook 3 asks:

> Can we generate chemically valid molecules using different generation strategies?

### Chemical-space triage

Notebook 4 asks:

> What chemical space did those strategies produce, and which molecules are sufficiently novel, chemically connected to TAK1 inhibitor space, drug-like by simple filters, and structurally diverse enough to evaluate?

### Structure-guided evaluation

Notebook 5 asks:

> How do the shortlisted molecules fit into the experimentally observed TAK1 binding pocket?

This separation prevents a chemical-space priority score from being confused with a protein-binding prediction.

---

# Important Limitations

This project is a **computational hypothesis-generation workflow**, not an experimental validation pipeline.

The following distinctions are important:

- ChEMBL IC50 filtering defines the reference dataset but does not make the measurements directly comparable across all experimental contexts.
- Exact novelty means absence from the reference set at the canonical-SMILES level; it does not establish synthetic novelty.
- Tanimoto similarity measures fingerprint-based chemical proximity; it does not predict TAK1 potency.
- Lipinski/QED/PAINS criteria are heuristic chemical-quality filters.
- PCA is a visualization method and does not represent the full molecular chemical space.
- The Notebook 4 priority score is a transparent triage heuristic, not a machine-learned activity model.
- Experiment 1 uses SMILES-prefix continuation, so scaffold retention is measured after generation.
- Experiment 2 generates new combinations through RDKit R-group recombination rather than through a generative model discovering attachment points.
- AutoDock Vina scores are docking scores, not experimentally measured affinities or rigorous binding free energies.
- A favorable docking score alone does not establish biological activity.
- The current 4O91 analysis focuses on a DFG-out TAK1 conformation.
- Detailed binding-pose and interaction analysis is a subsequent step and should examine interactions, steric compatibility, and consistency with known TAK1 structural features.

---

# Computational Environment

The notebooks are designed primarily for **Google Colab** with Google Drive used for persistent project files.

Core software and libraries include:

- Python
- RDKit
- NumPy
- Pandas
- Matplotlib
- Seaborn
- Scikit-learn
- PyTorch
- Hugging Face Transformers
- Hugging Face Hub
- AutoDock Vina 1.2.7
- Meeko
- Biopython
- Gemmi
- Py3Dmol

Notebook 5 uses the **precompiled AutoDock Vina 1.2.7 Linux executable** rather than the Python `vina` package, allowing the workflow to run in the current Colab Python 3.13 environment.

---

# Reproducibility

The notebooks use fixed random seeds where appropriate:

```text
SEED = 42
```

Generated molecules retain provenance information such as:

- generation category;
- scaffold ID;
- scaffold SMILES;
- attachment information for decorated molecules;
- candidate ID in the docking panel.

This makes it possible to trace a docking candidate back through the pipeline to its generation strategy and scaffold origin.

---

# Repository Structure

A suggested repository organization is:

```text
.
├── 01_Preprocess_ChEMBL_TAK1_Bioactivity_Data.ipynb
├── 02_Exploratory_Analysis_of_TAK1_Inhibitors.ipynb
├── 03_DeNovo_Generation.ipynb
├── 04_Chemical_Space_Analysis_and_Docking_Prioritization.ipynb
├── 05_TAK1_Docking_4O91_AutoDockVina.ipynb
│
├── data/
│   └── TAK1_clean_canonical.csv
│
├── outputs/
│   ├── MolGPT_baseline_unconditional.csv
│   ├── MolGPT_experiment1_seeded.csv
│   ├── MolGPT_experiment2_decorated.csv
│   ├── Notebook4_all_generated_with_analysis.csv
│   ├── Notebook4_docking_candidates.csv
│   └── Notebook4_population_summary.csv
│
└── docking/
    ├── Notebook5_docking_results.csv
    ├── Notebook5_successful_docking_results.csv
    ├── Notebook5_ligand_preparation.csv
    ├── Notebook5_redocking_summary.csv
    └── docking_results/
```

Large raw datasets, generated pose files, and other computationally heavy artifacts may be better stored separately rather than committed directly to GitHub.

---

# Next Step

The natural continuation of Notebook 5 is **structure-based pose and interaction analysis**.

For the strongest computationally prioritized candidates, the next analysis should examine:

- binding-pocket occupancy;
- hinge-region interactions;
- interactions with the DFG region;
- hydrogen bonds and other non-covalent contacts;
- steric clashes;
- consistency of binding orientation;
- comparison with the crystallographic NG2 pose.

A later cross-conformation analysis can also use **TAK1 PDB 4L53 (DFG-in)** to investigate whether promising candidates show consistent structural behavior across TAK1 conformational states.

---

## Disclaimer

This repository describes a computational drug-design workflow for research and hypothesis generation. Generated structures, chemical-space scores, and docking scores are **computational predictions** and do not constitute evidence of experimental TAK1 inhibition, binding affinity, efficacy, safety, or drug-likeness.
