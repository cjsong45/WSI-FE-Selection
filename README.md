# SAM-Cluster and SAM-LP: Structure-Aware Evaluation Metrics for Selecting WSI Feature Extractors without MIL Training
This repository contains the implementation of "<em>SAM-Cluster and SAM-LP: Structure-Aware Evaluation Metrics for Selecting WSI Feature Extractors without MIL Training</em>," accepted to MICCAI 2026. It is a fork of the official repository, [juhyeon-ai/WSI-FE-Selection](https://github.com/juhyeon-ai/WSI-FE-Selection).

<p align="center">
    <a href="docs/paper.pdf"><img src="https://img.shields.io/badge/Paper-MICCAI'26-05185E.svg?style=for-the-badge" alt="Paper"/></a>
    <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache%202.0-0A31C0.svg?style=for-the-badge" alt="License"/></a>
    <img src="https://img.shields.io/badge/Python-3.10%20%7C%203.11-9DB1FA.svg?style=for-the-badge" alt="Python"/>
</p>

<p align="center">
    🖇️&nbsp;<a href="#-overview">Overview</a>
    | 👀&nbsp;<a href="#-installation">Installation</a>
    | 📚&nbsp;<a href="#-dataset">Dataset</a>
    | 🚀&nbsp;<a href="#-usage">Usage</a>
    | 📊&nbsp;<a href="#-results">Results</a>
    | 📝&nbsp;<a href="#-citation">Citation</a>
    | 🙏&nbsp;<a href="#-acknowledgements">Acknowledgements</a>
</p>

## 🖇️ Overview
Choosing a feature extractor (FE) is one of the most consequential decisions in a whole-slide image (WSI) pipeline. On Camelyon16, replacing an ImageNet ResNet50 with Virchow2 raises balanced accuracy from 76.6% to 98.0%, and no single foundation model is best on every cohort. Finding the right FE usually means training and comparing dozens of Multiple Instance Learning (MIL) models. **WSI-FE-Selection** ranks candidate pathology FEs **before any MIL training**. It scores pre-extracted features from a small random subset of slides (about 30), using the Segment Anything Model (SAM) as a zero-shot proxy for tissue structure.

<p align="center">
  <img src="./assets/overview.png" width="95%">
</p>

**WSI-FE-Selection** consists of three main components:
1. **SAM regions as pseudo-structural labels**: SAM ViT-H is run once per slide on a thumbnail, and its zero-shot masks are aligned to the patch grid. The masks follow the morphological boundaries a pathologist would annotate, so patches are no longer treated as i.i.d. samples ([`preprocessing/`](preprocessing)).
2. **SAM-Cluster (unsupervised)**: compares intra-region cohesion with a relaxed inter-region separation, in which region pairs whose centroids are more similar than a threshold τ are excluded to absorb SAM over-segmentation. The score is $1 - S_{\text{inter}} / S_{\text{intra}}$ ([`suitability_score/sam_cluster.py`](suitability_score/sam_cluster.py)).
3. **SAM-LP (supervised)**: rebuilds each slide from its region prototypes instead of mean pooling. One EM step fits a linear classifier on the aggregated embeddings and sharpens the region weights toward label-predictive prototypes (sharpness β). The score is the resulting log-likelihood ([`suitability_score/sam_lp.py`](suitability_score/sam_lp.py)).

### Highlights
- Closest agreement with MIL rankings: mean Spearman ρ of 0.535 (SAM-Cluster) and 0.626 (SAM-LP) against ground truth from 10 FEs × 9 aggregators × 8 datasets.
- Reliable shortlists: the top-5 candidates of SAM-LP contain the best FE on all 8 datasets.
- Low cost: about 30 slides, no aggregator training, and SAM once per slide (under 1% of a single feature extraction). A candidate is scored in about 6–7 s once its features exist.
- With or without labels: seven metrics are implemented (SAM-Cluster, SAM-LP and five baselines), and five complementary ones are reported by default.

## 👀 Installation
We used Python 3.10 and 3.11. Scoring existing features runs on CPU with a few packages.
```
git clone https://github.com/cjsong45/WSI-FE-Selection.git
cd WSI-FE-Selection
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```
For the full WSI workflow (Linux, CUDA GPU recommended), install a compatible PyTorch + torchvision pair, then install the pinned TRIDENT and SAM revisions:
```
bash scripts/setup_dependencies.sh
```
Download the SAM ViT-H checkpoint and request access to any gated encoders before processing real slides ([workflow guide](docs/quickstart.md#1-clone-and-install)).

## 📚 Dataset
The ground-truth FE rankings come from exhaustive MIL training: every FE × aggregator combination below, averaged over five seeds (450 MIL runs per dataset).

| | |
| :-- | :-- |
| **Datasets (8)** | Subtyping (balanced accuracy): Camelyon16 · BRACS · UBC-OCEAN · TCGA-GLIOMA · TCGA-NSCLC · TCGA-RCC · Histai-skin-b1; Grading (quadratic weighted kappa): PANDA |
| **Feature extractors (10)** | CONCH v1 · CONCH v1.5 · Hibou-B · Lunit-ViT · MUSK · Phikon-v2 · ResNet50 · UNI v1 · UNI v2 · Virchow2 |
| **MIL aggregators (9)** | Mean pooling · AB-MIL · DS-MIL · CLAM · TransMIL · DTFD-MIL · WiKG · RRT-MIL · ILRA |

The MIL training code and data splits are in [`Benchmark-MIL/`](Benchmark-MIL), and the per-dataset results used as reference rankings are in [`Ground Truth/`](Ground%20Truth). The slides themselves are not redistributed; download them from each dataset's official source.

For your own cohort, describe the slides in a manifest like [`examples/slides.csv`](examples/slides.csv):
```
slide_id,wsi_path,label,split,patient_id
slide_001,/data/slides/slide_001.svs,normal,train,patient_001
slide_002,/data/slides/slide_002.svs,tumor,train,patient_002
```

## 🚀 Usage
**CPU demo** (synthetic features, no downloads; shows the input and output format only):
```
python scripts/make_demo_features.py
python -m suitability_score --feature-root runs/demo/features --output-dir runs/demo/scores
```

**Score existing feature files.** Arrange them as `features/<encoder>/<slide_id>.h5` with the same slides under every encoder. Each file holds `features`; supervised metrics also need `label`, and SAM metrics need patch-aligned `sam_region` IDs ([metric reference](suitability_score/README.md)).
```
python -m suitability_score --feature-root /data/features --output-dir runs/my_scores
```

**Start from WSIs.** Set `manifest`, `work_dir`, the candidate `encoders` and `n_slides` in [`configs/quickstart.json`](configs/quickstart.json). The workflow samples a label-balanced cohort, extracts TRIDENT features on one shared patch grid, runs SAM once per slide, merges everything into `.h5` files, and scores the candidates.
```
python -m preprocessing --config configs/quickstart.json --dry-run
python -m preprocessing --config configs/quickstart.json
```
Results are written to `runs/quickstart/scores/`: `scores.csv` (raw score per metric and FE, higher is better), `ranking.csv` (per-metric ranks and mean rank, lower is better), and `coverage.csv` (SAM coverage per slide). Metric defaults (τ = 0.95, β = 5, one EM step, 128-d projection) are in [`configs/suitability.json`](configs/suitability.json); pass a copy with `--config` to change them.

## 📊 Results
Spearman rank correlation (ρ) between each metric's FE ranking and the ground-truth MIL ranking, averaged over 8 datasets (each entry averages five random 30-slide subsets).

| Setting | Metric | Mean ρ |
| :-- | :-- | :--: |
| Unsupervised | Effective Dimension | 0.1318 |
| | NESum | 0.4439 |
| | Self-Cluster | 0.4364 |
| | **SAM-Cluster (ours)** | **0.5348** |
| Supervised | LogME | 0.2342 |
| | Linear Probing | 0.5485 |
| | **SAM-LP (ours)** | **0.6258** |

SAM-Cluster is higher than Self-Cluster and NESum on all 8 datasets (Wilcoxon signed-rank *p* = 0.0078). SAM-LP matches or exceeds Linear Probing on 6 of 8 datasets, with the largest gain on TCGA-NSCLC (0.6364 → 0.9515). Per-dataset results and standard deviations are in [`docs/paper_details.md`](docs/paper_details.md) and [`docs/results/table1.csv`](docs/results/table1.csv).

## 📝 Citation
If you find our work useful, please cite:
```
@inproceedings{kim2026samcluster,
    title = "SAM-Cluster and SAM-LP: Structure-Aware Evaluation Metrics for Selecting WSI Feature Extractors without MIL Training",
    author = "Kim, Juhyeon and Ssemakula, Paul and Song, Chanjae and Yi, Mun Yong",
    booktitle = "Medical Image Computing and Computer Assisted Intervention (MICCAI)",
    year = "2026"
}
```

## 🙏 Acknowledgements
This work was supported by the National Research Foundation of Korea (NRF) grant funded by the Korea government (MSIT) (No. RS-2022-NR068758) and by the Seegene Medical Foundation, Republic of Korea. The preprocessing workflow builds on **TRIDENT** and **Segment Anything**.

## License
Original code is released under the Apache 2.0 License. See [LICENSE](LICENSE) for details. Third-party software and weights keep their own terms (the pinned TRIDENT revision restricts commercial use, and several encoders are gated); see [`docs/third_party.md`](docs/third_party.md).
