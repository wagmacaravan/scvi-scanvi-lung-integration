# Batch integration and cell-type label transfer of lung scRNA-seq with scVI / scANVI

Training and evaluating deep generative models for single-cell RNA-seq integration and
annotation. A variational autoencoder (**scVI**) is trained to integrate 32,472 cells across
16 experimental batches of the [scIB lung atlas benchmark](https://theislab.github.io/scib-reproducibility/),
then a semi-supervised model (**scANVI**) is used for cross-batch cell-type label transfer.
Integration quality is quantified against a PCA baseline with the `scib-metrics` suite, and
label transfer is evaluated on a held-out query set with a confidence-based triage analysis.

This project moves beyond *running* single-cell pipelines to *training and evaluating* the
models underneath them — reading loss curves, choosing and defending hyperparameters,
benchmarking against baselines, and characterizing failure modes honestly.

## Pipeline

1. **Load & QC** — verify the count matrix is raw integers (required by scVI's negative-binomial
   likelihood) and confirm batch/biology are separable before training.
2. **Train scVI** — a 30-dimensional variational autoencoder; read the train/validation ELBO
   curve for convergence and overfitting.
3. **Evaluate integration** — UMAP on the latent space, then quantify batch correction vs.
   bio-conservation against a PCA baseline using `scib-metrics`.
4. **Train scANVI** — warm-started from scVI with cell-type supervision; re-benchmark three ways.
5. **Held-out label transfer** — hide labels on a 20% query set, predict them, and evaluate with
   overall accuracy, a per-cell-type breakdown, confidence-based error triage, and a confusion matrix.

## Key results

| Embedding | Batch correction | Bio-conservation | Total |
|-----------|:----------------:|:----------------:|:-----:|
| PCA (baseline) | 0.29 | 0.67 | 0.52 |
| scVI | 0.61 | 0.65 | 0.63 |
| scANVI | 0.58 | 0.71 | 0.66 |

*Exact metric values vary by ~±0.01 between runs due to non-deterministic GPU operations; the
qualitative comparisons below are stable.*

- **scVI** more than doubled the batch-correction score over uncorrected PCA (0.29 → 0.61) at a
  ~1-point bio-conservation cost — it removed batch variance without materially sacrificing
  biological structure.
- **scANVI** recovered and exceeded bio-conservation (→ 0.71, beating even PCA) for a small
  batch-correction trade, giving the best overall integration. The gain concentrated in
  clustering-recovery metrics (KMeans NMI 0.66 → 0.76), consistent with label supervision
  sharpening cell-type boundaries.
- **Label transfer** reached **91.4% accuracy** on 6,490 held-out cells. Rarity was not the
  failure mode — query Ionocytes were labeled correctly 11/12 times (92%) despite only ~34
  labeled Ionocytes in the reference, because their CFTR-high signature is distinctive. The
  lowest accuracies were *common* types with close neighbors (Basal 2 at 88%).
- **Errors followed biological structure**, not random scatter: the clearest systematic confusion
  was Basal 2 → Basal 1 (~11%, shared lineage), with Secretory acting as a mild sink for several
  airway epithelial types.
- **Prediction confidence** carried real signal (correct calls averaged 0.98, wrong calls 0.83)
  but the distributions overlapped — no threshold cleanly separates right from wrong, consistent
  with known neural-network overconfidence. Confidence is useful for *prioritizing* manual review
  (flagging the lowest-confidence ~10% catches roughly half the errors), not for certifying
  high-confidence calls.

## Data

The [scIB lung atlas integration benchmark](https://theislab.github.io/scib-reproducibility/):
32,472 cells × 2,000 highly variable genes, 16 batches, 17 curated cell types, with raw counts in
`layers["counts"]`. The notebook downloads it automatically on first run (~628 MB).

## Running it

Designed for **Google Colab** with a single **T4 GPU** (Runtime → Change runtime type → T4).
scVI training takes ~20 minutes; scANVI refinement ~2 minutes each.

```bash
pip install -r requirements.txt
```

Then open `scVI_scANVI_lung_integration.ipynb` and run top to bottom. The notebook is committed
without outputs; run it to regenerate the figures and tables inline.

## Methods & references

- Lopez et al. (2018). *Deep generative modeling for single-cell transcriptomics.* Nature Methods 15, 1053–1058. (scVI)
- Xu et al. (2021). *Probabilistic harmonization and annotation of single-cell transcriptomics data with deep generative models.* Molecular Systems Biology 17(1):e9620. (scANVI)
- Luecken et al. (2022). *Benchmarking atlas-level data integration in single-cell genomics.* Nature Methods 19, 41–50. (scIB metrics)
- Guo et al. (2017). *On Calibration of Modern Neural Networks.* ICML. (neural-network overconfidence)
