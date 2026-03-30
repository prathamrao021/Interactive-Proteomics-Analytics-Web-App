# Proteomics log2FC Heatmap and Interactive Gene Filter

An end-to-end workflow to visualize pairwise log2 fold-changes from a MaxQuant-style proteomics workbook and interactively extract gene lists by column-wise thresholds. The script normalizes headers, validates intensity columns, computes log2FC for chosen contrasts across time points, collapses duplicate genes, and renders a Plotly heatmap with rich hover details. The generated standalone HTML includes a filter panel, column-wise sorting, “click to copy” helpers, and CSV export.

---

## Table of Contents
- [Key Features](#key-features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Input Data Schema](#input-data-schema)
- [Configuration](#configuration)
- [Running](#running)
- [Output](#output)
- [How It Works](#how-it-works)
- [User Interface Guide](#user-interface-guide)
- [Customization](#customization)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Performance Tips](#performance-tips)
- [Project Structure](#project-structure)
- [License and Attribution](#license-and-attribution)

---

## Key Features
- Robust header normalization for messy spreadsheets (“24 h” → “24h”, non-breaking spaces, multiple spaces).
- Validates all expected intensity columns for 4 conditions × 3 time points (12 total).
- Preprocessing: multiplies intensities and floors zeros to avoid log issues; configurable pseudocount for stability.
- Computes log2FC for user-defined pairwise contrasts at each time point.
- Collapses duplicate genes by median, ensuring one row per gene.
- Interactive Plotly heatmap with:
  - Centered diverging color scale (RdBu), symmetric clamp, y-axis labels on the right.
  - Dropdown to sort rows ascending/descending by any contrast/time column.
  - Hover tooltip that shows gene, contrast/time, log2FC, protein name, and only the two raw intensity values used for that cell’s log2FC.
- Filter panel (client-side):
  - Choose a single column, pick direction (> or <), set cutoff, and list matching genes.
  - Copy-friendly table (click a header to copy the entire column), CSV download.
  - A “Copy” button near the summary that copies only “COLUMN, >/< CUTOFF, NVS”.

---

## Requirements
- Python 3.9+ recommended
- Packages: `numpy`, `pandas`, `plotly`, `openpyxl` (for .xlsx)

### Install with pip
```bash
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows
# .venv\Scripts\activate

pip install numpy pandas plotly openpyxl
```

---

## Input Data Schema
The script expects an Excel workbook (.xlsx) with at least:
- Gene names
- Protein names
- Intensity columns (case-insensitive after normalization) named exactly:
  - CNTRL 6h, CNTRL 12h, CNTRL 24h
  - EBV 6h, EBV 12h, EBV 24h
  - CD40 6h, CD40 12h, CD40 24h
  - EAG 6h, EAG 12h, EAG 24h

Header normalization performed:
- Trim leading/trailing spaces
- Replace non-breaking spaces with regular spaces
- Collapse multiple spaces to one
- Normalize “{number} h” → “{number}h”

Row labeling:
- Prefers “Gene names”; if all unknown and “Protein names” exists, uses protein names.
- Rows sharing the same gene label are collapsed by median.

---

## Configuration
At the top of the script:
```python
INPUT_PATH = "/path/to/MaxQuant-analysis_all NVS.xlsx"
OUTPUT_HTML = "/path/to/NVS_Proteomics.html"

PSEUDOCOUNT = 1.0
COLOR_RANGE = 10.0

PROTEIN_COL = "Protein names"
GENE_COL = "Gene names"

CONDITIONS = ["CNTRL", "EBV", "CD40", "EAG"]
TIMES = ["6h", "12h", "24h"]

# Pairwise contrasts: compute log2((A + eps) / (B + eps)) for each time
CONTRASTS = [
    ("EBV", "CNTRL"),
    ("EBV", "CD40"),
    ("EBV", "EAG"),
    ("EAG", "CNTRL"),
    ("CD40", "CNTRL"),
    ("EAG", "CD40"),
]
```

Notes:
- Preprocessing multiplies each of the 12 intensity columns by 1000 and replaces exact zeros with 0.001 (edit in the script if your data scale differs).
- COLOR_RANGE sets symmetric zmin/zmax for the diverging color scale.

---

## Running
From a terminal:
```bash
python your_script_name.py
```

The script writes a single HTML file to `OUTPUT_HTML`. Open it in any modern browser.

Tip: Ensure the last line reads exactly:
```python
if __name__ == "__main__":
    main()
```

---

## Output
- A self-contained HTML file with:
  - Interactive heatmap of log2FCs by contrast/time.
  - Dropdown to sort rows by any column (ascending/descending).
  - Filter panel to extract gene lists by threshold on a selected column.
  - Copy and CSV export utilities.
  - Hover tooltips showing the two raw intensities that produced each cell’s log2FC, plus protein name.

---

## How It Works
1. Normalize headers:
   - Harmonizes whitespace and “{t} h” → “{t}h” for reliable matching.

2. Validate intensity columns:
   - `find_intensity_columns` identifies all 12 expected columns using regex; errors if any are missing.

3. Preprocess intensities:
   - Multiplies values by 1000 and floors exact zeros to 0.001 (ensures stability in ratio/log).

4. Compute log2FC matrix:
   - For each contrast (A vs B) at each time, computes `log2((A + PSEUDOCOUNT) / (B + PSEUDOCOUNT))`.
   - Constructs a wide matrix with columns named like “A vs B | 6h”.
   - Collapses duplicate genes by median; drops rows that are all-NaN.

5. Hover metadata (customdata):
   - For each cell, packs `[Protein, A_value, B_value, "A time", "B time"]`.
   - Hover template renders only the two raw inputs used for that cell’s log2FC, keeping tooltips concise.

6. Sorting views:
   - Precomputes ascending/descending row indices per column, pushing NaNs to bottom.
   - Dropdown switches y-order and updates z and customdata accordingly.

7. HTML assembly:
   - Embeds Plotly figure and custom JS/HTML for the filter panel, results table, “click to copy,” and CSV download.

---

## User Interface Guide
Heatmap:
- Columns represent contrast/time pairs, e.g., “EBV vs CNTRL | 6h”.
- Colors: centered at zero (blue negative, red positive with RdBu, reversed), clamped by COLOR_RANGE.
- Hover: shows Gene, the column label, log2FC, Protein, and the two raw intensities used (e.g., “EBV 6h: …”, “CNTRL 6h: …”).
- Sorting: use the dropdown at top-right to sort rows ascending (↑) or descending (↓) by a specific column.

Filter panel:
- Column: pick one heatmap column to filter by.
- Direction: “>” or “<”.
- Cutoff: numeric threshold.
- Apply: updates the results table.
- Download CSV: saves the current results (Gene, Value).
- Results table:
  - Shows genes matching the threshold and their numeric value for the chosen column.
  - Click the “Gene” header to copy all gene names in the current results (click “Value” to copy the values).
- Summary:
  - Displays: `Matches: N genes (column: COLUMN, >/< CUTOFF, NVS)`
  - A small “Copy” button copies just `COLUMN, >/< CUTOFF, NVS`.

---

## Customization
- Conditions, time points, contrasts:
  - Edit `CONDITIONS`, `TIMES`, and `CONTRASTS` to match your study.
  - Ensure the spreadsheet columns match “{CONDITION} {TIME}”.

- Color behavior:
  - Adjust `COLOR_RANGE` to widen/narrow the saturation window.
  - You can also change `colorscale` or remove `reversescale`.

- Aggregation of duplicates:
  - Currently `.median()`. Swap for `.mean()` or a custom reducer if desired.

- Pseudocount and preprocessing:
  - Tune `PSEUDOCOUNT` to match your intensity scale and noise floor.
  - Remove or change the multiply-by-1000 and zero-flooring steps.

- Hover content variants:
  - Current setup shows only the two raw inputs used for each cell.
  - To show all 12 raw intensities instead, build `customdata` with all 12 values and update the `hovertemplate` accordingly.

- Filter panel enhancements (optional):
  - Multi-column union/intersection, Top-N, or in-table search can be added in JS.

---

## Troubleshooting
- ValueError: Missing intensity columns for: …
  - Ensure the sheet has all 12 normalized labels (e.g., “EBV 6h”). Normalize unusual headers or extend the regex in `find_intensity_columns`.

- Missing required column: 'Gene names' or 'Protein names'
  - Confirm column names match config; update `GENE_COL` and `PROTEIN_COL` if your headers differ.

- Clipboard copy does not work
  - Some browsers restrict clipboard in `file://` contexts. Use the provided button (a user gesture), or serve the HTML via a local HTTP server.

- Extremely large figures (slow rendering)
  - Reduce the dataset prior to plotting, limit columns/contrasts, or lower the heatmap height scaling.
  - Consider filtering the DataFrame before computing `mat`.

- Script won’t run due to main guard
  - Ensure it reads `if __name__ == "__main__": main()`.

---

## FAQ
- How is log2FC computed?
  - `log2((A + PSEUDOCOUNT) / (B + PSEUDOCOUNT))` per contrast and time.

- What happens to duplicate genes?
  - Collapsed by median so each gene appears once.

- Where do NaNs go when sorting?
  - NaNs are placed at the bottom for both ascending and descending sorts.

- Can I change which raw values are shown on hover?
  - Yes. The code maps each heatmap column “A vs B | t” back to “A t” and “B t”, and places only those two in `customdata`. You can include more or fewer fields by editing that builder and the `hovertemplate`.

---

## Performance Tips
- Pre-filter to a subset of genes of interest before computing `mat`.
- Avoid overly large color ranges if most values lie near zero; it improves perceptual contrast.
- If the HTML is huge, serving it via a simple HTTP server (e.g., `python -m http.server`) can improve clipboard and performance behavior versus opening via `file://`.

---

## Project Structure
Single-file script that:
- Loads Excel, normalizes headers, validates columns.
- Preprocesses intensities and computes the gene × contrast/time log2FC matrix.
- Builds Plotly heatmap, sorting controls, and hover metadata.
- Writes a self-contained HTML with a filter panel and export utilities.

You may optionally break it into modules:
- io.py (loading/normalization), compute.py (log2FC, aggregation), viz.py (heatmap + html), app.py (entrypoint).

---

## License and Attribution
- Uses open-source libraries (`numpy`, `pandas`, `plotly`). Review their licenses for redistribution.
- If you use this visualization in a publication, consider citing Plotly and the data processing tools you relied on.

If you want, I can tailor this README with lab-specific wording, add screenshots, or include a small sample dataset for quick testing.
