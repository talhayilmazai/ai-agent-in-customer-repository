---
applyTo: "**/*.py"
---

# PySpark Development Guidelines for the Rocks Ecosystem

This document provides detailed instructions and best practices for working with PySpark within the Rocks Ecosystem. Adhering to these guidelines will ensure that our data processing is efficient, scalable, and maintainable.

## 1. Core Principles

-   **Immutability**: DataFrames are immutable. Every transformation creates a new DataFrame. Embrace this by chaining transformations rather than trying to modify a DataFrame in place.
-   **Lazy Evaluation**: Transformations in PySpark are lazy. They are only executed when an *action* (like `count()`, `collect()`, `show()`, or writing to a file) is called. Understand this to avoid unexpected performance bottlenecks.
-   **Standard Functions First**: Always prefer using the built-in functions from `pyspark.sql.functions` over writing User-Defined Functions (UDFs). Standard functions run directly on the JVM within Spark and are significantly faster.

## 2. Import Conventions

### Standard Import Pattern

Follow the established import pattern used throughout the scripts under rocks_extension:

```python
from pyspark.sql import DataFrame, Window, functions as F
from pyspark.sql import types as T  # when working with data types
```

### Data Types Import

When working with schemas and UDFs, import types module:

```python
from pyspark.sql import types as T

# Use for UDF return types, schema definitions, etc.
generate_family_id_udf = F.udf(generate_family_id, T.StringType())
```

## 3. DataFrame Transformations

This is the most common type of operation you will perform.

### Selecting and Renaming Columns

Use `select()` for selecting columns and `withColumnRenamed()` for renaming. For creating new columns, use `withColumn()`.

```python
from pyspark.sql import functions as F

# Select specific columns using settings
df_selected = input_df.select("product_id", "store_id", "sales_quantity")

# Create a new column based on existing ones
df_with_revenue = df_selected.withColumn(
    "revenue",
    F.col("sales_quantity") * F.col("price")
)

# Rename a column
df_renamed = df_with_revenue.withColumnRenamed("sales_quantity", "units_sold")
```

### Filtering Data

Use the `filter()` method with column conditions.

```python
# Filter for a specific store using settings
store_1_df = input_df.filter(F.col(S.store_col) == 1)

# Complex filtering with multiple conditions
filtered_df = input_df.filter(
    (F.col("is_promo") == True) & (F.col("sales_quantity") > 10)
)

# Null value filtering (common pattern in rocks-extension)
clean_df = input_df.filter(F.col("price").isNotNull())
```

### Date Operations

Date operations are common in the rocks-extension library:

```python
# Date arithmetic
df_with_lag_date = df.withColumn(
    "previous_date",
    F.date_add(F.col("date"), -1)
)

# Date filtering for time series operations
recent_df = df.filter(F.col("date") >= F.lit("2023-01-01"))
```

## 4. Joins

When joining DataFrames, always be explicit about the join type and the join keys.

### Broadcast Joins

If one of the DataFrames is small (e.g., < 100MB), use a broadcast join to avoid a costly shuffle. This is a common pattern in rocks-extension for joining with calendar, lookup tables, and aggregation levels.

```python
from pyspark.sql import functions as F

# Common pattern: broadcasting small lookup tables
joined_df = sales_df.join(
    F.broadcast(calendar_df),
    on=["date"],
    how="left"
)

# Broadcast aggregation level mappings
product_enriched = daily_data.join(
    F.broadcast(product_agg_level),
    on=[S.product_col],
    how="left"
)

# Multiple column joins with broadcast
enriched_df = base_df.join(
    F.broadcast(weekly_calendar),
    on=["week_index"],
    how="left"
)
```

## 5. Window Functions

Window functions are powerful for performing calculations over a group (or "window") of rows. They are essential for tasks like calculating running totals, moving averages, or ranking.

### Common Window Patterns in Rocks-Extension

```python
from pyspark.sql import Window

# Standard partitioning pattern for product-store analysis
window_spec = (
    Window
    .partitionBy("product_id", "store_id")
    .orderBy("date")
)

# Get the sales from the previous day for each product/store
df_with_lag = input_df.withColumn(
    "previous_day_sales",
    F.lag("sales_quantity", 1).over(window_spec)
)

# Lead function for range formatting (common in data_access.py)
range_window = Window.partitionBy(groupby_cols).orderBy(order_cols)
df_range = df.withColumn(
    "end_date",
    F.date_add(F.lead("date").over(range_window), -1)
)

# Aggregation level pattern
agg_window = Window.partitionBy("season_id", S.product_col, "warehouse_id")
df_with_totals = df.withColumn(
    "total_quantity",
    F.sum("quantity").over(agg_window)
)
```

### Share Calculation Windows

For share and normalization calculations (common in share_calculation modules):

```python
# Normalization window for share calculations
normalize_window = Window.partitionBy(*normalize_by, "period_index")
df_normalized = df.withColumn(
    "share",
    F.col("value") / F.sum("value").over(normalize_window)
)
```

## 6. User-Defined Functions (UDFs)

**Use UDFs as a last resort.** They are rarely used in the rocks-extension codebase. When absolutely necessary:

```python
from pyspark.sql import types as T

# Define the Python function
def generate_family_id(family_members):
    """Generate a unique family ID from members list"""
    if not family_members or len(family_members) == 0:
        return None
    return f"family_{hash('_'.join(sorted(family_members)))}"

# Register the UDF with proper return type
generate_family_id_udf = F.udf(generate_family_id, T.StringType())

# Apply the UDF
df_with_family = df.withColumn(
    "family_id",
    generate_family_id_udf(F.col("family_members"))
)
```

## 7. Performance Best Practices

### Avoid Shuffles

Shuffles are the most expensive operation in Spark. Operations that cause shuffles include:
- `join()` (unless broadcasted)
- `groupBy()`
- `orderBy()` (on a different key than the partition key)
- `repartition()`

### Partitioning Strategy

Data should be partitioned by the keys you most frequently join or group by:

```python
# Common partitioning patterns in rocks-extension

# Partition by date for time series analysis
df.write.partitionBy("date").parquet(output_path)

# Multi-level partitioning for hierarchical data
df.write.partitionBy("year", "month").parquet(output_path)
```

### Select Early, Filter Early

Reduce the size of your data as early as possible in your pipeline:

```python
# Good: Select and filter early
processed_df = (
    input_df
    .select("product_id", "store_id", "date", "sales_quantity", "price")
    .filter(F.col("date") >= start_date)
    .filter(F.col("sales_quantity") > 0)
    .withColumn("revenue", F.col("sales_quantity") * F.col("price"))
)
```

### Caching Strategy

Cache DataFrames only when they will be reused multiple times:

```python
# Cache when DataFrame will be used multiple times
base_df = input_df.filter(common_conditions).cache()

# Use the cached DataFrame multiple times
result_a = base_df.groupBy(S.product_col).agg(F.sum("sales"))
result_b = base_df.groupBy(S.store_col).agg(F.avg("price"))

# Don't forget to unpersist when done
base_df.unpersist()
```

## 8. Handling Null Values

Explicitly handle null values using `isNull()`, `isNotNull()`, or the `fillna()` method:

```python
# Filter out rows with null prices
filtered_df = input_df.filter(F.col("price").isNotNull())

# Fill null values in the 'inventory' column with 0
filled_df = input_df.fillna(0, subset=["inventory"])

# Use coalesce for handling nulls in calculations
df_with_defaults = df.withColumn(
    "effective_price",
    F.coalesce(F.col("sale_price"), F.col("regular_price"), F.lit(0))
)

# Handle nulls in date operations (common pattern)
df_with_end_date = df.withColumn(
    "end_date",
    F.coalesce(F.col("actual_end_date"), F.lit("2999-12-31"))
)
```

## 9. Aggregations and GroupBy Operations

### Standard Aggregation Patterns

```python
# Product-store level aggregations
product_store_agg = df.groupBy(S.product_col, S.store_col).agg(
    F.sum("sales_quantity").alias("total_sales"),
    F.avg("price").alias("avg_price"),
    F.count("*").alias("transaction_count")
)

# Time-based aggregations
daily_summary = df.groupBy("date").agg(
    F.sum("sales_quantity").alias("daily_sales"),
    F.countDistinct(S.product_col).alias("product_count"),
    F.countDistinct(S.store_col).alias("store_count")
)
```

### Complex Aggregations with Conditions

```python
# Conditional aggregations
promo_analysis = df.groupBy(S.product_col).agg(
    F.sum(F.when(F.col("is_promo"), F.col("sales_quantity"))).alias("promo_sales"),
    F.sum(F.when(~F.col("is_promo"), F.col("sales_quantity"))).alias("regular_sales"),
    F.avg(F.when(F.col("is_promo"), F.col("price"))).alias("avg_promo_price")
)
```

## 10. DataFrame Column Operations

### Column Transformations

```python
# Case/when operations for categorical encoding
df_categorized = df.withColumn(
    "sales_category",
    F.when(F.col("sales_quantity") > 100, "High")
    .when(F.col("sales_quantity") > 10, "Medium")
    .otherwise("Low")
)

# Mathematical operations
df_metrics = df.withColumn(
    "sales_per_day",
    F.col("total_sales") / F.col("days_in_period")
).withColumn(
    "price_change_pct",
    (F.col("current_price") - F.col("previous_price")) / F.col("previous_price") * 100
)
```

## 11. Data Type Conventions

### Type Annotations

Use proper type annotations for DataFrame parameters and return values:

```python
from typing import Optional, Union
from pyspark.sql import DataFrame

# Standard type patterns in rocks-extension
def process_data(self, input_df: DataFrame) -> DataFrame:
    """Process input data and return transformed DataFrame"""
    return input_df.withColumn("processed", F.lit(True))

# Use Union types when supporting both DataFrame types
def flexible_process(
    self, 
    df: DataFrame
) -> DataFrame:
    """Process data supporting both DataFrame types"""
    return df.filter(F.col("active") == True)

# Optional DataFrame for methods that might not return data
def try_get_data(self) -> Optional[DataFrame]:
    """Attempt to read data, return None if not available"""
    try:
        return self.data.try_read("optional_table")
    except Exception:
        return None
```

## 12. Error Handling Philosophy

**Do NOT add defensive checks.** Let code fail loudly when data or columns are missing. This makes misconfiguration obvious rather than hiding it.

### What NOT to Do

```python
# BAD: Defensive column checking
if "event_multiplier" in df.columns:
    df = df.withColumn("result", F.col("x") * F.col("event_multiplier"))

# BAD: Wrapping reads in try/except
try:
    df = self.data.try_read("some_table")
except Exception:
    df = None

# BAD: Checking if DataFrame is empty before processing
if df is not None and df.count() > 0:
    # process
```

### What TO Do

```python
# GOOD: Just use the column - fails loudly if missing
df = df.withColumn("result", F.col("x") * F.col("event_multiplier"))

# GOOD: Use required=True to fail explicitly on missing tables
df = self.data.try_read("some_table", required=True)

# GOOD: Trust that data exists and process it
result = df.groupBy("product_id").agg(F.sum("sales"))
```

Schema enforcement should happen at the configuration level (`config_schema_tables.yml`), not in application code.

## 13. Best Practices Summary

### Do's
- Use `product_id, store_id` for consistent column references
- Prefer `F.broadcast()` for small table joins
- Use `Window` functions for complex aggregations over partitions
- Chain transformations for readability
- Handle null values explicitly
- Use proper type annotations
- Cache DataFrames only when reused multiple times

### Don'ts
- Don't use UDFs unless absolutely necessary
- Don't collect large DataFrames to driver
- Don't use hardcoded column names when settings are available
- Don't cache DataFrames that are used only once
- Don't create unnecessary shuffles with poorly designed joins
- Don't add defensive checks like `if column in df.columns` - just use the column
- Don't wrap data reads in try/except - use `required=True` parameter instead

### Performance Tips
- Select and filter early in the transformation chain
- Use broadcast joins for small dimension tables
- Partition data by frequently used join/group keys
- Understand lazy evaluation to optimize action placement
- Monitor Spark UI to identify performance bottlenecks

## 14. Extension Pattern
When extending base classes from rocks libraries:
- Read the base class completely first
- Follow patterns used in the base class
- If you see bad code in the base class, fix it in your extension
- Check the correct version: e.g., `rocks-reporting==2.0.1` in `requirements-spark.txt`

Base class repositories:
- reporting: https://github.com/inventanalytics/rocks-reporting
- rocks-noob: https://github.com/inventanalytics/rocks-noob
- rocket: https://github.com/inventanalytics/rocks-rocket
- rocket_pandas: https://github.com/inventanalytics/rocket_pandas
- rocks-core: https://github.com/inventanalytics/rocks
- omega-ui: https://github.com/inventanalytics/omega-ui
- rocks-opal: https://github.com/inventanalytics/rocks-opal
- future-visibility: https://github.com/inventanalytics/rocks-future-visibility
- enigma: https://github.com/inventanalytics/optimization-enigma
- gensight: https://github.com/inventanalytics/gensight

#### Accessing Base Class Source Code

To understand base class implementations:

1. **Find the version** - Check `requirements-spark.txt` for the exact package version
2. **Access via GitHub API** - Use `gh api` to fetch files from the repository:
   ```bash
   # Get directory structure
   gh api repos/inventanalytics/{repo-name}/contents/{path} --jq '.[].name'

   # Get file content (base64 encoded)
   gh api repos/inventanalytics/{repo-name}/contents/src/rocks/noob/seasonality/data_prep.py --jq '.content' | base64 -d
   ```

3. **Note on directory structure** - Most base repositories use `src/rocks/` path prefix for source code:
   - ✓ `src/rocks/noob/seasonality/data_prep.py`
   - ✗ `rocks/noob/seasonality/data_prep.py`

**Example: SeasonalityDataPrep Analysis**

When analyzing `BootsSeasonalityDataPrep` in [rocks_extension/noob/seasonality/data_prep.py](rocks_extension/noob/seasonality/data_prep.py):

- Base class core workflow: `run()` → `holidays_prep()` → `sales_prep()`
- Base `sales_prep()` handles: data filtering, quality checks (stock rate, total sales, gaps), aggregation
- Base `holidays_prep()` handles: holiday calendar creation, promo integration

Boots extension adds performance optimizations:
- **`preprocess()`** - Pre-filters products (210+ day history), creates smaller aggregation tables
- **`manipulate_metadata()`** - Removes problematic columns from promo schema upfront
- **Caching strategy** - Pre-joins and caches expensive daily_data × promo joins to avoid repetition during iterative promo aggregation

This trades upfront compute for faster downstream processing.