# NextSeq 2000 Flowcell Calculator

A standalone web tool that recommends the right Illumina NextSeq 2000 flowcell from a
selected application and its sequencing parameters, and reports how many samples fit.

Open `index.html` in a browser. No build step, no server, no dependencies — the data
tables are embedded in the page so it works from `file://` and can be dropped into the
ATGC site as-is.

## Files

| File | Purpose |
|---|---|
| `index.html` | The tool. Self-contained: markup, styles, data, and logic. |
| `applications.csv` | Per-application defaults (depth, read length, PhiX, links). |
| `flowcells.csv` | The 14 NextSeq 2000 kit configurations and their read yields. |
| `parameters.csv` | Hover-explanation text for each sequencing parameter. |
| `CALCULATION.md` | How reads, capacity, and the flowcell ranking are computed. |

The CSVs are the source of truth for the data; `index.html` embeds a copy of each.
**Edit the CSV and the matching block in `index.html` together** — they are not loaded at
runtime (a `file://` page cannot fetch sibling files).

## ⚠ Application defaults are not verified

Every row in `applications.csv` is `verified = no`. The depth, coverage, and genome-size
values are plausible placeholders, **not ATGC's validated numbers**, and should not be
used to plan a real run until reviewed. Known gaps:

- **Amplicons** has no `default_reads_per_sample_M`, so the tool cannot recommend a
  flowcell for it.
- **Metagenomics** specifies both a reads and a coverage default; only coverage is used
  while `driver = coverage`.
- **Variant analysis** rows are single-end, which needs twice the reads of paired-end for
  the same coverage — neither default fits on one flowcell at the stated sample count.

## Notes for future sessions

Things that are easy to get wrong or to undo by accident.

### The data lives in two places

Each CSV is mirrored by a `const` block in `index.html`. Nothing is fetched at runtime,
because a `file://` page cannot read sibling files (CORS). **Any data change must be made
in both**, or the tool and the CSV silently disagree. The blocks are marked
`/* data (from <file>.csv) */`.

### Verifying a change

There is no test runner. Behaviour was checked by evaluating the shipped `calc()` and the
data blocks out of `index.html` under Node, then asserting against every application's
defaults plus edge cases (over-capacity, below-minimum read length, zero input), and by
checking that every `getElementById` target exists in the markup. Re-do this after logic
edits — the calculation has no other safety net.

### Decisions that look like bugs but are not

- **Flowcell ranking is tier-then-cycles** (P1 < P2 < P3 < P4, then cycle count), *not*
  read output. Sorting by output puts P2 600 (300 M) ahead of P2 100 (400 M) and
  recommends the far more expensive kit. Add real prices to `flowcells.csv` if you want
  true cost sorting.
- **`floor()` carries a `+ 1e-9` epsilon.** `60 / (0.05 / 0.85)` is 1019.9999… in floating
  point and would floor to 1019 instead of 1020.
- **No dark mode.** Deliberately removed so the tool matches the light-only ATGC site.
- **Tooltips toggle on tap, not just hover.** Hover-only made every explanation
  unreachable on phones. Do not revert to `:hover` alone.
- **PhiX is editable**, defaulting to 40% for low-diversity libraries. It was briefly
  locked; "ATGC adds the control library as requested" means it is a default, not a
  constraint.

### Design source

The palette is the four-stop `.brandline` gradient in
`ATGC_website_2025/webpage/index.html`: red `#e73334`, orange `#fbac45`, blue `#289de0`,
pink `#e82281`. Tokens, fonts (Raleway / Roboto / IBM Plex Mono), radii, and panel
treatments are copied from that file so the tool drops into the site cleanly.

### Environment

- Git is installed **per-user** at `%LOCALAPPDATA%\Programs\Git` — not on the system PATH
  by default.
- The working copy lives on a network share, so Git rejects it with "dubious ownership"
  until the path is added to `safe.directory`. Git prints the exact command to run,
  including the resolved UNC path — copy it from the error message.

### Never visually verified

Every layout and styling change so far was verified structurally only — the browser
automation extension was not connected in any session. **Nobody has confirmed how the
page actually renders.** Check it in a real browser, at desktop and phone widths, before
trusting the design.

### Open questions for ATGC

1. `amplicons` needs a `default_reads_per_sample_M`.
2. `metagenomics` — reads or coverage? Both are set; only coverage is used.
3. Are the single-end defaults on the variant rows intended? SE doubles the reads needed.
4. Three applications have no dedicated ATGC page and use a nearest match: 16S and
   Amplicons → Amplicon-seq, Assembly de novo → DNA-seq, RRBS → DNA Methylation.

## PhiX control library

Low-diversity libraries (16S, amplicons) default to a **40%** PhiX spike-in, Illumina's
recommendation for the NextSeq 1000/2000, where at least 32% of reads should align to
PhiX. ATGC adds the control library as requested. That share is deducted from flowcell
output before sample capacity is calculated. See
[Illumina KB 000001527](https://knowledge.illumina.com/instrumentation/general/instrumentation-general-reference_material-list/000001527).
