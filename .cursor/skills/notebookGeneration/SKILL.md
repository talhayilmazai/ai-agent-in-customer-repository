---
name: notebook-generator
description: Creates ready-to-run Databricks test notebooks for rocks_extension modules, including environment setup and metadata injection.
---

## 🎯 Goal
Generate a **ready-to-run Databricks Python notebook script** for testing the currently active PySpark class (Step/Job).
The goal is to allow the developer to manually test logic changes without running the full pipeline.

When used with a module name, class name, or config section, perform:
1. Locate the Python implementation in `rocks_extension/`
2. Identify the parent class from `rocks.*` framework
3. Find the config section in `rocks_extension/*/config.yml`
4. Find task definition in `dags/fabbrica.yaml` for runtime parameters
5. Identify metadata tables that may need path injection
6. Generate a complete test notebook with all required cells

## ⚠️ CRITICAL: Output Instructions

**Save the notebook as an `.ipynb` file in a designated location.**

### Output Path
```
notebooks/test_[module]_[class_name].ipynb
```

Or save to user's preferred location if specified.

## Sample Usage

- "Generate notebook for `BootsLookupTableGenerator`"
- "Create test notebook for `dotcom_smartlag_array_lookup` config"
- "Notebook for `BootsSmartlagArrayLookupForecast`"

Search for:

```bash
# Find the Python implementation
grep -rn "class CLASS_NAME" rocks_extension/

# Find config section
grep -B5 -A100 "^config_section_name:" rocks_extension/*/config.yml

# Find DAG task for runtime parameters
grep -B5 -A20 "class: CLASS_NAME" dags/fabbrica.yaml

# Find metadata definitions for injection
grep -B2 -A10 "table_name:" rocks_extension/*/config_metadata.yml
```

## Notebook Structure

The generated notebook follows this pattern derived from the development workflow:

### Cell 1: Environment Variables Setup
```python
from pyspark.sql import functions as F
import os

os.environ["CUSTOMER_NAME"] = <customer-name>
os.environ["ENABLE_ROCKS_CONFIG"] = "TRUE"
os.environ["CONFIG_PREFIX"] = "/dbfs/FileStore/src/prod/"  # Production config
# os.environ["CONFIG_PREFIX"] = "/dbfs/FileStore/tables/{username}/{module}/"  # Dev config
os.environ["DATASTORE_BUCKET_NAME"] = "invent-<customer-name>-datastore"
```
### Cell 2: Class Definition (if extending)
If the class needs modifications for testing, include the extension code:

```python
"""
{Class Name} Extension for Testing
"""
from pyspark.sql import functions as F, DataFrame

from rocks.{module}.{submodule} import {ParentClass}
# Additional imports as needed


class {ClassName}({ParentClass}):
    """
    Extension of {ParentClass} for testing purposes
    """
    
    # Override methods if needed for testing
    def preprocess(self, df: DataFrame) -> DataFrame:
        # Custom preprocessing logic
        return df
```
### Cell 3: Class Instantiation
```python
self = {ClassName}(
    run_date="{run_date}",
    config_section="{config_section}",
    # Additional runtime parameters from fabbrica.yaml
    # mode="fit_predict",
    # run_type="operation",
    # decomposition_enabled=False,
)
```
### Cell 4-N: Metadata Injection (if needed)
For each table that needs path override:

```python
# Check current metadata
self.data.metadata_provider.get_metadata(self.{table_name_property})

# Inject custom path for testing
self.data.metadata_provider.put_to_inner(
    self.{table_name_property}, 
    "key", 
    value="noob/temp/{path_pattern}"
)

# Verify injection
self.data.metadata_provider.get_metadata(self.{table_name_property})
```

### Cell N+1: Execute
```python
self.run()
```

### Cell N+2: Lineage Inspection (optional)
```python
import pprint

print("--- Pretty Printed Lineage Data ---")
pprint.pprint(self.lineage)
```

### Cell N+3: Output Verification (optional)
```python
# Read and inspect output
output_df = spark.read.load(
    "/mnt/invent-boots-wba-datastore/{output_path}/run_date={run_date}"
)
output_df.display()
```
## Runtime Parameters Reference

Extract from `dags/fabbrica.yaml` task definitions:

| Parameter | Source | Description |
|-----------|--------|-------------|
| `run_date` | DAG conf or manual | Execution date in YYYY-MM-DD format |
| `config_section` | fabbrica.yaml `application_args` | Config section name |
| `mode` | fabbrica.yaml `application_args` | `fit`, `predict`, `fit_predict` |
| `run_type` | fabbrica.yaml `application_args` | `operation`, `training`, custom |
| `update_range` | fabbrica.yaml `application_args` | Days to process |
| `model_id` | config.yml | Model identifier |
| `decomposition_enabled` | config.yml / override | Enable forecast decomposition |

---

## Metadata Injection Patterns

### When to Use Metadata Injection

Use `self.data.metadata_provider.put_to_inner()` when:
1. Testing with alternative data paths
2. Redirecting output to temp folders
3. Using different model versions
4. Testing with subset of data

### Common Injection Patterns

```python
# Input table path override
self.data.metadata_provider.put_to_inner(
    self.input_table_name, 
    "key", 
    value="noob/temp/dotcom/daily-data"
)

# Output table path override (to temp folder)
self.data.metadata_provider.put_to_inner(
    self.output_table_name, 
    "key", 
    value="noob/temp/lookup-tables-dotcom-family-v2/{key}"
)

# Seasonality table with model_id placeholder
self.data.metadata_provider.put_to_inner(
    self.forecast_coefficients_with_shares_table_name, 
    "key", 
    value="noob/temp/{seasonality_model_id}/seasonality-with-shares"
)

# Share table override
self.data.metadata_provider.put_to_inner(
    self.day_effect_by_week_table_name, 
    "key", 
    value="noob/temp/agg-shares-selected/day_of_week"
)

# Lookup table override
self.data.metadata_provider.put_to_inner(
    self.lookup_table_name, 
    "key", 
    value="noob/temp/lookup-tables-dotcom-family-v2/{key}"
)
```

### Path Variable Patterns

| Pattern | Description | Example |
|---------|-------------|---------|
| `{key}` | Lookup table key from config | `product_id_store_id` |
| `{run_type}` | Operation or training | `operation` |
| `{model_id}` | Model identifier | `dotcom-smartlag` |
| `{seasonality_model_id}` | Seasonality model ID | `dotcom-seasonality` |

---

## Config Section Discovery

### Finding Config Parameters

```bash
# Find config section in config.yml
grep -B5 -A100 "^config_section_name:" rocks_extension/*/config.yml

# Find table name references
grep "_table_name:" rocks_extension/*/config.yml | grep config_section
```

### Common Config Properties

| Property | Description |
|----------|-------------|
| `input_table_name` | Main input table |
| `output_table_name` | Primary output table |
| `lookup_table_name` | Lookup/reference table |
| `forecast_coefficients_table_name` | Seasonality coefficients |
| `forecast_coefficients_with_shares_table_name` | Seasonality with shares |
| `day_effect_by_week_table_name` | Day-of-week effect table |
| `promo_table_name` | Promotion data table |
| `model_id` | Model identifier for output paths |
| `run_type` | Execution type (operation/training) |

---

## Class Inheritance Reference

### Data Prep Classes

| Extension Class | Parent Class | Module |
|----------------|--------------|--------|
| `BootsLookupTableGenerator` | `rocks.noob.data_prep.LookupTableGenerator` | noob/data_prep |
| `BootsCustomScopeGenerator` | `rocks.noob.data_prep.ScopeGenerator` | noob/data_prep |
| `BootsSimpleLookupGenerator` | `rocks.noob.data_prep.SimpleLookupGenerator` | noob/data_prep |

### Example Forecast Classes

| Extension Class | Parent Class | Module |
|----------------|--------------|--------|
| `BootsSmartlagArrayLookupForecast` | `rocks.noob.forecast.smartlag.SmartlagArrayLookupForecast` | noob/forecast/smartlag |
| `BootsLongSmartlagArrayLookupForecast` | `rocks.noob.forecast.smartlag.SmartlagArrayLookupForecast` | noob/forecast/smartlag |
| `BootsRuleBasedSelection` | `rocks.noob.forecast.selection.RuleBasedSelection` | noob/forecast/selection |

### Example ETL Classes

| Extension Class | Parent Class | Module |
|----------------|--------------|--------|
| `BootsProductEtl` | `rocks.opal.pre_etl.PreETL` | opal/pre_etl |
| `BootsStoreEtl` | `rocks.opal.pre_etl.PreETL` | opal/pre_etl |

---

## Notes

1. **Run Date Format**: Always use `YYYY-MM-DD` format
2. **Config Prefix**: Use prod for production testing, dev path for development
3. **Run Type**: Use custom `run_type` (e.g., `dotcom-test-family`) to avoid overwriting production data
4. **Metadata Injection**: Always verify with `get_metadata()` before and after injection
5. **Lineage**: Check `self.lineage` after run to verify data flow
6. **Temp Folders**: Use `noob/temp/` prefix for all test outputs