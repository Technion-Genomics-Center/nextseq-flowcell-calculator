# Flowcell selection — calculation spec

Source data: `applications.csv` (per-application defaults), `flowcells.csv` (kit specs).
All parameters are user-overridable; the CSV only supplies the starting values.

## 1. Reads required per sample

**Read-driven applications** (`driver = reads`):

```
usable_reads_per_sample = default_reads_per_sample_M        (user-adjustable)
```

**Coverage-driven applications** (`driver = coverage`):

```
usable_reads_per_sample_M = (coverage_x * genome_size_Mb) / (read_length * bases_per_cluster)
```

where `bases_per_cluster` = 2 for PE, 1 for SE.

Both then convert to raw reads, since not every read survives QC/alignment:

```
raw_reads_per_sample_M = usable_reads_per_sample_M / usable_read_fraction
```

## 2. Reads available on the flowcell

A control library (PhiX) occupies a fixed share of the flowcell output. Those clusters
produce data but are **not** available to the samples, so they come off the top:

```
available_reads_M = flowcell.reads_M * (1 - phix_percent / 100)
```

`phix_percent` is per-application, from `applications.csv`:

| Application | PhiX | Reason |
|---|---|---|
| Amplicons | **40%** | Low-diversity library — default control library share |
| 16S | **40%** | Low-diversity library — default control library share |
| RRBS | 30% | Reduced base diversity at the MspI site |
| All others | 2% | Routine run-quality spike |

ATGC adds the control library as requested. For low-diversity libraries the default is
40% — Illumina's recommendation for the NextSeq 1000/2000, where at least 32% of reads
should align to PhiX and 40% leaves headroom for run-to-run variation
([Illumina KB 000001527](https://knowledge.illumina.com/instrumentation/general/instrumentation-general-reference_material-list/000001527)).
The field stays editable so a different spike-in can be costed.

## 3. Flowcell compatibility filter

A flowcell is eligible only if **all** hold:

- Run mode is supported: `PE` requires `pe_supported = yes`, `SE` requires `se_supported = yes`
- Read length fits: `read_length <= max_read_length_PE_each` (PE) or `<= max_read_length_SE` (SE)
- `read_length >= min_read_length_required` for the application
- `available_reads_M >= num_samples * raw_reads_per_sample_M`

## 4. Outputs

```
max_samples_possible = floor(available_reads_M / raw_reads_per_sample_M)
```

### Ranking eligible flowcells

Recommend the cheapest eligible kit, ranked by **flowcell tier first (P1 < P2 < P3 < P4),
then cycle count**.

Read output is *not* a usable cost proxy: P2 600 yields 300 M reads while P2 100 yields
400 M, so sorting by output puts the far more expensive 600-cycle kit first. An RNAseq
polyA run at PE50 was being sent to P2 600 for exactly this reason.

This ranking is still only a proxy. If you add real per-kit prices to `flowcells.csv`,
the recommendation can sort on actual cost instead.

### When nothing fits

Report the **largest compatible** kit and how many samples it does hold, plus the number
of runs the requested sample count would need. A bare "no flowcell fits" leaves the user
with no way to find the number that works.

Note: `floor()` needs a small epsilon (`+ 1e-9`). `60 / (0.05 / 0.85)` evaluates to
1019.9999999999999 in floating point, which floors to 1019 instead of 1020.

## Worked example — 16S, 2x300, 50k reads/sample

`min_read_length_required = 250` and PE leaves only the 600-cycle kits eligible.

```
raw_reads_per_sample = 0.05 / 0.85            = 0.0588 M

P1 600:  100 M * (1 - 0.40) = 60 M available  -> 1020 samples
P2 600:  300 M * (1 - 0.40) = 180 M available -> 3060 samples
```

Ignoring the PhiX share would have overstated P1 600 at 1700 samples — a 40% overbooking
of the flowcell.
