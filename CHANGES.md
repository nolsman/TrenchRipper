# TrenchRipper Changes Report

This document describes the changes made to the TrenchRipper codebase to support modern versions of Dask (2025.10.0+) and related dependencies. These changes were necessary due to breaking API changes in Dask's DataFrame operations.

## Summary of Changes

### Core Motivation
The original TrenchRipper codebase was written for older versions of Dask that have since deprecated or removed several APIs. The main updates address:

1. **Dask Parquet API changes** - Removal of `calculate_divisions` parameter
2. **Dask DataFrame index API changes** - `sorted` parameter renamed to `sort`
3. **Parquet engine migration** - Changed from `fastparquet` to `pyarrow` engine
4. **Synchronization improvements** - Added explicit `wait()` calls for better task coordination

---

## File-by-File Changes

### `trenchripper/kymograph.py`
**Lines changed: ~513 additions/deletions**

This file received the most extensive modifications as it contains the core kymograph processing pipeline.

#### API Updates (Throughout file)
```python
# OLD (deprecated):
df = dd.read_parquet(path, calculate_divisions=True)
df = df.set_index("column", sorted=True)
dd.to_parquet(df, path, engine='fastparquet', ...)

# NEW:
df = dd.read_parquet(path)
df = df.set_index("column", sort=True)
dd.to_parquet(df, path, engine='pyarrow', ...)
```

#### `add_trenchids()` - Major Rewrite
**Purpose**: Assigns unique sequential IDs to each trench (fov/row/trench combination)

**Problem Solved**: The original implementation relied on index ordering assumptions that could produce incorrect trenchid assignments when keys weren't naturally sorted.

**Changes**:
- Added explicit sorting of unique keys before assignment
- Added diagnostic print statements to detect sorting issues
- Changed from `set_index(sorted=True)` groupby approach to explicit mapping creation

```python
# Key fix: Ensure keys are sorted before creating ID mapping
unique_keys = df["key"].unique().compute()
unique_keys_list = list(unique_keys)
unique_keys = sorted(unique_keys_list)  # Critical fix

trenchid_mapping = pd.DataFrame({
    "key": unique_keys,
    "trenchid": range(len(unique_keys))
}).set_index("key")
```

#### `reindex_trenches()` - Complete Rewrite
**Purpose**: Re-numbers trenches after filtering to maintain sequential indices

**Problem Solved**: Original used `repartition(divisions=...)` which no longer works with modern Dask when divisions aren't computed.

**Changes**:
- Removed reliance on DataFrame divisions
- Uses distributed merge instead of manual index manipulation
- Added `meta=` parameter for `apply()` operations (required in newer Dask)

```python
# Creates mapping of (fov-row, original_trench) -> new_trench_rank
unique_combos = df[["fov-row Index", "trench"]].drop_duplicates().compute()
unique_combos = unique_combos.sort_values(["fov-row Index", "trench"])
unique_combos["trench_rank"] = unique_combos.groupby("fov-row Index")["trench"].transform(
    lambda x: pd.factorize(x)[0]
)
# Merge back to main dataframe
df = df.merge(unique_combos_dd, on=["fov-row Index", "trench"], how="left")
```

#### `reorg_kymograph()` - Efficiency Update
**Purpose**: Reorganizes kymograph data from per-file to per-trenchid structure

**Changes**:
- Replaced Dask `read_parquet().loc[]` with PyArrow dataset for filtered reading
- Significantly improves performance by filtering at read time

```python
# NEW: Efficient filtered reading with PyArrow
import pyarrow.dataset as ds
dataset = ds.dataset(path + "/trenchiddf", format="parquet")
table = dataset.to_table(filter=ds.field("trenchid").isin(trenchids_list))
working_trenchdf = table.to_pandas()
```

#### `generate_kymograph()` and Related Methods
**Changes**:
- Added explicit `wait()` calls before parquet writes
- Added progress print statements
- Changed all parquet operations to use `pyarrow` engine

---

### `trenchripper/analysis.py`
**Lines changed: ~558 additions/deletions**

#### `regionprops_extractor.analyze_all_files()` - Major Rewrite
**Purpose**: Extracts region properties from segmented images

**Changes**:
- Added progress counter showing completion status
- Added explicit `wait()` calls between pipeline stages
- Added step-by-step progress messages
- Fixed `set_index(sorted=False)` to `set_index(sort=False)`

```python
# Progress reporting during processing
total = len(all_delayed_futures)
while any(future.status == "pending" for future in all_delayed_futures):
    statuses = Counter(f.status for f in all_delayed_futures)
    print(f"Progress: {statuses.get('finished', 0)}/{total} finished", end="\r")
    sleep(5)
```

#### `get_image_measurements()` - Simplified
**Purpose**: Computes image-based measurements per file

**Problem Solved**: Original passed `n_partitions` and `divisions` parameters that could be `None` in modern Dask.

**Changes**:
- Removed `n_partitions` and `divisions` parameters
- Uses simple boolean filtering instead of complex index slicing

```python
# OLD: Complex index-based selection
df = df.set_index("File Parquet Index", sorted=True, npartitions=n_partitions, divisions=divisions)
working_filedf = df.loc[start_idx:end_idx].compute()

# NEW: Simple filtering
df = dd.read_parquet(kymographpath + "/metadata")
working_filedf = df[df["File Index"] == file_idx].compute()
```

#### `get_all_image_measurements()` - Major Rewrite
**Changes**:
- Added progress reporting
- Added error reporting for failed futures
- Changed from `dd.from_delayed()` to gathering results and concatenating in pandas
- More robust error handling

---

### `trenchripper/marlin.py`
**Lines changed: ~78 additions/deletions**

#### `fish_analysis.__init__()` - API Updates
**Changes**:
- Replaced `calculate_divisions=True` usage
- Uses boolean filtering (`isin()`) instead of `.loc[]` on index
- Computes to pandas before setting MultiIndex

```python
# OLD:
kymograph_metadata = dd.read_parquet(path, calculate_divisions=True)
kymograph_metadata = kymograph_metadata.set_index("trenchid", sorted=True)
kymograph_metadata = kymograph_metadata.loc[selected_trenchids]

# NEW:
kymograph_metadata = dd.read_parquet(path)
kymograph_metadata = kymograph_metadata[kymograph_metadata["trenchid"].isin(selected_trenchids)]
self.kymograph_metadata_subsample = kymograph_metadata.compute()
self.kymograph_metadata_subsample = self.kymograph_metadata_subsample.set_index(["trenchid", "timepoints"])
```

#### `get_barcode_df()` - Type Handling Fix
**Problem Solved**: Barcode data could be returned as list, string, or array depending on Dask version.

**Changes**:
- Added type-checking function to handle multiple input types

```python
def to_array(x):
    if isinstance(x, list):
        return np.array(x)
    elif isinstance(x, str):
        return np.array(eval(x))
    else:
        return np.array(x)

barcodes = barcodes.apply(to_array)
```

#### `get_nanopore_df()` - Format Change
**Change**: Switched from CSV to parquet format for nanopore data
```python
# OLD:
nanopore_df = pd.read_csv(self.nanoporedfpath, delimiter="\t", index_col=0)

# NEW:
nanopore_df = pd.read_parquet(self.nanoporedfpath)
```

---

### `trenchripper/interactive.py`
**Lines changed: ~41**

#### Default Parameter Value Changes
The interactive widgets have updated default values, likely tuned for specific datasets. These are user-preference changes, not API fixes:

| Parameter | Old Default | New Default |
|-----------|-------------|-------------|
| `y_percentile` | 99 | 95 |
| `y_foreground_percentile` | 80 | 65 |
| `smoothing_kernel_y_dim_0` | 29 | 51 |
| `y_percentile_threshold` | 0.2 | 0.5 |
| `expected_num_rows` | 2 | 10 |
| `trench_len_y` | 270 | 130 |
| `orientation_detection` | 0 | 1 |
| `background_kernel_x` | 21 | 51 |
| `smoothing_kernel_x` | 9 | 5 |
| `otsu_scaling` | 0.25 | 0.0 |
| `min_threshold` | 0 | 150 |
| `trench_width_x` | 30 | 20 |
| `trench_present_thr` | 0.0 | 0.8 |
| Various segmentation params | (various) | (various) |

---

### `trenchripper/daskutils.py`, `utils.py`, `tracking.py`, `segment.py`, etc.
**Changes**: File mode changed from 644 to 755 (made executable). No code changes.

---

### `.gitignore`
**Changes**: Updated to properly exclude Python cache and Jupyter checkpoint files.

---

## Environment Dependencies

The code has been tested with and requires:
- **Dask**: 2025.10.0 (pinned - older versions have incompatible API)
- **Dask-distributed**: 2025.10.0 (pinned)
- **PyArrow**: >= 10.0.0
- **Python**: >= 3.10

**Important**: This code requires Dask 2025.10.0 specifically. Earlier versions of Dask have incompatible API changes (e.g., `calculate_divisions` parameter, `sorted` vs `sort` in `set_index()`).

The `environment.yml` pins the exact Dask version to ensure compatibility.

---

## Migration Notes

### For Users of the Original TrenchRipper
If migrating existing pipelines:

1. **Re-run focus filtering** after updating - the trenchid assignment fix may change ID mappings
2. **Check parquet files** - files written with `fastparquet` should still be readable with `pyarrow`
3. **Monitor memory usage** - the new `wait()` calls may change memory patterns

### Known Behavioral Changes
1. **Trenchid ordering**: Now guaranteed to be sorted, which may differ from previous runs
2. **Progress output**: More verbose logging during processing
3. **Error handling**: Failed futures now report errors instead of silent failure

---

## Testing

The changes have been validated using the `1_Core_Image_Processing.ipynb` notebook which demonstrates the standard analysis pipeline.

---

## Contact

For questions about these changes, please open an issue on the fork repository.
