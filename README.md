import pandas as pd

# Get file paths from user
source_file = input("Enter source file path: ").strip()
target_file = input("Enter target file path: ").strip()
mapping_file = input("Enter mapping file path: ").strip()

# Load source and target files
source_df = pd.read_csv(source_file)
target_df = pd.read_csv(target_file)

# Get fixed column positions (convert to zero-based index)
fixed_cols_source = list(map(int, input("Enter fixed column positions for source (comma-separated, 1-based): ").split(',')))
fixed_cols_target = list(map(int, input("Enter fixed column positions for target (comma-separated, 1-based): ").split(',')))

# Get mapping column position (convert to zero-based index)
source_mapping_idx = int(input("Enter mapping column position in source (1-based index): ")) - 1
target_mapping_idx = int(input("Enter mapping column position in target (1-based index): ")) - 1

# Identify value column (last column in both source and target)
source_value_col = source_df.columns[-1]
target_value_col = target_df.columns[-1]

# Read mapping file
mapping_dict = {}
with open(mapping_file, "r") as f:
    for line in f:
        source_val, target_val = line.strip().split()
        mapping_dict[source_val.lower()] = target_val.lower()  # Case-insensitive mapping

# Extract column names based on positions
source_fixed_cols = [source_df.columns[i-1] for i in fixed_cols_source]
target_fixed_cols = [target_df.columns[i-1] for i in fixed_cols_target]
source_mapping_col = source_df.columns[source_mapping_idx]
target_mapping_col = target_df.columns[target_mapping_idx]

# Apply mapping to source
source_df[target_mapping_col] = source_df[source_mapping_col].str.lower().map(mapping_dict).fillna(source_df[source_mapping_col])

# Select only relevant columns
source_key_columns = source_fixed_cols + [target_mapping_col]
target_key_columns = target_fixed_cols + [target_mapping_col]

# Aggregate values by summing duplicates
source_agg = source_df.groupby(source_key_columns)[source_value_col].sum().reset_index()
target_agg = target_df.groupby(target_key_columns)[target_value_col].sum().reset_index()

# Merge source and target for comparison
comparison_df = pd.merge(target_agg, source_agg, on=target_key_columns, how="left", suffixes=("_target", "_source"))

# Determine correctness of values
comparison_df["match"] = (comparison_df[f"{target_value_col}_target"] == comparison_df[f"{source_value_col}_source"])
comparison_df["color"] = comparison_df["match"].map({True: "green", False: "red"})

# Prepare tooltip data (source mapping details)
comparison_df["tooltip"] = "Mapped from: " + comparison_df[target_mapping_col] + " (Sum: " + comparison_df[f"{source_value_col}_source"].astype(str) + ")"

# Save output to CSV
output_file = "comparison_output.csv"
comparison_df.to_csv(output_file, index=False)

print(f"Comparison results saved to {output_file}")

