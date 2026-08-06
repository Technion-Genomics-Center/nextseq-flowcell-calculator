# Flowcell selection — calculation spec

Source data: `applications.csv` (per-application defaults), `flowcells.csv` (kit specs).
All parameters are user-overridable; the CSV only supplies the starting values.

## 1. Reads required per sample

**Read-driven applications** (`driver = reads`):

```
reads_per_sample_M = default_reads_per_sample_M             (user-adjustable)
```

**Coverage-driven applications** (`driver = coverage`):

```
reads_per_sample_M = (coverage_x * genome_size_Mb) / (read_length * bases_per_cluster)
```

where `bases_per_cluster` = 2 for PE, 1 for SE.

This is raw sequencer output. There is deliberately **no QC/alignment loss factor** — a
"usable read fraction" was removed because its value is unknowable up front and it
silently inflated every request. Users who need a guaranteed post-alignment depth should
raise the requested reads themselves.

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
- `available_reads_M >= num_samples * reads_per_sample_M`

## 4. Outputs

```
max_samples_possible = floor(available_reads_M / reads_per_sample_M)
```

### Ranking eligible flowcells

Recommend the eligible kit with the lowest `cost_rank` in `flowcells.csv`.

The rank is an **explicit ordering, not a formula**, because no derived proxy works:

- **Output size fails.** P2 600 yields 300 M reads while P2 100 yields 400 M, so sorting
  by output puts the far more expensive 600-cycle kit first.
- **Flowcell tier fails.** P4 50 is cheaper than P3 100 despite being a higher tier, so
  an SE 72–100 bp RNAseq run should land on P4 50, never P3 100. (P3 100 is still correct
  when PE 50 is required — P4 50 is single-end only, so the compatibility filter removes
  it and P3 100 becomes the cheapest eligible kit. That falls out of the ordering; it is
  not special-cased.)
- **Cycle count fails too.** It would rank P4 100 above P1 300, which is wrong.

Current order (cheapest → most expensive):

```
P1 100 · P1 300 · P1 600 · P2 100 · P2 200 · P2 300 · P4 50 ·
P2 600 · P3 100 · P3 200 · P3 300 · P4 100 · P4 200 · P4 300
```

**Only the P4 50 < P3 100 relationship is confirmed by ATGC.** The rest is a plausible
seeded order and should be replaced with the real price list — edit `cost_rank` in
`flowcells.csv` and the matching `costRank` values in `index.html`.

### When nothing fits

Report the **largest compatible** kit and how many samples it does hold, plus the number
of runs the requested sample count would need. A bare "no flowcell fits" leaves the user
with no way to find the number that works.

Note: `floor()` needs a small epsilon (`+ 1e-9`). Division like `60 / 0.1` can land on
599.9999999999999 in floating point, which floors to 599 instead of 600.

## Worked example — 16S, 2x300, 100k reads/sample

`min_read_length_required = 250` and PE leaves only the 600-cycle kits eligible.

```
reads_per_sample = 0.1 M

P1 600:  100 M * (1 - 0.40) = 60 M available  -> 600 samples
P2 600:  300 M * (1 - 0.40) = 180 M available -> 1800 samples
```

Ignoring the PhiX share would have overstated P1 600 at 1000 samples — a 40% overbooking
of the flowcell.
