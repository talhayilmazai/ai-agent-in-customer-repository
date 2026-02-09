---
name: documentation-generator
description: Generates standardized technical documentation (Markdown) for Python modules, classes, or config sections, saving them to the docs directory.
---

## 🎯 Goal
Analyze the currently active/selected Python code (and its config context) to generate a standardized technical documentation file (`.md`).

## 🧠 Analysis Strategy
Do not search for files blindly. When invoked with a module name, class name, or config section, perform followings:
1. Locate the Python implementation (in `rocks_extension/` or `libs/rocks-*/`)
2. Extract all input/output tables from `self.data.try_read()` and `self.data.write()` calls
3. Find the DAG configuration in `dags/fabbrica.yaml`
4. Extract config parameters from `config.yml`
5. Identify metadata definitions from `config_metadata.yml`
6. Document the main logic and data flow
7. **Save the documentation as a markdown file**

### Output Instructions

**Do NOT paste documentation in chat. Save directly to file.**

#### Output Path
```
docs/rocks_extension/[module]/[script_name].md
```

##### Examples
| Class | Output File |
|-------|-------------|
| `BootsSimpleLookupGenerator` | `docs/rocks_extension/noob/lookup_table_generator.md` |
| `BootsSimulation` | `docs/rocks_extension/future_viz/rocket/simulation.md` |
| `ForecastErrorReport` | `docs/rocks_extension/reporting/forecast_error_report.md` |
| `RuleBasedSelection` | `docs/rocks_extension/noob/forecast/rule_based_selection.md` |

#### Steps
1. Create directory if needed: `mkdir -p docs/rocks_extension/[module]/`
2. Use `create_file` tool to save the markdown
3. Respond with: `✅ Documentation saved to docs/rocks_extension/[module]/[script_name].md`

## Sample Usage

- "Document `BootsSimpleLookupGenerator`"
- "Generate docs for `rule_based_selection` config"
- "Document the `smartlag_forecast` task"

Conduct these searches automatically:

```bash
# Find the Python implementation
grep -rn "class MODULE_NAME" rocks_extension/
grep -rn "class MODULE_NAME" libs/rocks-*/

# Find config section usage
grep -n "config_section:" dags/fabbrica.yaml | grep MODULE_NAME
grep -B5 -A50 "^MODULE_NAME:" rocks_extension/*/config.yml

# Find I/O operations
grep -n "self.data.try_read\|self.data.write" PATH_TO_MODULE.py

# Find metadata definitions
grep -B2 -A10 "table_name" rocks_extension/*/config_metadata.yml

# Find DAG task definition
grep -B5 -A30 "MODULE_NAME:" dags/fabbrica.yaml
```

## Documentation Template

Save this template (filled in with actual data) to the output file:

```markdown
# [ClassName] Documentation

## 📌 Overview
> **One-line description from docstring or ABOUTME comment.1-2 sentences explaining the business purpose of this step.**

---

## 📍 Location

| Property | Value |
|----------|-------|
| **Python Module** | `rocks_extension/[module]/[submodule]/[file].py` |
| **Class** | `ClassName` |
| **Parent Class** | `ParentClass` (from `rocks.module.submodule`) |
| **Config Section** | `config_section_name` |
| **DAG** | `dag_name` |
| **Task ID** | `task_id_in_dag` |

---

## ⏱️ DAG Schedule & Dependencies

```yaml
# From dags/fabbrica.yaml
dag_name:
  plugin: fabbrica-plugin-invent::plugin_name
  rocks_pipeline:
    tasks:
      task_name:
        module: rocks_extension.module.submodule
        class: ClassName
        upstream_task_ids:
          - upstream_task_1
          - upstream_task_2
        downstream_task_ids:
          - downstream_task_1
```

**Execution Flow:**
```
[upstream_task_1] ──┐
                    ├──► [THIS TASK] ──► [downstream_task_1]
[upstream_task_2] ──┘
```

---

## 📥 Inputs

| Table Name | Access Method | Config Key | Path | Required |
|------------|---------------|------------|------|----------|
| `table_1` | `try_read` | `self.input_table_name` | `${operation_home}/path` | ✅ |
| `table_2` | `try_read_latest` | `self.lookup_table_name` | `${noob_home}/lookup/historic` | ✅ |
| `table_3` | `try_read` | hardcoded | `${master_data_home}/products` | ❌ |

### Input Table Details

<details>
<summary><code>table_1</code> - Main input data</summary>

```yaml
# From config_metadata.yml
table_1:
  key: ${operation_home}/path
  primary_key:
    - product_id
    - store_id
  partition_columns:
    - date
  schema: ${schema_config.table_1.schema}
```

**Required Columns:** `product_id`, `store_id`, `date`, `value`

</details>

---

## 📤 Outputs

| Table Name | Write Method | Config Key | Path | Partitioned |
|------------|--------------|------------|------|-------------|
| `output_table` | `write_with_decision` | `self.output_table_name` | `${noob_home}/output/historic` | ✅ `date` |

### Output Table Details

<details>
<summary><code>output_table</code> - Processed results</summary>

```yaml
# From config_metadata.yml
output_table:
  key: ${noob_home}/output/historic
  primary_key:
    - product_id
    - store_id
    - date
  partition_columns:
    - date
  is_historic_table: true
  schema: ${schema_config.output_table.schema}
```

**Output Columns:** `product_id`, `store_id`, `date`, `processed_value`, `flag`

</details>

---

## ⚙️ Configuration Parameters

```yaml
# From rocks_extension/[module]/config.yml
config_section_name:
  # Core parameters
  input_table_name: table_1
  output_table_name: output_table
  lookup_table_name: table_2
  
  # Processing parameters
  param_1: value_1
  param_2: ${commons.shared_value}
  param_3:
    - list_item_1
    - list_item_2
  
  # Feature flags
  enable_feature_x: true
  debug_mode: false
```

### Parameter Reference

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `input_table_name` | `str` | - | Logical name of input table |
| `param_1` | `int` | `10` | Controls X behavior |
| `param_2` | `str` | `${commons.shared_value}` | Inherited from commons |
| `enable_feature_x` | `bool` | `true` | Enables feature X processing |

---

## 🔄 Main Logic

**Purpose:** [One paragraph describing what this module does]

### Algorithm

1. **Load Inputs**
   - Read `table_1` filtered by date range
   - Read `table_2` lookup table
   - Join on `[product_id, store_id]`

2. **Transform**
   - Apply preprocessing (e.g., filter nulls, cast types)
   - Calculate derived columns
   - Apply business rules from config

3. **Output**
   - Partition by `date`
   - Write to `output_table` using `write_with_decision`

### Key Methods

| Method | Description |
|--------|-------------|
| `run()` | Main entry point, orchestrates the pipeline |
| `preprocess(df)` | Data cleaning and type casting |
| `transform(df)` | Core business logic |
| `postprocess(df)` | Final formatting before write |

### Code Flow
```
run()
  │
  ├──► load_inputs()
  │      ├── try_read("table_1")
  │      └── try_read_latest("table_2")
  │
  ├──► preprocess(df)
  │      └── filter, cast, join lookup
  │
  ├──► transform(df)
  │      └── business logic, calculations
  │
  └──► write_output(df)
         └── write_with_decision("output_table")
```

---

## 🧪 Testing Notes

**Test File:** `tests/[module]_tests/test_[class_name].py`

**Key Test Cases:**
- [ ] Empty input handling
- [ ] Null value handling
- [ ] Partitioning correctness
- [ ] Schema validation

---

## 📝 Usage Examples

**Run via DAG:**
```bash
airflow dags trigger dag_name --conf '{"run_date": "2024-01-15"}'
```

**Run Standalone (for testing):**
```python
from rocks_extension.module.submodule import ClassName

step = ClassName(
    config_section="config_section_name",
    run_date="2024-01-15"
)
step.run()
```

---

## 🔗 Related Components

| Component | Relationship |
|-----------|--------------|
| `UpstreamClass` | Produces input `table_1` |
| `DownstreamClass` | Consumes output `output_table` |
| `rule_based_selection` | Uses this output for model selection |

---

## Quick Reference: Search Commands

```bash
# Find all methods in a class
grep -n "def " rocks_extension/[module]/[file].py

# Find all config.yml references in code
grep -rn "self.config\." rocks_extension/[module]/

# Find all table reads
grep -n "try_read\|try_read_latest" rocks_extension/[module]/[file].py

# Find all table writes
grep -n "\.write\|write_with_decision" rocks_extension/[module]/[file].py

# Find schema definition
grep -B2 -A20 "^  table_name:" rocks_extension/[module]/config_schema_tables.yml

# Find metadata definition  
grep -B2 -A15 "^    table_name:" rocks_extension/[module]/config_metadata.yml

# Find DAG task
grep -B3 -A20 "task_name:" dags/fabbrica.yaml

# Find upstream/downstream tasks
grep -A50 "task_name:" dags/fabbrica.yaml | grep -E "upstream_task_ids|downstream_task_ids" -A5
```

---

## Data Access Patterns Reference

| Method | Description | Example |
|--------|-------------|---------|
| `self.data.try_read(name)` | Read latest partition | `self.data.try_read("products")` |
| `self.data.try_read_latest(name)` | Read from `/latest` path | `self.data.try_read_latest("forecast_historic")` |
| `self.data.try_read(name, path_suffix=...)` | Read specific partition | `self.data.try_read("sales", path_suffix=f"date={run_date}")` |
| `self.data.daily_data(name, date=...)` | Read daily partitioned data | `self.data.daily_data("inventory", date=run_date)` |
| `self.data.write(df, name)` | Simple write | `self.data.write(df, "output")` |
| `self.data.write_with_decision(df, name, ...)` | Write with mode control | `self.data.write_with_decision(df, "output", mode="append")` |

---

## Config Access Patterns Reference

| Pattern | Description | Example |
|---------|-------------|---------|
| `self.config.param` | Direct config access | `self.config.forecast_horizon` |
| `self.input_table_name` | Table name from config | `self.data.try_read(self.input_table_name)` |
| `${commons.value}` | Reference to commons | `forecast_horizon: ${commons.forecast_horizon}` |
| `${oc.select:path, default}` | Optional with default | `${oc.select:schema_config.table.user_schema, null}` |
| `${oc.env:VAR, default}` | Environment variable | `${oc.env:DATASTORE_BUCKET_NAME}` |

```