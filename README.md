# NSE Securities Categorization Script

## Overview
This Jupyter notebook (`main.ipynb`) processes securities data from the National Stock Exchange of India (NSE) to categorize securities into trading categories: "5x", "3x", or "Only Delivery". The categorization is based on average traded value, price band percentage, F&O (Futures and Options) status, and variance data. This helps in determining leverage or margin requirements for trading.

## Prerequisites
### Required Libraries
- `pandas` for data manipulation
- `numpy` for numerical operations
- `pathlib` for file path handling

Install via pip if not already installed:
```
pip install pandas numpy
```

### Required Files
- `security.txt`: Main securities file with symbol, series, and price range data.
- `nse security headers.txt`: Optional file containing column headers for the securities file.
- `BhavCopy_NSE_CM (1).csv`, `BhavCopy_NSE_CM (2).csv`, `BhavCopy_NSE_CM (3).csv`: Bhav copy files for traded value data.
- `FNO_Underlyings.csv`: List of F&O underlying symbols.
- `C_VAR1_06112025_6112025.csv`: Variance data file (note: the code has a typo, it should be `C_VAR1_06112025_6.DAT` based on environment, but code uses `.csv`).

### Output File
- `securities_categorized.csv`: Final categorized securities data.

## Detailed Step-by-Step Walkthrough

### 1. Imports and Initial Setup
- Import necessary libraries: `pandas`, `numpy`, `pathlib`.
- Define file paths for input files:
  - `securities_file_path = "security.txt"`
  - `bhav_copy_files`: List of three Bhav copy CSV files.
  - `fno_underlyings_file_path = "FNO_Underlyings.csv"`
  - `var_data_file_path = "C_VAR1_06112025_6112025.csv"`
- Define output file: `output_csv_file = "securities_categorized.csv"`
- Define column names for consistency:
  - `symbol_column = "SYMBOL"`
  - `series_column = "SERIES"`
  - `traded_value_column = "TtlTrfVal"`
  - `price_range_column = "PRICE_RANGE"`
  - `bhav_symbol_column = "TckrSymb"`
  - `bhav_series_column = "SctySrs"`

### 2. Function Definitions

#### extract_price_band_percentage(price_range_string)
- Takes a string representing price range (e.g., "100-110").
- Handles missing or invalid data by returning 0.0.
- Splits the string by "-" or "–" to get lower and upper limits.
- Calculates midpoint as average of limits.
- Computes band percentage as ((upper - midpoint) / midpoint) * 100.
- Returns the percentage or 0.0 on error.

#### load_and_aggregate_bhav_data(bhav_file_list)
- Loads multiple Bhav copy CSV files.
- Checks if each file exists; skips if not.
- Reads each CSV, selects required columns: symbol, series, traded value.
- Renames columns to standard names.
- Concatenates all dataframes.
- Groups by symbol and aggregates:
  - Average traded value (mean of traded_value).
  - Last non-null series.
- Returns aggregated dataframe or empty if no files loaded.

#### load_fno_underlyings_list(fno_file_path)
- Loads F&O underlyings CSV file.
- Checks if file exists; returns empty set if not.
- Reads CSV, extracts unique symbols, strips whitespace.
- Returns set of F&O symbols.

#### load_variance_data_from_var_file(var_file_path)
- Loads variance data from a DAT file (treated as CSV).
- Checks if file exists; returns empty dataframe if not.
- Parses each line, looking for records with type '20'.
- Extracts symbol and variance value (field 4).
- Converts variance to float, skips invalid.
- Returns dataframe with symbol and variance, dropping duplicates.

#### categorize_securities_into_groups(dataframe)
- Validates required columns: average_traded_value, price_band_percentage, is_fno_stock, variance, series.
- Fills missing values: traded_value=0, price_band=0, variance=9999.
- Defines conditions for categories:
  - Valid series: EQ, BE, BZ.
  - 5x standard: valid series, traded > 50M, price band > 5%.
  - 3x standard: valid series, traded > 20M, price band > 5%.
  - 5x F&O: F&O stock, variance <= 20.
  - 3x F&O: F&O stock, variance <= 33.33, not 5x F&O.
- Assigns categories: "Only Delivery" default, "3x" or "5x" based on conditions.
- Returns dataframe with added "category" column.

### 3. Main Execution

#### Load Securities Data
- Checks if `security.txt` exists; raises error if not.
- Checks for `nse security headers.txt`:
  - If exists, parses headers list, converts to uppercase.
  - Reads securities file with headers.
- Else, reads without headers, warns about missing headers.
- Ensures symbol column exists; raises error if not.
- Strips whitespace from symbols.
- Prints number of securities loaded.

#### Merge with Bhav Data
- Calls `load_and_aggregate_bhav_data` to get aggregated traded values and series.
- Merges securities dataframe with bhav data on symbol (left join).
- Prints combined data shape.

#### Handle Series Column
- If series column missing, uses series_from_bhav if available.
- Fills missing series with series_from_bhav.
- Drops series_from_bhav column if present.

#### Handle Average Traded Value
- If missing, sets to 0.0, warns.

#### Calculate Price Band Percentage
- If price_range_column exists, applies `extract_price_band_percentage` to each row.
- Else, sets to 0.0, warns.

#### Add F&O Status
- Loads F&O symbols set.
- Adds "is_fno_stock" boolean column based on symbol presence in set.
- Prints updated shape.

#### Merge Variance Data
- Loads variance dataframe.
- Merges with combined data on symbol (left join).

#### Categorize Securities
- Calls `categorize_securities_into_groups` to assign categories.

#### Save Output
- Saves categorized dataframe to `securities_categorized.csv` without index.
- Prints success message.

#### Print Summaries
- Categorization Summary: Counts per category.
- Data Quality Report: Counts of missing values for traded value, price band, variance.
- Processing Assumptions: Lists 5 key assumptions about data handling.

## Outputs
- `securities_categorized.csv`: CSV file with all securities data plus category column.
- Console output:
  - Number of securities loaded.
  - Data shapes after merges.
  - Success message for file save.
  - Categorization counts.
  - Missing data counts.
  - Assumptions list.

## Assumptions and Data Handling
1. If a security has no Bhav Copy data in the last 3 trading days, its average traded value is treated as 0.
2. If price band range is missing or malformed in the securities file, price band percentage is treated as 0%.
3. If a security is not listed in the F&O underlyings file, it is treated as a non-F&O security (regular equity only).
4. If a security has no matching entry in the VAR file, it is not eligible for F&O-based 5x/3x overrides.
5. Categorization precedence: 5x > 3x > Only Delivery. A security is placed in the highest applicable category.

## Notes
- The script handles missing files gracefully with warnings.
- Variance file is expected to be in DAT format but read as CSV with comma separation.
- All monetary values are in rupees, traded values in lakhs or similar units as per NSE data.
