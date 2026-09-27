# Quantitative Affinity Data at Scale: Addressing the Data Bottleneck in AI-Enabled Protein Design

**Speaker:** Natasha Seelam (A-Alpha Bio)
**Venue:** Boston Protein Design and Modeling Club

## Acknowledgements

This repository contains technical notes, pipeline breakdowns, and architectural benchmarks summarized from a presentation delivered at the **Boston Protein Design and Modeling Club**. 
The entire work featured in this repository is executed and validated by speaker **Natasha Seelam** and her team at **A-Alpha Bio**.
This project serves as a structured technical reference of their presentation for the computational protein biology community.

## Background

    Antibody design is bottlenecked by **data, not architecture**: the CDR design space is ~10³⁰ sequences, but the public structural/affinity corpus is small (~10K antibody x antigen structures in SAbDab-2 database, ~1K affinity labels across AB-Bind/SKEMPI/ANTIPASTI).
    Structure-prediction confidence (ipTM, ipSAE, pLDDT, Boltz-2 confidence) **does not correlate with binding** - the model might have nearly identical confidence scores for a real 17 nM binder and a >10 µM non-binder. 
    A-Alpha Bio's fix: uses their high throughput assay: **AlphaSeq**, to generate paired in silico/in vitro labels at scale, across their three workstreams.


## AlphaSeq: Yeast-display library

    Separate antigen and antibody libraries are generated; if specific binding happens, it drives cellular fusion, which is further quantified via NGS. This provides a parallel, quantitative K_D affinity readout

------------------------------------------------------------------------

## 1. Pseudo-structures (SEPIA → ABACUS)

**Problem** 
    The PDB only contains the true binding complexes but no negative examples, which is the reason why structure models never learn to spot "plausible but non-binding" interactions, meaning they cannot screen for them.
    ipSAE vs. AlphaSeq affinity confusion matrix of generated designs reveals a True Positive rate of 17.2%, a False Positive rate of 19.6%, False Negative of 3.8%, True Negative of 59.4%, which indicates ~1 in 5 "confident" predictions is a false positive (hard negative).

**Pseudo-structure**: in silico structure prediction along with AlphaSeq validation to have every candidate real K_D value.

**SEPIA (Synthetic Epitope Atlas)**: 
    Instead of matching one antibody with one natural epitope, multiple synthetic epitopes that mimic the natural epitope and are recognized by that single antibody were created to massively expand structural and interface diversity per antibody.

- pipeline: 
    VHH Structure -> RFDiffusion (generation) -> ProteinMPNN (inverse fold) -> Sequence naturalness filter -> Boltz2 (refolding) -> TM-score & confidence filter

- yield
    928 high-quality hits were obtained.
    On-target hits (~4% overall hit rate) validated with a KD < 1 µM and no significant off-target binding. 
    Notably, 71% of the tested VHHs (34 out of 48) yielded at least one high-quality hit, and all validated hits were multi-specific.

**Validation: SEPIA is real signal and not noise**
    To ensure the synthetic data provides genuine biological signal rather than artifacts or noise.
    - An antibody-blind structure language model (NTX) was used to prevent data leakage. NTX was pre-trained only on monomers from the AlphaFold Database (AFDB) and explicitly excluded all structural complexes from the PDB and SAbDab.
    - Breaking Data Plateaus: Adding SEPIA data significantly enhanced learning efficiency, breaking through the performance ceilings seen when training decoy-detection classifiers on public data alone.
    - The classifier ABACUS scores de novo VHH designs, serving as the primary deployment model to rank candidate antibodies in active, live design campaigns.
    - Fixing High-Confidence Inversion: Standard metrics like ipTM and ipSAE ironically invert and decline at the highest scores, meaning the most "confident" structural predictions are frequently false positives. A classifier trained on NTX + SAbDab-nano + SEPIA completely eliminates this inversion, heavily outperforming raw structural metrics and real-data-only models.

------------------------------------------------------------------------

## 2. Mutational affinity landscapes (AlphaBind)

**Problem:** 
The combinatorial Explosion: We cannot find a plausible antibody by simply guessing and testing every possible mutant combination in a lab. While existing AI models (such as ESM-C) are completely blind when it comes to predicting how tightly an antibody will bind to a target, current models score weakly on the rigid structural backbone (framework) of antibody and become ineffective when evaluating flexible loops that form bonds with the target. So, it is more of random guesses.

**Massive Labeled Data**: The data shortage for antibody AI models was solved by open-sourcing a massive dataset called open-alphaseq. It contains 2 million data points tracking how mutations affect binding and accounts for more than half (>60%) of all public nanobody data and >99% of all public single-chain antibody (scFv) data.

**Filtering Data: AlphaBind**: In silico, it is possible to generate 10 million theoretical antibody designs, but testing those designs in a lab is incredibly difficult. AlphaBind AI screens them and provides the top ~10 highest-confidence candidate antibodies, driven by millions of pre-trained data points.

**Validating high-confidence candidates**: In the wet lab, those candidates are screened through BLI affinity, CHO expression, ELISA polyreactivity, aSEC purity, and DSF stability.

**Key results:**
- The AlphaSeq & AlphaBind pipeline successfully retains its predictive accuracy for identifying functional variants.
- Current open-source inverse-folding models are limited in their ability to accurately separate overall protein quality (including stability) from protein-protein interactions (including affinity scores).
- The massive data used to train AlphaBind helps track single de novo binders which might has significantly stronger sequences accessible through just 1–2 downstream mutations. This feature is highly useful to deploy in silico to affinity-mature weak initial structures.

------------------------------------------------------------------------

## 3. Benchmarking (defining SOTA)

**Tiered Difficulty Framework**
To standardize evaluation, the computational design targets were categorized into 3 tiers as follows:
1. Tier1:  Known antibody bound complex structure exists
2. Tier2: Known antigen structure exists, but no bound antibody structure is available
3. Tier3: Unbound target

**Benchmark evaluation**:  The 3 best AI antibody design models were tested against 30 different targets, which revealed that no single model is successful at designing a working antibody for all 30 targets. Additionally, off-target or promiscuous binding remains a significant issue. This suggests that each model has its own unique strengths and weaknesses.

**Broader validation effort is in progress**

------------------------------------------------------------------------


## References
- SEPIA preprint: https://www.biorxiv.org/content/10.64898/2026.04.17.719295v2.full.pdf
- Open AlphaSeq dataset: https://huggingface.co/datasets/aalphabio/open-alphaseq
- NaturalAntibody DB: https://naturalantibody.com/agab/
- Younger et al., *PNAS* (2017); Engelhart et al., *Antibody Therapeutics* (2022) — AlphaSeq methodology

---
*Notes compiled from a live seminar (A-Alpha Bio slide deck, "Boston Protein Design and Modeling Club").*
