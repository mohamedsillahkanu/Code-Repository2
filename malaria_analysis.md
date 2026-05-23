## Step 1: Import Libraries

Import all necessary Python libraries for data processing, analysis, and visualization.

```python
import pandas as pd
from pathlib import Path
from tabulate import tabulate
```

## Step 2: Set Folder Path

Replace `"your_folder_path_here"` with the path where your Excel files are stored.

```python
file_path = "your_folder_path_here"
```

## Step 3: Find All Excel Files

Find all files ending with `.xlsx` in the folder.

```python
files = Path(file_path).glob("*.xlsx")
```

## Step 4: Read Each Excel File into a DataFrame

Loop through each Excel file and read into a pandas DataFrame.

```python
df_list = [pd.read_excel(file) for file in files]
```

## Step 5: Combine All DataFrames

Merge all DataFrames into a single DataFrame.

```python
combined_df = pd.concat(df_list, ignore_index=True)
```

## Step 6: Preview the Combined Data

Display the last 5 rows of the combined DataFrame.

```python
print("\n=== Preview of Combined Data ===")
print(tabulate(combined_df.tail(), headers='keys', tablefmt='grid'))
```

## Step 7: List Column Names in 3 Columns

Display all column names in a grid with 3 columns per row.

```python
columns = list(combined_df.columns)
padded_cols = columns + [""] * ((3 - len(columns) % 3) % 3)
col_table = [padded_cols[i:i+3] for i in range(0, len(padded_cols), 3)]
print("\n=== Column Names (3 per row) ===")
print(tabulate(col_table, headers=["Column 1", "Column 2", "Column 3"], tablefmt="grid"))
```

## Step 8: Sample Output Results

Upload your results.png image to display your analysis output.
```

**To save:**
1. Copy the text above
2. Open Notepad
3. Paste
4. Save As → `malaria-analysis.md` (choose "All Files" as file type)

This will work perfectly with your HTML page and display all 8 steps correctly!
