---
title: "Processing 150,000 Rows: A Python Data Pipeline with Pandas & NumPy"
excerpt: "How I built a reconciliation pipeline that saved 25 hours of manual work per month, and the performance tricks that made it scale to 150K rows per execution."
tags:
  - Python
  - Pandas
  - NumPy
  - Data Engineering
date: "2025-08-20"
featured: false
coverEmoji: "📊"
---

Excel reconciliation is the silent productivity killer in finance and ops teams. I was handed a spreadsheet workflow that took 25 hours a month and turned it into a 3-minute automated pipeline. Here's the full breakdown.

## The Original Workflow

Two teams were manually comparing exports from two different systems — cross-referencing transaction IDs, amounts, and dates row by row, flagging mismatches in a separate sheet. Error-prone and soul-crushing.

## Reading Multiple Excel Sources

```python
import pandas as pd
import numpy as np

df_system_a = pd.read_excel('system_a_export.xlsx', sheet_name='Transactions')
df_system_b = pd.read_excel('system_b_export.xlsx', dtype={'transaction_id': str})
```

Always specify `dtype` for ID columns — Pandas will happily truncate long numeric IDs if you let it infer the type.

## Vectorised Reconciliation with NumPy

Avoid row-by-row loops at all costs. Use `merge` for matching and boolean masks for flagging:

```python
merged = df_system_a.merge(
    df_system_b,
    on='transaction_id',
    how='outer',
    suffixes=('_a', '_b'),
    indicator=True
)

# Flag amount mismatches
merged['amount_mismatch'] = ~np.isclose(
    merged['amount_a'].fillna(0),
    merged['amount_b'].fillna(0),
    rtol=1e-5
)
```

`np.isclose` handles floating-point precision issues that kill naive `==` comparisons on currency values.

## Chunked Processing for Memory

At 150K rows, a naive read blows past 2GB RAM on a small EC2 instance. Process in chunks:

```python
chunk_size = 10_000
results = []
for chunk in pd.read_excel('large_file.xlsx', chunksize=chunk_size):
    results.append(process_chunk(chunk))
final = pd.concat(results, ignore_index=True)
```

## Writing the Output Report

Used `openpyxl` via Pandas to write a colour-coded output Excel file with conditional formatting — green for matched, red for mismatches, yellow for missing in one system.

## Outcome

- Runtime: ~3 minutes for 150K rows
- Manual time saved: 25 hours/month (65% reduction)
- Error rate: dropped to near-zero
