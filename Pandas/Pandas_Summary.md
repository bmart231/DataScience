# Pandas Summary — Study Reference

A condensed, organized reference distilled from the notebooks in this `Pandas/` folder. Each section explains the core concepts, why they matter for real data science work, and includes runnable code patterns you can adapt.

**Source notebooks:**
1. `Getting_started_with_pandas.ipynb` — Series basics
2. `DataFrame.ipynb` — DataFrame fundamentals
3. `DataLoading/DataLoading.ipynb` — reading/writing CSV, JSON
4. `DataLoading/InteractingWithDatabases.ipynb` — SQL databases
5. `DataCleaning_Prep/DataCleaning_and_Preparation.ipynb` — missing data, dedup, transforms, strings
6. `Data_Wrangling/Join_Combine_Reshape.ipynb` — MultiIndex, merge/join
7. `CorrCov.ipynb` — correlation & covariance

## Table of Contents

1. [Getting Started with Pandas](#getting-started-with-pandas)
2. [DataFrame Fundamentals](#dataframe-fundamentals)
3. [Data Loading & I/O](#data-loading--io)
4. [Interacting with Databases](#interacting-with-databases)
5. [Data Cleaning & Preparation](#data-cleaning--preparation)
6. [Data Wrangling: Join, Combine & Reshape](#data-wrangling-join-combine--reshape)
7. [Correlation & Covariance](#correlation--covariance)

---

## Getting Started with Pandas

Pandas provides data structures and manipulation tools designed to make data cleaning and analysis fast and convenient in Python. It's built to work with tabular or heterogeneous data, and is typically used alongside `numpy`, `scipy`, `statsmodels`/`scikit-learn`, and `matplotlib`. The standard import convention is:

```python
import numpy as np
import pandas as pd
from pandas import Series, DataFrame
```

### The `Series`: a labeled 1D array

A `Series` is a one-dimensional array-like object holding a sequence of values of the same type, plus an associated array of labels called its *index*. It's the pandas equivalent of a single labeled column of data. Why it matters: unlike a plain NumPy array, every value in a Series is tied to a label, so you can select, align, and combine data by meaning (e.g., "Texas") rather than by raw position.

```python
obj = pd.Series([4, 7, -5, 3])
obj.array   # the underlying data (a PandasArray wrapping a NumPy array)
obj.index   # default index: RangeIndex(start=0, stop=4, step=1)

# Give the index meaningful labels instead of the default 0..N-1
obj2 = pd.Series([4, 7, -5, 3], index=["d", "b", "a", "c"])
obj2["a"]            # -5, label-based lookup
obj2[["c", "a", "d"]]  # select multiple labels at once, order preserved
```

### NumPy-style operations preserve the index

Because a Series wraps a NumPy array, vectorized operations (filtering, scalar math, ufuncs) work exactly like they do on arrays — but the index-to-value link is preserved through every transformation. This is useful because it lets you keep track of *which* label each transformed value belongs to, instead of losing that context after each operation.

```python
obj2[obj2 > 0]   # boolean filtering keeps matching labels
obj2 * 2         # scalar multiplication keeps the same index
np.exp(obj2)     # NumPy ufuncs work directly on a Series
```

### Series as an ordered dictionary

A Series can be thought of as a fixed-length, ordered dict — a mapping from index labels to values. This makes dict-like operations (`in`, construction from a dict) natural, and is handy when your data is naturally key-value shaped (e.g., a lookup table).

```python
sdata = {"Ohio": 35000, "Texas": 71000, "Oregon": 16000, "Utah": 5000}
obj3 = pd.Series(sdata)      # dict keys become the index, in insertion order
obj3.to_dict()                # convert back to a plain Python dict

"b" in obj2   # True — checks against the index, like a dict key check
```

### Reindexing and missing data (`NaN`)

When you build a Series from a dict but pass an explicit `index`, pandas aligns the dict's values to that index: labels found in the dict get their value, labels not found get `NaN` (missing data), and dict keys not in the passed index are dropped. This is important because it's the mechanism pandas uses whenever you need to conform data to a specific set of labels — e.g., aligning multiple data sources to a common index.

```python
states = ["California", "Ohio", "Oregon", "Texas"]
obj4 = pd.Series(sdata, index=states)
# California -> NaN (not in sdata), Utah is dropped (not in states)

pd.isna(obj4)     # boolean mask of missing values (also available as obj4.isna())
pd.notna(obj4)    # inverse mask
```

### Automatic index alignment in arithmetic

When you perform arithmetic between two Series, pandas automatically aligns values by index label rather than by position — similar to a database join. Labels present in only one Series produce `NaN` in the result. This "align first, then compute" behavior is one of pandas' most important features: it means you rarely have to manually match up rows/labels before combining datasets.

```python
obj3 + obj4
# Ohio, Oregon, Texas add normally (present in both)
# California and Utah become NaN (present in only one of the two)
```

### Renaming an index in place

The `.index` attribute can be reassigned directly to relabel a Series' data without changing the underlying values — useful when you want to swap out generic labels (like default integer positions) for meaningful ones after the fact.

```python
obj.index = ["Bob", "Steve", "Jeff", "Ryan"]
obj
# Bob      4
# Steve    7
# Jeff    -5
# Ryan     3
```

---

## DataFrame Fundamentals

A `DataFrame` is a rectangular table of data: an ordered collection of named columns (each column can hold a different dtype — numeric, string, boolean, etc.) sharing a common row index. Mentally, it's a dict of `Series` objects that all align on the same index. This is the workhorse structure for almost all tabular data analysis in pandas.

### Creating a DataFrame

The most common way to build one is from a dictionary of equal-length lists/arrays. You can control column order, and pandas will insert `NaN` for any requested column that isn't in the source data — useful for stubbing out columns you plan to fill in later.

```python
import pandas as pd
import numpy as np

data = {
    "state": ["Ohio", "Ohio", "Ohio", "Nevada", "Nevada", "Nevada"],
    "year": [2000, 2001, 2002, 2001, 2002, 2003],
    "pop": [1.5, 1.7, 3.6, 2.4, 2.9, 3.2],
}
frame = pd.DataFrame(data)

# Explicit column order
pd.DataFrame(data, columns=["year", "state", "pop"])

# Requesting a column that doesn't exist -> filled with NaN (handy placeholder column)
frame2 = pd.DataFrame(data, columns=["year", "state", "pop", "debt"])
```

A **nested dictionary of dictionaries** is another common input shape: the outer keys become columns, the inner keys become the row index (union of all inner keys, sorted). This is a natural way to build a DataFrame out of several "columns worth" of `{index: value}` mappings.

```python
populations = {
    "Ohio": {2000: 1.5, 2001: 1.7, 2002: 3.6},
    "Nevada": {2001: 2.4, 2002: 2.9},
}
frame3 = pd.DataFrame(populations)  # missing (Ohio, 2001... wait Nevada,2000) -> NaN
```

`head()` / `tail()` return the first/last 5 rows — the standard quick-look tool when a DataFrame is too large to print in full.

### Inspecting and Reshaping the Whole Frame

- `frame.columns`, `frame.index` — the column/row labels.
- `frame.index.name` / `frame.columns.name` — label the axes themselves (shows up in the printed output header), useful for documenting what a MultiIndex-free axis represents.
- `frame.T` — transpose (swap rows/columns). Note: if columns have mixed dtypes, transposing collapses everything to `object` dtype, so transposing twice can lose your original type info.
- `frame.to_numpy()` — pull the underlying data out as a 2D ndarray. If the columns have different dtypes, NumPy picks a common dtype (e.g., everything becomes `object`) that can hold them all.

```python
frame3.index.name = "year"
frame3.columns.name = "state"
frame3.T          # rows <-> columns
frame3.to_numpy() # -> 2D array, dtype chosen to fit all columns
```

### Accessing and Modifying Columns

Columns come back as a `Series` (sharing the DataFrame's index) via dict-style `[ ]` or dot-attribute access. Rows come back by position via `.iloc`.

```python
frame2["state"]   # dict-style column access
frame2.year        # attribute-style column access (fails if name collides with a method)
frame2.iloc[1]     # row by integer position -> returned as a Series
```

Assigning to a column: a scalar broadcasts to every row; an array/list must match the DataFrame's length exactly; a `Series` gets **realigned to the DataFrame's index**, so any index values in the DataFrame that aren't in your Series become `NaN`. This alignment-on-assignment behavior is important to know — it's easy to accidentally wipe out a column with NaNs if the Series index doesn't match.

```python
frame2["debt"] = 16.5                 # scalar -> broadcasts
frame2["debt"] = np.arange(6.)        # array -> must match length
val = pd.Series([-1.2, -1.5, -1.7], index=["two", "four", "five"])
frame2["debt"] = val                  # realigned to frame2's index -> mostly NaN here

# Assigning to a new column name creates it; boolean columns are common for flags/filters
frame2["eastern"] = frame2["state"] == "Ohio"

# del removes a column in place
del frame2["eastern"]
```

### Index Objects

Every axis label sequence (row index or `.columns`) is internally a pandas `Index`. Two properties matter in practice:

- **Immutable** — you can't do `index[1] = "x"`. This makes it safe to *share* the same Index object across multiple Series/DataFrames without fear that one will mutate it out from under another.
- **Set-like but allows duplicates** — you can do `"Ohio" in frame.columns`, but unlike a Python `set`, an Index can contain repeated labels (selecting a duplicated label returns all matching rows/columns).

```python
labels = pd.Index(np.arange(3))
obj2 = pd.Series([1.5, -2.5, 0], index=labels)
obj2.index is labels   # True -- the same Index object is shared, safely, since it's immutable

"Ohio" in frame3.columns   # membership test, set-like
pd.Index(["foo", "foo", "bar", "bar"])  # duplicates are allowed
```

### Reindexing

`reindex` creates a **new** object whose data is rearranged to match a new index, inserting `NaN` for any label that wasn't already present. This is the standard way to align disparate data onto a common index (e.g., before combining several series/frames), or to reorder rows/columns explicitly.

```python
obj = pd.Series([4.5, 7.2, -5.3, 3.6], index=["d", "b", "a", "c"])
obj.reindex(["a", "b", "c", "d", "e"])   # "e" is new -> NaN

# For ordered/time-series data, forward-fill instead of introducing NaNs
obj3 = pd.Series(["blue", "purple", "yellow"], index=[0, 2, 4])
obj3.reindex(np.arange(6), method="ffill")   # fills gaps with the last valid value
```

On a DataFrame, `reindex` can retarget rows, columns, or both, and can be called positionally with `axis=`:

```python
frame = pd.DataFrame(np.arange(9).reshape((3, 3)),
                      index=["a", "c", "d"], columns=["Ohio", "Texas", "California"])

frame.reindex(index=["a", "b", "c", "d"])        # reindex rows -> "b" row is NaN
frame.reindex(columns=["Texas", "Utah", "California"])  # reindex columns -> drops "Ohio", adds "Utah" (NaN)
frame.reindex(["Texas", "Utah", "California"], axis="columns")  # same, via axis kwarg
```

Key reindex options worth remembering: `method` (`"ffill"`/`"bfill"`), `fill_value` (what to put in new slots instead of NaN), `limit` (cap on how many consecutive gaps get filled).

Many pandas users prefer `.loc[...]` for reindexing instead, but it only works if **every** requested label already exists — it won't create new rows/columns the way `reindex` will:

```python
frame.loc[["a", "d", "c"], ["California", "Texas"]]  # reorders rows/cols, but all labels must already exist
```

### Dropping Entries

`.drop()` returns a **new** object with the given labels removed from an axis — cleaner than manually filtering with `reindex`/boolean masks when you know exactly what to remove.

```python
data = pd.DataFrame(np.arange(16).reshape((4, 4)),
                     index=["Ohio", "Colorado", "Utah", "New York"],
                     columns=["one", "two", "three", "four"])

data.drop(index=["Colorado", "Ohio"])     # drop rows
data.drop(columns=["two"])                # drop columns
data.drop("two", axis=1)                  # same, via axis=1 (NumPy-style)
data.drop(["two", "four"], axis="columns")
```

### Indexing, Selection, and Filtering

#### Series indexing

Series support NumPy-style indexing plus label-based indexing. Watch out: **plain `[]` indexing is ambiguous when the index itself contains integers** — pandas treats the integers as *labels*, not positions, which surprises people coming from lists/arrays.

```python
obj = pd.Series(np.arange(4.), index=["a", "b", "c", "d"])
obj["b"]          # label lookup
obj[2:4]           # positional slice still works
obj[obj < 2]       # boolean filtering

obj1 = pd.Series([1, 2, 3], index=[2, 0, 1])
obj1[[0, 1, 2]]    # !! these are treated as LABELS, not positions, because the index is integer-typed
```

#### `.loc` vs `.iloc` — the fix for that ambiguity

- **`.loc`** — always label-based.
- **`.iloc`** — always integer-position-based.

Prefer these explicitly over bare `[]` whenever the index could contain integers, so your intent (label vs. position) is unambiguous and your code behaves the same regardless of the index's dtype.

```python
obj1.iloc[[0, 1, 2]]     # first three rows by POSITION, regardless of index labels
obj2.loc["b":"c"]        # label slicing is INCLUSIVE of the endpoint (unlike normal Python slicing)
obj2.loc["b":"c"] = 5    # loc/iloc also support assignment
```

#### DataFrame `[]`, boolean masks, and `.loc`/`.iloc`

Plain `df[...]` mostly selects **columns** (`df["two"]`, `df[["three", "one"]]`), with two special-cased conveniences: a slice selects **rows** (`df[:2]`), and a boolean Series/array filters **rows** (`df[df["three"] > 5]`). A full boolean DataFrame (e.g. `df < 5`) can be used to conditionally assign values element-wise.

```python
data["two"]                 # column
data[["three", "one"]]      # multiple columns, reordered
data[:2]                     # first 2 rows (slice = row convenience)
data[data["three"] > 5]     # boolean row filter

data[data < 5] = 0          # element-wise conditional assignment via boolean DataFrame
```

`.loc`/`.iloc` generalize to two dimensions, letting you select rows and columns together in one call — this is the recommended way to do combined row+column selection rather than chaining two `[]` lookups:

```python
data.loc["Colorado"]                      # single row -> Series
data.loc[["Colorado", "New York"]]        # multiple rows -> DataFrame
data.loc["Colorado", ["two", "three"]]    # row AND column selection together
data.iloc[2]                               # row by position
data.iloc[[2, 1]]                          # rows by position
data.loc[data.three >= 2]                 # boolean row filter works with loc (not iloc)
```

Also available: `.at[row, col]` / `.iat[row, col]` for fast scalar (single-value) access by label / position, respectively — cheaper than `.loc`/`.iloc` when you only need one cell.

#### Integer-indexing pitfalls

If a Series/DataFrame's index is made of integers, plain `[]` indexing is **always label-based** — `ser[-1]` raises an error rather than "falling back" to positional indexing, because pandas refuses to guess your intent. Slicing (`ser[:2]`), however, is always positional regardless of index dtype. When in doubt, use `.loc` (label) or `.iloc` (position) explicitly.

#### Chained-indexing pitfall (and how to avoid it)

Writing `df.loc[cond]["col"] = value` looks reasonable but silently fails to update the original DataFrame — the first `.loc[cond]` produces an intermediate copy (Copy-on-Write), so the assignment lands on a throwaway object. Pandas raises a `ChainedAssignmentError` warning for this. **Always combine the row and column selector into a single `.loc[...]` call** when assigning:

```python
# WRONG — operates on a copy, original data unchanged, raises ChainedAssignmentError
data.loc[data.three == 5]["three"] = 6

# RIGHT — single .loc call with both row and column selectors
data.loc[data.three == 5, "three"] = 6
```

### Function Application and Mapping

NumPy ufuncs (`np.abs`, `np.sqrt`, etc.) work directly on DataFrames/Series element-wise, since pandas objects wrap NumPy arrays under the hood.

For applying **your own** function, use `.apply()` to run a function once per column (or per row with `axis="columns"`), and `.map()` for true element-wise application of a Python function to every scalar value. Use `.apply()` when your function reduces a column/row to a summary (or a small Series of summaries); use `.map()` when you want to transform every individual cell (e.g., formatting).

```python
frame = pd.DataFrame(np.random.standard_normal((4, 3)), columns=list("bde"),
                      index=["Utah", "Ohio", "Texas", "Oregon"])

def f1(x):
    return x.max() - x.min()

frame.apply(f1)               # applied once per COLUMN -> Series indexed by column name
frame.apply(f1, axis="columns")  # applied once per ROW instead

# apply can also return a Series with multiple values per column
def f2(x):
    return pd.Series([x.min(), x.max()], index=["min", "max"])
frame.apply(f2)                # -> DataFrame with "min"/"max" rows

# element-wise transform of every scalar value
frame.map(lambda x: f"{x:.2f}")
```

Note: common aggregate stats (`sum`, `mean`, `std`, etc.) are already built-in DataFrame methods, so `.apply()` is only needed for custom logic — reach for the built-in first since it's faster (vectorized in C) and more readable.

### Sorting

`sort_index` sorts by the row or column **labels**; `sort_values` sorts by the actual **data values**. Both return a new sorted object by default (ascending); pass `ascending=False` to reverse.

```python
obj = pd.Series(np.arange(4), index=["d", "a", "b", "c"])
obj.sort_index()                 # sort by index labels

frame = pd.DataFrame(np.arange(8).reshape((2, 4)), index=["three", "one"], columns=["d", "a", "b", "c"])
frame.sort_index()                    # sort rows by label
frame.sort_index(axis="columns")      # sort columns by label
frame.sort_index(axis="columns", ascending=False)  # descending

obj = pd.Series([4, 7, -3, 2])
obj.sort_values()                # sort by value; NaNs go to the end by default

obj = pd.Series([4, np.nan, 7, np.nan, -3.2])
obj.sort_values(na_position="first")   # push NaNs to the front instead
```

---

## Data Loading & I/O

Before you can analyze anything, you need to get data into pandas. This section covers reading delimited text (CSV) files, handling messy/missing data on load, reading huge files in manageable pieces, writing data back out, low-level parsing with Python's `csv` module, and working with JSON.

### `pandas.read_csv` — The Workhorse Reader

`read_csv` is the most common way to load tabular data into a DataFrame. Pandas has similar `read_*` functions for other formats (`read_excel`, `read_json`, `read_html`, `read_parquet`, `read_sql`, etc.), but they all share the same philosophy: infer as much as possible (column types, delimiters), while giving you plenty of keyword arguments to override that inference when your file is messy — which real-world files usually are.

```python
import pandas as pd

# Simplest case: comma-delimited file with a header row
df = pd.read_csv("examples/ex1.csv")
```

Useful because real data almost never arrives "clean" — knowing the full menu of `read_csv` options (below) means you rarely need to hand-write a parser.

### Controlling Headers and Column Names

If a file has no header row, tell pandas explicitly — otherwise it will treat your first data row as column names.

```python
# No header in the file: let pandas auto-name columns 0, 1, 2, ...
pd.read_csv("examples/ex2.csv", header=None)

# Or supply your own column names
pd.read_csv("examples/ex2.csv", names=["a", "b", "c", "d", "message"])
```

### Setting the Index While Reading

You can promote one or more columns to be the row index right at load time with `index_col`, instead of loading then calling `.set_index()` afterward.

```python
names = ["a", "b", "c", "d", "message"]

# Single column as index
pd.read_csv("examples/ex2.csv", names=names, index_col="message")

# Multiple columns -> hierarchical (MultiIndex)
parsed = pd.read_csv("examples/csv_mindex.csv", index_col=["key1", "key2"])
```

This is handy when a column is naturally a unique identifier (like a "message" label or a composite key) — indexing on it up front makes later `.loc` lookups and joins simpler.

### Non-Comma Delimiters and Whitespace-Separated Data

Not every "flat file" uses commas. When fields are separated by variable amounts of whitespace, pass a regex to `sep`.

```python
# \s+ matches one-or-more whitespace characters as the delimiter
result = pd.read_csv("examples/ex3.txt", sep=r"\s+")
```

Note: if there's one fewer column name than data columns, pandas assumes the first column is meant to be the index — a small but easy-to-forget inference rule.

### Skipping Rows

Use `skiprows` to ignore junk at the top of a file (titles, comments, blank separator rows) — common in files exported from Excel or other reporting tools.

```python
pd.read_csv("examples/ex4.csv", skiprows=[0, 2, 3])  # skip specific row numbers (0-indexed)
```

### Handling Missing Data on Load

Pandas recognizes common "missing" sentinels (`NA`, `NULL`, empty string, etc.) automatically and converts them to `NaN`. This matters because how missing data gets encoded is inconsistent across sources, and getting it wrong silently corrupts downstream math (a string `"NA"` sitting in a numeric column, for example).

```python
result = pd.read_csv("examples/ex5.csv")
pd.isna(result)          # boolean mask of what pandas considered missing

# Add extra strings to the default missing-value list
pd.read_csv("examples/ex5.csv", na_values=["NULL"])

# Turn OFF the default sentinel list entirely
pd.read_csv("examples/ex5.csv", keep_default_na=False)

# Turn off defaults but specify your own list
pd.read_csv("examples/ex5.csv", keep_default_na=False, na_values=["NA"])

# Per-column sentinels via a dict: different columns can have different "missing" markers
sentinels = {"message": ["foo", "message"], "something": ["two"]}
pd.read_csv("examples/ex5.csv", na_values=sentinels, keep_default_na=False)
```

**Why it's useful:** real datasets encode "missing" inconsistently (`"NA"`, `"n/a"`, `-999`, blank, `"two"` meaning an invalid sentinel for a specific column). Fine-grained control over what counts as missing lets you load data correctly instead of cleaning it up after the fact.

### Key `read_csv` Arguments (Cheat Sheet)

These are the options you'll reach for again and again:

| Argument | What it does |
|---|---|
| `sep` / `delimiter` | Character or regex used to split fields |
| `header` | Row to use as column names (`None` if no header row) |
| `names` | Explicit list of column names |
| `index_col` | Column(s) to use as the row index |
| `skiprows` | Rows to skip at the top of the file |
| `na_values` / `keep_default_na` | Customize what counts as missing |
| `parse_dates` | Parse column(s) as datetimes (can combine multiple columns) |
| `converters` | Dict of `{column: function}` to transform values while parsing |
| `nrows` | Only read the first N rows |
| `chunksize` | Read the file in pieces (see below) |
| `encoding` | Text encoding, e.g. `"utf-8"` |
| `thousands` | Character used as a thousands separator (e.g. `","`) |
| `engine` | Parser engine: `"c"`/`"pyarrow"` (fast), `"python"` (more flexible) |

### Reading Large Files in Pieces

For files too large to comfortably fit in memory (or when you just want a quick peek), pandas lets you read a limited number of rows, or iterate over the file in chunks.

```python
pd.options.display.max_rows = 10   # cosmetic: keep console output short

# Peek at just the first 5 rows without reading the whole file
pd.read_csv("examples/ex6.csv", nrows=5)

# Read the file in chunks of 1000 rows at a time
chunker = pd.read_csv("examples/ex6.csv", chunksize=1000)
print(type(chunker))   # TextFileReader — an iterable, not a DataFrame

# Aggregate across chunks: running value_counts on a "key" column
tot = pd.Series([], dtype="int64")
for piece in chunker:
    tot = tot.add(piece["key"].value_counts(), fill_value=0)

tot = tot.sort_values(ascending=False)
```

**Why it's useful:** this is the standard pattern for computing aggregates (counts, sums, sample statistics) over a dataset that doesn't fit in RAM — you never hold more than one chunk at a time, accumulating a small running result instead.

### Writing DataFrames Back Out (`to_csv`)

The mirror image of `read_csv` — with equally fine-grained control over delimiters, missing-value representation, and which rows/columns get written.

```python
import sys

data.to_csv("examples/out.csv")                 # write to a file
data.to_csv(sys.stdout, sep="|")                 # custom delimiter, printed to console
data.to_csv(sys.stdout, na_rep="NULL")           # write "NULL" instead of leaving blanks for NaN
data.to_csv(sys.stdout, index=False, header=False)      # drop row index and header row
data.to_csv(sys.stdout, index=False, columns=["a", "b", "c"])  # write only a subset/order of columns
```

Controlling `na_rep`, `index`, and `columns` matters when the output file is meant to be consumed by another system (a database loader, another team's script) that expects a specific shape and missing-value convention.

### Low-Level Parsing with the `csv` Module

`read_csv` handles the vast majority of cases, but malformed or unusually-delimited files sometimes need to be parsed by hand using Python's built-in `csv` module before you build a DataFrame yourself.

```python
import csv

with open("examples/ex7.csv") as f:
    reader = csv.reader(f)
    for line in reader:
        print(line)   # each line -> a list of string values, quote characters stripped

# Manually assemble a dict of columns from the raw rows
with open("examples/ex7.csv") as f:
    lines = list(csv.reader(f))

header, values = lines[0], lines[1:]
# zip(*values) transposes rows -> columns (careful: loads everything into memory)
data_dict = {h: v for h, v in zip(header, zip(*values))}
```

For files with a non-standard delimiter, quote character, or line terminator, define a custom `csv.Dialect`, or just pass the option directly:

```python
class my_dialect(csv.Dialect):
    lineterminator = "\n"
    delimiter = ":"
    quotechar = '"'
    quoting = csv.QUOTE_MINIMAL

# or, without a subclass:
reader = csv.reader(open("examples/ex7.csv"), delimiter="|")
```

**Why it's useful:** this is your escape hatch — when a file is too irregular for `read_csv`'s options to handle cleanly, dropping to the `csv` module gives you full row-by-row control before converting to a DataFrame.

### Working with JSON Data

JSON is the standard format for data moving over HTTP (web APIs, JS applications) — much more free-form than a flat CSV, since it supports nested objects and arrays. Python's built-in `json` module converts between JSON text and native Python objects (dicts/lists), which you then shape into a DataFrame yourself.

```python
import json

obj = """
{"name": "Wes",
 "cities_lived": ["Akron", "Nashville", "New York", "San Francisco"],
 "pet": null,
 "siblings": [{"name": "Scott", "age": 34, "hobbies": ["guitars", "soccer"]},
              {"name": "Katie", "age": 42, "hobbies": ["diving", "art"]}]}
"""

result = json.loads(obj)      # JSON string -> Python dict/list structure
asjson = json.dumps(result)   # Python object -> JSON string

# Pick out a nested list of records and load it directly into a DataFrame
siblings = pd.DataFrame(result["siblings"], columns=["name", "age"])
```

**Why it's useful:** JSON's nesting doesn't map onto a flat table automatically — you decide which part of the structure becomes rows/columns. This is the pattern you'll use constantly when pulling data out of web API responses. (Note: pandas also has `pd.read_json`/`.to_json` for when the whole file is already row-shaped JSON — worth learning as a faster alternative to manual `json.loads` + DataFrame construction for simple cases.)

---

## Interacting with Databases

In real-world data science work, data often lives in SQL-based relational databases (SQL Server, PostgreSQL, MySQL, SQLite, etc.) rather than flat CSV/Excel files. pandas provides convenient functions for pulling query results directly into a DataFrame, so you can go from "rows in a database" to "data ready for analysis" in one step.

### Connecting to a database and running raw SQL (`sqlite3`)

Python's built-in `sqlite3` module lets you connect to a SQLite database file, execute SQL statements (like `CREATE TABLE` and `INSERT`), and fetch results as plain Python tuples. This is useful to understand because it's the low-level foundation that higher-level tools (like SQLAlchemy) build on — and SQLite itself is handy for lightweight, file-based databases with no server setup required.

```python
import pandas as pd
import sqlite3

# Create/connect to a local SQLite database file
con = sqlite3.connect("mydata.sqlite")

# Define a table schema and create it
query = """
CREATE TABLE test
(a VARCHAR(20), b VARCHAR(20),
 c REAL,        d INTEGER
);"""
con.execute(query)
con.commit()  # commit the DDL change

# Insert multiple rows at once using parameterized placeholders (?)
# Parameterized queries avoid SQL injection and handle type conversion safely
data = [("Atlanta", "Georgia", 1.25, 6),
        ("Tallahassee", "Florida", 2.6, 3),
        ("Sacramento", "California", 1.7, 5)]
stmt = "INSERT INTO test VALUES(?, ?, ?, ?)"
con.executemany(stmt, data)
con.commit()
```

### Fetching query results manually into a DataFrame

With raw `sqlite3`, query results come back as a list of tuples, not a DataFrame — you have to build the DataFrame yourself, pulling column names off the cursor's `.description` attribute. This is worth knowing because it shows *why* pandas' built-in SQL readers are so useful: without them, you'd repeat this "munging" step every time you query.

```python
cursor = con.execute("SELECT * FROM test")
rows = cursor.fetchall()  # list of tuples, e.g. [('Atlanta', 'Georgia', 1.25, 6), ...]

# cursor.description holds column metadata; x[0] is the column name
columns = [x[0] for x in cursor.description]
df = pd.DataFrame(rows, columns=columns)
```

### Reading SQL directly into a DataFrame with SQLAlchemy + `pd.read_sql`

Manually converting cursor results to a DataFrame every time is tedious and database-specific (different SQL backends have subtly different data type/column behaviors). **SQLAlchemy** is a popular Python SQL toolkit that abstracts over these differences across database engines (SQLite, PostgreSQL, MySQL, etc.), and pandas' `pd.read_sql` function uses a SQLAlchemy connection ("engine") to run a query and return a ready-to-use DataFrame in one call. This is the pattern you'll use most often in practice — it's concise, portable across database backends, and skips the manual cursor-to-DataFrame conversion entirely.

```python
import sqlalchemy as sqla

# Create a SQLAlchemy engine pointing at the SQLite file
# The connection string format is "dialect:///path/to/file"
db = sqla.create_engine("sqlite:///mydata.sqlite")

# Run the query and get a DataFrame directly, with correct column names/types
df = pd.read_sql("SELECT * FROM test", db)
```

**Why this matters for a data scientist:** in practice you'll rarely hand-build DataFrames from cursor results — `pd.read_sql(query, engine)` is the standard way to pull data from a production database (Postgres, MySQL, etc.) into pandas for analysis, and because SQLAlchemy engines work the same way across database backends, the same code pattern works no matter which SQL database your organization uses.

---

## Data Cleaning & Preparation

Real-world data is messy: values are missing, rows are duplicated, categories are inconsistently spelled, and numeric columns hide outliers. This section covers pandas' toolkit for getting raw data into an analysis-ready shape.

### Handling Missing Data

#### Detecting Missing Values
pandas represents missing (null) data with the floating-point sentinel `NaN` (and Python's `None` is treated the same way). Almost every descriptive statistic in pandas silently excludes NA values by default, so knowing where your NAs live matters before you trust a `.mean()` or `.sum()`.

| Method | Description |
|---|---|
| `isna()` | Boolean mask, `True` where a value is missing |
| `notna()` | Negation of `isna()` |
| `dropna()` | Drop labels (rows/columns) with missing data |
| `fillna()` | Fill missing values with a constant or interpolation method |

```python
import numpy as np
import pandas as pd

float_data = pd.Series([1.2, -3.5, np.nan, 0])
float_data.isna()          # True at the NaN position

string_data = pd.Series(["aardvark", np.nan, None, "avocado"])
string_data.isna()         # both np.nan and None register as missing
```

#### Filtering Out Missing Data (`dropna`)
Useful when a row/column with missing values can't be meaningfully imputed and is safer to discard than to guess at.

```python
data = pd.DataFrame([[1., 6.5, 3.],
                     [1., np.nan, np.nan],
                     [np.nan, np.nan, np.nan],
                     [np.nan, 6.5, 3.]])

data.dropna()                       # drops ANY row containing a NaN (default)
data.dropna(how="all")              # only drops rows that are ENTIRELY NaN
data.dropna(axis="columns", how="all")  # same idea, but for columns
data.dropna(thresh=2)               # keep rows with at least 2 non-NA values
```

#### Filling in Missing Data (`fillna`)
Filtering throws away potentially useful non-null values in the same row. `fillna` lets you patch the holes instead — with a constant, a per-column value, a forward/backward fill, or a summary statistic (mean/median imputation).

```python
df.fillna(0)                        # replace all NaNs with 0
df.fillna({1: 0.5, 2: 0})           # different fill value per column
df.ffill()                          # forward-fill (carry last valid value down)
df.ffill(limit=2)                   # cap how many consecutive NaNs get filled
data.fillna(data.mean())            # classic mean imputation
```

### Data Transformation

#### Removing Duplicates
Duplicate rows commonly creep in from merged data sources, repeated ETL runs, or logging systems that double-record events.

```python
data = pd.DataFrame({"k1": ["one", "two"] * 3 + ["two"],
                     "k2": [1, 1, 2, 3, 3, 4, 4]})

data.duplicated()                   # Boolean Series: True = seen before
data.drop_duplicates()              # drop exact duplicate rows
data.drop_duplicates(subset=["k1"]) # dedupe based on a subset of columns
data.drop_duplicates(["k1", "k2"], keep="last")  # keep the LAST occurrence instead of the first
```

#### Transforming Data with `map`
`map` applies a function or dictionary lookup element-wise on a Series — perfect for deriving a new column from an existing categorical one (e.g., looking up a category from a raw label).

```python
data = pd.DataFrame({"food": ["bacon", "pulled pork", "nova lox"],
                     "ounces": [4, 3, 6]})

meat_to_animal = {"bacon": "pig", "pulled pork": "pig", "nova lox": "salmon"}
data["animal"] = data["food"].map(meat_to_animal)   # dict-based mapping

# equivalently, with a function:
data["food"].map(lambda x: meat_to_animal[x])
```

#### Replacing Values
`replace` is a more general and flexible tool than `map` for swapping out specific values — most commonly used to turn sentinel "junk" values (like `-999`) into proper `NaN`s pandas understands.

```python
data = pd.Series([1., -999., -1000., 3.])

data.replace(-999, np.nan)                    # single value
data.replace([-999, -1000], np.nan)           # multiple values -> one replacement
data.replace([-999, -1000], [np.nan, 0])      # multiple values -> matching replacements
data.replace({-999: np.nan, -1000: 0})        # dict form (clearest for multiple mappings)
```

#### Renaming Axis Indexes
Row/column labels can be transformed just like values — useful for standardizing casing, trimming labels, or giving a subset of columns human-readable names without rebuilding the DataFrame from scratch.

```python
data = pd.DataFrame(np.arange(12).reshape((3, 4)),
                    index=["Ohio", "Colorado", "New York"],
                    columns=["one", "two", "three", "four"])

data.index = data.index.map(lambda x: x[:4].upper())   # in-place style transform

# rename() returns a NEW object and leaves the original untouched:
data.rename(index=str.title, columns=str.upper)
data.rename(index={"OHIO": "INDIANA"}, columns={"three": "peekaboo"})  # partial rename via dict
```

### Discretization and Binning

Continuous numeric data (age, income, a measurement) is often more useful for analysis/modeling when grouped into discrete buckets — e.g., age brackets for a demographic breakdown.

```python
ages = [20, 22, 25, 27, 21, 23, 37, 31, 61, 45, 41, 32]
bins = [18, 25, 35, 60, 100]

age_categories = pd.cut(ages, bins)      # bin by EXPLICIT edges
age_categories.codes                     # integer bin index for each value
age_categories.categories                # the Interval objects describing each bin

pd.cut(ages, bins, right=False)                       # make bins left-inclusive instead
group_names = ["Youth", "YoungAdult", "MiddleAged", "Senior"]
pd.cut(ages, bins, labels=group_names)                # human-readable bin labels

pd.cut(np.random.uniform(size=20), 4, precision=2)    # 4 EQUAL-WIDTH bins (auto edges)
pd.qcut(np.random.standard_normal(1000), 4)           # 4 EQUAL-SIZE bins by quantile
```

`cut` gives equal-*width* bins (bin counts can be unbalanced), while `qcut` gives equal-*count* bins based on quantiles (bin widths can be unbalanced). Use `qcut` when you want roughly the same number of observations per bucket, e.g. for building balanced strata.

### Detecting and Filtering Outliers

Outliers can distort means, standard deviations, and model training. A common workflow: find values beyond some threshold, inspect the rows they belong to, then cap ("winsorize") or remove them.

```python
data = pd.DataFrame(np.random.standard_normal((1000, 4)))

col = data[2]
col[col.abs() > 3]                              # outliers in a single column

data[(data.abs() > 3).any(axis="columns")]       # rows with an outlier in ANY column
# note: parentheses around the comparison are required before calling .any()

data[data.abs() > 3] = np.sign(data) * 3         # cap ("winsorize") values at +/-3
```

### Permutation and Random Sampling

Randomly reordering or subsetting rows is a building block for train/test splitting, bootstrapping, and shuffling data to remove ordering bias before analysis.

```python
df = pd.DataFrame(np.arange(5 * 7).reshape((5, 7)))

sampler = np.random.permutation(5)     # random reordering of row positions
df.take(sampler)                       # == df.iloc[sampler]

df.take(np.random.permutation(7), axis="columns")   # permute columns instead of rows

df.sample(n=3)                          # random subset, WITHOUT replacement
choices = pd.Series([5, 7, -1, 6, 4])
choices.sample(n=10, replace=True)      # bootstrap-style sample, WITH replacement
```

### Computing Indicator/Dummy Variables

Many statistical and machine learning models can't consume raw text categories directly — they need numeric input. `get_dummies` performs one-hot encoding: each distinct category becomes its own 0/1 column.

```python
df = pd.DataFrame({"key": ["b", "b", "a", "c", "a", "b"], "data1": range(6)})

pd.get_dummies(df["key"])                          # one column per category
dummies = pd.get_dummies(df["key"], prefix="key")   # prefix to avoid name collisions
df_with_dummy = df[["data1"]].join(dummies)         # merge back onto the original data

# Multi-label categories (a row can belong to several categories at once, e.g. movie genres
# stored as "Comedy|Romance") need str.get_dummies with the delimiter instead:
movies["genres"].str.get_dummies("|")

# Handy combo: discretize a continuous variable, then one-hot encode the bins
pd.get_dummies(pd.cut(values, bins=[0, 0.2, 0.4, 0.6, 0.8, 1]))
```

### Extension Data Types

NumPy's native types can't represent a missing *integer* (only float `NaN`), which silently upcasts int columns to float when NAs appear. pandas' extension types (`Int64`, `string`, etc.) fix this by supporting a real `pd.NA` missing-value marker while preserving the original dtype.

```python
s = pd.Series([1, 2, 3, None])
s.dtype                                   # float64 -- forced upcast, not what we wanted

s = pd.Series([1, 2, 3, None], dtype="Int64")   # capital "I" -> nullable integer extension type
s.isna()                                  # last entry is True
s[3]                                      # <NA>, pandas' own missing-value sentinel

s = pd.Series(["one", "two", None, "three"], dtype="string")  # nullable string extension type
```

### String Manipulation

#### Built-in Python String Methods
For simple, single-string munging, Python's own string methods are usually enough and are faster/clearer than reaching for regex.

```python
val = "a, b,     guido"
pieces = [x.strip() for x in val.split(",")]   # split + strip whitespace
"::".join(pieces)                              # rejoin with a new delimiter

"guido" in val          # substring test (preferred over find/index for a simple check)
val.index(",")           # position of substring; raises ValueError if not found
val.find(":")             # position of substring; returns -1 if not found (no exception)
val.count(",")            # number of occurrences
```

#### Regular Expressions (`re` module)
When patterns are more complex than a fixed delimiter (variable whitespace, optional characters, multiple possible formats), regular expressions give you a compact pattern-matching language. The `re` module's functions fall into three families: matching, substitution, and splitting.

```python
import re

text = "foo         bar\t baz   \tqux"
re.split(r"\s+", text)          # split on ANY run of whitespace (spaces, tabs, etc.)

# Compile once, reuse many times -- faster when applying the same pattern repeatedly
regex = re.compile(r"\s+")
regex.split(text)
```

---

## Data Wrangling: Join, Combine & Reshape

Real-world data is rarely in one tidy table — it's split across files, databases, or shaped inconveniently for analysis. This section covers the core pandas tools for combining datasets (merging, joining) and reshaping them (hierarchical indexing, stacking).

### Hierarchical Indexing (MultiIndex)

A `MultiIndex` lets an axis (rows or columns) carry two or more levels of labels. This is how pandas represents higher-dimensional data (e.g., "state + color", or "key1 + key2") inside a 1-D `Series` or 2-D `DataFrame`, instead of needing a true N-dimensional structure. It's the foundation for reshaping and for group-based operations like pivot tables.

```python
import pandas as pd
import numpy as np

# A Series with a list-of-lists as the index becomes a MultiIndex
data = pd.Series(
    np.random.uniform(size=9),
    index=[["a", "a", "a", "b", "b", "c", "c", "d", "d"],
           [1, 2, 3, 1, 3, 1, 2, 2, 3]],
)
# a  1    0.1464
#    2    0.4064
#    3    0.9978
# b  1    0.9174
# ...

# Partial indexing: select by the outer level
data["b"]          # rows where outer level == "b"
data["b":"c"]       # slice across outer level
data.loc[["b", "d"]]

# Select by an INNER level using .loc[:, level_value]
data.loc[:, 2]      # all rows where the second index level == 2
```

A DataFrame can have a MultiIndex on either or both axes, and each level can be given a name for readability:

```python
frame = pd.DataFrame(
    np.arange(12).reshape((4, 3)),
    index=[["a", "a", "b", "b"], [1, 2, 1, 2]],
    columns=[["Ohio", "Ohio", "Colorado"], ["Green", "Red", "Green"]],
)
frame.index.names = ["key1", "key2"]
frame.columns.names = ["state", "color"]

frame["Ohio"]          # partial column selection works too
frame.index.nlevels    # 2

# Build a MultiIndex standalone (useful for reusing across objects)
pd.MultiIndex.from_arrays(
    [["Ohio", "Ohio", "Colorado"], ["Green", "Red", "Green"]],
    names=["state", "color"],
)
```

#### Reordering and Sorting Levels

`swaplevel` exchanges two index levels without touching the data — useful when you want a different level to drive slicing or display order. `sort_index(level=...)` sorts by a specific level, which is often needed after a swap since pandas doesn't auto-sort for performance reasons.

```python
frame.swaplevel("key1", "key2")           # swap by name
frame.sort_index(level=1)                  # sort by the 2nd level only
frame.swaplevel(0, 1).sort_index(level=0)  # common combo: swap then sort
```

#### Summary Statistics by Level

Aggregation methods can operate on a single index level via `groupby(level=...)`, letting you roll up a MultiIndex axis without first collapsing it.

```python
frame.groupby(level="key2").sum()          # aggregate rows by the "key2" level
frame.T.groupby(level="color").sum()       # transpose first to aggregate a column level
```

#### Using DataFrame Columns as the Index (`set_index` / `reset_index`)

You'll often want to promote one or more columns into the row index (e.g., to enable hierarchical lookups or prep for reshaping), or do the reverse to get a "flat" DataFrame back for output or merging. `set_index` and `reset_index` are inverses of each other.

```python
frame = pd.DataFrame({
    "a": range(7), "b": range(7, 0, -1),
    "c": ["one", "one", "one", "two", "two", "two", "two"],
    "d": [0, 1, 2, 0, 1, 2, 3],
})

frame2 = frame.set_index(["c", "d"])          # c, d become a MultiIndex; dropped from columns
frame.set_index(["c", "d"], drop=False)        # keep c, d as columns too

frame2.reset_index()                            # move index levels back into columns
```

### Combining and Merging Datasets

Pandas gives you three main ways to combine objects, each suited to a different situation:

| Tool | Use case |
|---|---|
| `pd.merge` | SQL-style join: line up rows across DataFrames using shared key column(s) |
| `pd.concat` | Stack/glue objects together along an axis (no key matching) |
| `combine_first` | Patch missing values in one object using values from another, aligned by index |

#### Database-Style Joins with `pd.merge`

`pd.merge` links rows between two DataFrames based on one or more key columns — exactly like a SQL `JOIN`. This is the tool to reach for whenever you have related data spread across two tables (e.g., transactions + customer lookup) and need to combine them into one.

```python
df1 = pd.DataFrame({"key": ["b", "b", "a", "c", "a", "a", "b"],
                     "data1": pd.Series(range(7), dtype="Int64")})
df2 = pd.DataFrame({"key": ["a", "b", "d"],
                     "data2": pd.Series(range(3), dtype="Int64")})

# Best practice: always name the join key explicitly with `on`
pd.merge(df1, df2, on="key")     # inner join by default

# If key columns have different names in each frame:
pd.merge(df3, df4, left_on="lkey", right_on="rkey")
```

**Join types** — controlled with `how`:

| `how=` | Behavior |
|---|---|
| `"inner"` (default) | keep only keys present in **both** frames |
| `"left"` | keep all keys from the left frame |
| `"right"` | keep all keys from the right frame |
| `"outer"` | keep the union of keys from both frames (fills gaps with `NaN`/`<NA>`) |

```python
pd.merge(df1, df2, how="outer")   # union of keys; unmatched rows get <NA>
```

- **Many-to-one** merge: one side has duplicate keys, the other has unique keys — each duplicate gets matched to the single row on the other side.
- **Many-to-many** merge: both sides have duplicate keys — the result is the **Cartesian product** of matching rows for each key (row counts can explode, so be intentional about this).

```python
# many-to-many example: 3 "b" rows on the left x 2 "b" rows on the right -> 6 "b" rows out
pd.merge(df1, df2, on="key", how="left")
```

Merging on **multiple keys** just means passing a list to `on` — the join key becomes the combination of all listed columns:

```python
pd.merge(left, right, on=["key1", "key2"], how="outer")
```

#### Combining Overlapping Data (motivation for `combine_first`)

Sometimes two datasets share the same index but each has holes (`NaN`) in different places — this isn't really a merge/join situation since there's no "key" to match on, just index alignment. NumPy's `np.where` shows the underlying logic: for each position, take the value from `a` unless it's missing, in which case fall back to `b`.

```python
a = pd.Series([np.nan, 2.5, 0.0, 3.5, 4.5, np.nan],
              index=["f", "e", "d", "c", "b", "a"])
b = pd.Series([0., np.nan, 2., np.nan, np.nan, 5.],
              index=["a", "b", "c", "d", "e", "f"])

np.where(pd.isna(a), b, a)   # elementwise "prefer a, fall back to b"
```

This pattern — patching gaps in one Series/DataFrame with values from another, aligned by label — is exactly what `Series.combine_first()` / `DataFrame.combine_first()` automate (matching on index labels rather than raw array position, unlike `np.where`).

> **Note:** the source notebook cuts off right after introducing this motivating example (a kernel crash interrupted the session before `combine_first`, `pd.concat`, and the `stack`/`unstack`/pivot reshaping material were written up). Revisit this notebook to fill in: `combine_first`, `pandas.concat` (axis stacking, keys, join types), and reshaping with `stack`/`unstack`, `pivot`, `melt`, and `wide_to_long`.

---

## Correlation & Covariance

> Note: This notebook (`CorrCov.ipynb`) sets up an example using stock price/volume data pickled from Yahoo Finance (`yahoo_price.pkl`, `yahoo_volume.pkl` in the `Pandas/` folder), but the `pd.read_pickle()` call errors out (`ModuleNotFoundError: No module named 'pandas.indexes'`) because the pickle files were saved with a much older pandas version. The notebook itself never got past this point, so the sections below fill in the standard corr/cov workflow this exercise was building toward — the same one used throughout *Python for Data Analysis* — using DataFrame-of-returns as the running example.

### Why correlation & covariance matter

Correlation and covariance are pairwise summary statistics — unlike `.mean()` or `.std()`, which describe a single column, these describe how **two columns move together**. For a data scientist this is the fastest way to spot redundant features, find relationships worth investigating, or (in a finance context, like this notebook's stock data) see which assets tend to rise and fall together — useful for diversification and risk analysis.

- **Covariance** tells you the *direction* of the relationship (positive vs negative) but its magnitude isn't standardized, so it's hard to compare across pairs of variables with different scales.
- **Correlation** is covariance normalized to a range of -1 to 1, which makes it interpretable and comparable across any pair of variables.

### Fixing the broken pickle load

Since the actual blocker in this notebook is a pandas version incompatibility, the practical fix (not part of the "concept" itself but useful to remember) is usually one of:

```python
import pandas as pd

# Option 1: re-save the pickle with the current pandas version once you can load it
# (e.g. load it in an environment with the matching old pandas version, then:)
# price.to_pickle("yahoo_price.pkl")

# Option 2: if the underlying data is small/simple, re-fetch it fresh instead of
# relying on a stale pickle, e.g. via pandas-datareader or a CSV export.
```

### `.pct_change()` — turning prices into returns

Raw prices aren't directly comparable across stocks (a $5 move on a $50 stock is very different from a $5 move on a $500 stock). Percent change converts a price series into **returns**, which puts every column on the same relative scale — this is almost always the first step before computing correlation/covariance on financial time series.

```python
# price: DataFrame of daily closing prices, columns = tickers, index = dates
returns = price.pct_change()   # row i = (price[i] - price[i-1]) / price[i-1]
returns.tail()
```

### `.corr()` — pairwise correlation

Called on a Series, `.corr(other)` returns a single correlation coefficient between two columns. Called on a whole DataFrame, `.corr()` returns a full correlation matrix between every pair of columns — a quick way to eyeball which variables move together.

```python
# Correlation between two specific columns
returns["MSFT"].corr(returns["IBM"])

# Full pairwise correlation matrix across all columns
returns.corr()
```

### `.cov()` — pairwise covariance

Same idea as `.corr()`, but returns covariance instead of the normalized correlation coefficient. Useful when you need the raw, scale-dependent relationship (e.g. as an input to portfolio variance calculations), rather than just direction and strength.

```python
returns["MSFT"].cov(returns["IBM"])

# Full covariance matrix
returns.cov()
```

### `.corrwith()` — correlate one column/Series against many

`.corrwith()` computes correlation between a DataFrame's columns and either a Series or another DataFrame's matching columns, all in one call — much faster than looping and calling `.corr()` column by column.

```python
# Correlate every column's returns against IBM's returns
returns.corrwith(returns["IBM"])

# Correlate matching columns between two DataFrames (e.g. returns vs. volume)
returns.corrwith(volume)

# axis="columns" correlates row-wise instead of column-wise
returns.corrwith(volume, axis="columns")
```

### Reading the results

- Values near **+1**: strongly move together (e.g. two tech stocks during a market rally).
- Values near **-1**: strongly move opposite each other.
- Values near **0**: little to no linear relationship.

In practice, a correlation matrix like `returns.corr()` is often the very first exploratory step on any new multi-column dataset — it's a cheap way to catch multicollinearity before modeling, or to find which features carry redundant information.
