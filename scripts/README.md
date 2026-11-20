# Scripts

Parse the outputs of the plasmid inference tools into one standard table, then merge
the tables of all tools for a sample.

| Script | What it does |
|---|---|
| `parse_plasmid_inference.r` | Parses one tool's output into the standard table |
| `merge_plasmid_inference.r` | Merges the parsed tables of all tools for a sample |

## parse_plasmid_inference.r

```bash
./parse_plasmid_inference.r --tool genomad --input <file or tool output dir> \
    --sampleid <id> --output <sampleid>_genomad_parsed.tsv
```

`--input` can be the tool's output file or its whole output directory. With a
directory, the script finds the file by the tool's name pattern (see below) and
stops with an error if it finds none or more than one.

Output columns:

| Column | Meaning |
|---|---|
| `sampleid` | From `--sampleid` |
| `tool` | From `--tool` |
| `contig` | Contig ID: first word of the name, cut at any space or `\|` |
| `score` | The tool's plasmid score, on its own scale (`NA` if the tool gives none) |
| `call` | `1` if the tool calls the contig plasmid, `0` if it scored it but did not call it |

One row per contig. If a tool reports a contig more than once, the row with a call
is kept, then the highest score.

### Supported tools

| Tool | File (pattern) | Score | Call |
|---|---|---|---|
| `deepmicroclass2` | `classification.tsv` | `pls` | label is `pls` |
| `genomad` | `*_plasmid_summary.tsv` | `plasmid_score` | every row (file only lists plasmid calls) |

### Adding a tool

Each tool is one entry in the `tools` list at the top of the script: a file name
pattern and a `parse` function. The `--tool` choices, help text and error messages
come from that list, so adding an entry is enough to make the tool available.

```r
newtool = list(
  pattern = "_predictions\\.tsv$",                  # used when --input is a directory
  parse   = function(f) read_tsv(f, show_col_types = FALSE) %>%
    transmute(contig = clean_id(contig_name),       # column with the contig name
              score  = plasmid_prob,                # column with the plasmid score
              call   = as.integer(label == "plasmid")))   # 1 = plasmid, 0 = not
```

Separate it from the next entry with a comma, like the others in the list.

**If the tool's output looks different from the ones above, you still usually only
write its entry.** The `parse` function is where you say which columns hold the
contig name, the score and the call. Everything after it stays the same: cleaning
contig IDs, keeping one row per contig, writing the output, and the merge. Cases
that are still handled inside the entry:

- **The tool gives no score:** set `score = NA_real_`.
- **The tool gives a score but no call:** add a threshold option to `option_list`
  and use it in `call` through `opt$...`.
- **The file is a CSV, or has a different separator:** use `read_csv` or
  `read_delim` in `parse`.
- **The file only lists the tool's calls:** set `call = 1L`, as `genomad` does.

You need to change more than the entry when:

- **The result is split across two or more files:** each entry reads one file,
  so the input-finding step needs to pass several files to `parse`.
- **The tool only writes a FASTA of hits (no table):** `parse` needs to read the
  FASTA headers instead of a table, and contigs it didn't call won't appear
  (they become `0` in the merge).

## merge_plasmid_inference.r

```bash
./merge_plasmid_inference.r --prefix <sampleid>_plasmidinf \
    [--contigs contigs.fa --sampleid <id>] \
    <sampleid>_*_parsed.tsv
```

Writes two files:

- `<prefix>_long.tsv`: all parsed tables stacked (`sampleid | tool | contig | score | call`).
  Use it for scores, e.g. ROC/PR curves within a tool. Scores are not comparable
  across tools.
- `<prefix>_calls.tsv`: one row per contig, one `1`/`0` column per tool.
  `0` means the tool did not call the contig, or did not report it at all; check the
  long table to tell these apart. With `--contigs`, every contig in the assembly gets
  a row, including ones no tool reported.

```
sampleid  contig                  deepmicroclass2  genomad
test      frag_1000_NC_008212.1   0                1
test      frag_483_NZ_LR698974.1  1                0
```

## Requirements

R with `optparse`, `readr`, `dplyr` and `tidyr`.
