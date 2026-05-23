## Step 1: Import Libraries

```python
import pandas as pd
from pathlib import Path
from tabulate import tabulate
```

**Explanation:**

`pandas` → for reading and manipulating Excel files.
`Path` from `pathlib` → makes it easy to find and manage file paths.
`tabulate` → prints tables nicely in the console for better readability.



## Step 2: Set Folder Path

```python
file_path = "your_folder_path_here"
```

**Explanation:**

Replace `"your_folder_path_here"` with the path where your Excel files are stored.
This folder will be searched for all `.xlsx` files to combine.



## **Step 3: Find All Excel Files**

```python
files = Path(file_path).glob("*.xlsx")
```

**Explanation:**

* `.glob("*.xlsx")` → finds all files ending with `.xlsx` in the folder.
* `files` will be an iterable of all matching Excel files.



## **Step 4: Read Each Excel File into a DataFrame**

```python
df_list = [pd.read_excel(file) for file in files]
```

**Explanation:**

* Loop through each Excel file in `files`.
* Read each one into a pandas DataFrame.
* Store all DataFrames in the list `df_list`.



## **Step 5: Combine All DataFrames**

```python
combined_df = pd.concat(df_list, ignore_index=True)
```

**Explanation:**

* `pd.concat()` → merges all DataFrames in `df_list` into a single DataFrame.
* `ignore_index=True` → resets the row indices in the combined DataFrame.

---

## **Step 6: Preview the Combined Data**

```python
print("\n=== Preview of Combined Data ===")
print(tabulate(combined_df.tail(), headers='keys', tablefmt='grid'))
```

**Explanation:**

* `combined_df.tail()` → shows the last 5 rows of the combined DataFrame.
* `tabulate(..., tablefmt='grid')` → formats the output into a clean, readable grid.
* Helps you quickly verify that the data has been combined correctly.



## **Step 7: List Column Names in 3 Columns**

```python
columns = list(combined_df.columns)
padded_cols = columns + [""] * ((3 - len(columns) % 3) % 3)
col_table = [padded_cols[i:i+3] for i in range(0, len(padded_cols), 3)]
print("\n=== Column Names (3 per row) ===")
print(tabulate(col_table, headers=["Column 1", "Column 2", "Column 3"], tablefmt="grid"))
```

**Explanation:**

1. `columns = list(combined_df.columns)` → get all column names.
2. `padded_cols = ...` → add empty strings if necessary to make the total number of columns divisible by 3 (for neat display).
3. `col_table = [padded_cols[i:i+3] ...]` → split the list into groups of 3 for each row.
4. `print(tabulate(...))` → display column names in a grid, 3 per row.



