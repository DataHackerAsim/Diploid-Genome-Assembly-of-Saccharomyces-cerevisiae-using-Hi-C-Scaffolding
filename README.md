# Diploid Genome Assembly of *Saccharomyces cerevisiae* using Hi-C Scaffolding

**Author:** Asim Ahmed
**Institution:** National University of Sciences and Technology (NUST) — SINES
**Program:** BS Bioinformatics

---

## Project Overview

This project follows the [Vertebrate Genomes Project (VGP) assembly pipeline](https://training.galaxyproject.org/training-material/topics/assembly/tutorials/vgp_genome_assembly/tutorial.html) from the Galaxy Training Network, applied to *Saccharomyces cerevisiae* (baker's yeast) as a test organism. The yeast genome (~12 Mb, 16 chromosomes) serves as a practical, computationally lightweight model for learning and validating the full VGP workflow before scaling to larger vertebrate genomes.

Because *S. cerevisiae* is diploid, both haplotypes were assembled independently to avoid collapsing parental sequences into a single chimeric representation. The key technology enabling chromosome-level scaffolding is **Hi-C sequencing**, which captures chromatin conformation — the physical proximity of DNA segments inside the nucleus. By analyzing which contigs share dense 3D contact signals, we infer their linear adjacency on a chromosome and scaffold them accordingly.

---

## Datasets

| Dataset | Description |
|:---|:---|
| `HiFi_synthetic_50x_01/02/03.fasta` | PacBio HiFi contigs (high-accuracy, long-read) |
| `SRR7126301_1/2.fastq.gz` | Illumina Hi-C paired-end reads (forward and reverse) |
| `Hi-C_dataset_F`, `Hi-C_dataset_R` | Additional Hi-C read pairs for scaffolding |
| `bionano.cmap` | Bionano optical map for hybrid scaffolding |

---

## Pipeline

The full workflow was built and executed on the [Galaxy](https://usegalaxy.org/) platform. The exported `.ga` file is included in this repository for direct import and reproduction.

### 1. Contig Assembly — HiFiASM

Raw HiFi reads were assembled into phased contigs using **HiFiASM** (v0.25.0), which natively resolves haplotypes using Hi-C phasing information. The two output GFA graphs (one per haplotype) were converted to FASTA with **gfa_to_fa**.

### 2. K-mer Profiling and QC — Meryl + GenomeScope + Merqury

**Meryl** (v1.3) built a k-mer database from Illumina short reads, which served two purposes:

- **GenomeScope** (v2.1) used the k-mer frequency spectrum to estimate genome size, heterozygosity, and repeat content.
- **Merqury** (v1.3) evaluated assembly completeness and per-base accuracy (QV) by comparing the assembly's k-mer content against the read-derived truth set.

### 3. Read QC — Cutadapt

Hi-C reads were adapter-trimmed with **Cutadapt** (v5.2) prior to alignment to remove sequencing artifacts that would otherwise produce spurious contact signals.

### 4. Hybrid Scaffolding — Bionano

**Bionano Solve** (v3.7.0) integrated the optical map (`bionano.cmap`) with HiFiASM contigs to resolve large-scale structural ambiguities before Hi-C scaffolding. This was performed independently for both haplotypes.

### 5. Hi-C Scaffolding — BWA-MEM2 + Bellerophon + YAHS

For each haplotype:

1. **BWA-MEM2** (v2.3) aligned Hi-C read pairs to the Bionano-scaffolded assembly.
2. **Bellerophon** (v1.0) filtered chimeric Hi-C read pairs that span ligation junctions — these would otherwise introduce false long-range contacts.
3. **YAHS** (v1.2a.2) consumed the filtered alignments and iteratively joined contigs into chromosome-level scaffolds based on contact frequency.

### 6. Evaluation

| Tool | Purpose |
|:---|:---|
| **Fasta Statistics** | N50, total length, scaffold count |
| **BUSCO** (v5.8.0) | Gene-space completeness against saccharomycetes orthologs |
| **PretextMap + PretextSnapshot** | Hi-C contact matrix visualization |

---

## Results

| Metric | Haplotype 1 | Haplotype 2 |
|:---|:---|:---|
| **Total Length** | ~12.07 Mb | ~11.30 Mb |
| **Scaffold Count** | 15 | 16 |
| **N50** | 923,452 bp | 922,430 bp |
| **Longest Scaffold** | 2,610,027 bp | 1,532,843 bp |

Both haplotypes achieved high contiguity (N50 > 900 kb) and were condensed into approximately 15–16 scaffolds, consistent with the expected 16 chromosomes of *S. cerevisiae*. Full assembly statistics and BUSCO reports are included in this repository:

- **Haplotype 1 (dataset 224):** [`Galaxy233-..summary stats.tabular`](Galaxy233-[Fasta%20Statistics%20on%20dataset%20224_%20summary%20stats].tabular) | [`Galaxy238-..Busco..Specific lineage.txt`](Galaxy238-[Busco%20on%20dataset%20224_%20Short%20summary%20-%20Specific%20lineage].txt)
- **Haplotype 2 (dataset 218):** [`Galaxy237-..summary stats.tabular`](Galaxy237-[Fasta%20Statistics%20on%20dataset%20218_%20summary%20stats].tabular) | [`Galaxy246-..Busco..Specific lineage.txt`](Galaxy246-[Busco%20on%20dataset%20218_%20Short%20summary%20-%20Specific%20lineage].txt)

### Contact Maps

The diagonal pattern of dense, discrete squares confirms successful chromosome-level scaffolding with minimal inter-chromosomal noise.

| Haplotype 1 | Haplotype 2 |
|:---:|:---:|
| ![Hap1](Galaxy272-[pretext_snapshotFullMap].png) | ![Hap2](Galaxy255-[pretext_snapshotFullMap].png) |

---

## Reproducing This Analysis

1. Download `Galaxy-Workflow-Workflow-cleaned.ga` from this repository.
2. Log into a Galaxy server (e.g., [usegalaxy.org](https://usegalaxy.org/) or [usegalaxy.eu](https://usegalaxy.eu/)).
3. Go to **Workflows → Import** and upload the `.ga` file.
4. Provide the input datasets listed above when prompted.
5. Execute the workflow.

---

## Repository Contents

```
.
├── README.md
├── Galaxy-Workflow-Workflow-cleaned.ga                              # Cleaned Galaxy workflow (45 steps)
├── Galaxy272-[pretext_snapshotFullMap].png                          # Haplotype 1 Hi-C contact map
├── Galaxy255-[pretext_snapshotFullMap].png                          # Haplotype 2 Hi-C contact map
├── Galaxy238-[Busco on dataset 224_ Short summary - Specific lineage].txt   # BUSCO results — Haplotype 1
├── Galaxy246-[Busco on dataset 218_ Short summary - Specific lineage].txt   # BUSCO results — Haplotype 2
├── Galaxy233-[Fasta Statistics on dataset 224_ summary stats].tabular       # Assembly stats — Haplotype 1
└── Galaxy237-[Fasta Statistics on dataset 218_ summary stats].tabular       # Assembly stats — Haplotype 2
```

---

## Tools and Versions

| Tool | Version | Purpose |
|:---|:---|:---|
| HiFiASM | 0.25.0 | Haplotype-resolved contig assembly |
| Meryl | 1.3 | K-mer counting |
| GenomeScope | 2.1.0 | Genome profiling |
| Merqury | 1.3 | Assembly QV and completeness |
| Cutadapt | 5.2 | Adapter trimming |
| Bionano Solve | 3.7.0 | Optical map hybrid scaffolding |
| BWA-MEM2 | 2.3 | Hi-C read alignment |
| Bellerophon | 1.0 | Chimeric read filtering |
| YAHS | 1.2a.2 | Hi-C scaffolding |
| BUSCO | 5.8.0 | Gene completeness |
| PretextMap | 0.2.4 | Contact matrix generation |
| PretextSnapshot | 0.0.7 | Contact matrix visualization |
| Fasta Statistics | 2.0 | Assembly statistics |
