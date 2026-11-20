# Lab Notebook — plasmidinfbench_scripts

Chronological log of work sessions on this project. Newest entries at the bottom.
Append-only — never edit or reorder past entries.

Entry format:
## YYYY-MM-DD
- What was done / tried
- What happened (concrete results, errors, numbers — not a diff restatement)
- Decision or next step

---

## 2026-11-20
- Started this repo as the plasmid counterpart of `viralinfbench_scripts`.
  Viruses and plasmids get separate benchmark pipelines, so their parsers and
  merges are kept apart too. Background on the parser design, and why parsing
  runs as R scripts in Nextflow processes instead of in Nextflow operators,
  is in `viralinfbench_scripts/NOTEBOOK.md` (same date).
- Added `scripts/parse_plasmid_inference.r`. It has the same layout as the viral
  parser: output `sampleid | tool | contig | score | call`, with call = 1
  (plasmid) or 0 (scored but not called). Each tool is one entry (file
  pattern + parse function) in a `tools` list, so more tools can be added later.
  - DeepMicroClass2: score = `pls` probability, call = label is `pls`.
  - geNomad: reads `*_plasmid_summary.tsv`. Score = `plasmid_score`. Every row
    is a call, because the file only lists plasmid calls.
- Added `scripts/merge_plasmid_inference.r`: a long table plus a contig × tool
  1/0 table. With `--contigs`, every contig in the assembly gets a row.
- Added `scripts/README.md`: usage, output columns, the supported tools, and
  how to add a tool.
- Tested with no data in this repo yet:
  - DMC2: ran on `viralinfbench_scripts/data/deepmicroclass2/classification.tsv`.
    10000 rows, 2 plasmid calls.
  - geNomad: there's no real plasmid summary, so tested with a 2-row fake
    file. It was found correctly in a folder that also has the virus summary.
  - Merge: ran with `--contigs` on a small FASTA. A contig no tool reported
    got 0s, as it should.
- Next steps:
  - Get a real geNomad `*_plasmid_summary.tsv` and test on it.
  - Choose and add more plasmid tools.
  - Build the plasmid pipeline, with these scripts in its `bin/`.
