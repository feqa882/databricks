# Data Defect Register

This register documents every data quality issue identified during the raw file profiling phase (`03_profile_raw`). It establishes the technical cleaning requirements for the Silver layer pipelines in Sprint 2.

---

## 1. Summary of Data Defect Categories

| File Name | Impacted Column | Defect Category | Description | Example Value | Affected Rows | Proposed Fix for Sprint 2 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `transactions_batch_01.csv` & `transactions_batch_02.csv` | `amount` | **Numbers that will not cast cleanly** | Currency symbols and formatting commas force the column to read as text. Native casting to numeric directly yields Nulls. | `$1,250.00` | All rows | Use regex string manipulation `regexp_replace(amount, '[\\$,]', '')` to strip formatting before casting to `DOUBLE`. |
| `transactions_batch_02.csv` | `transaction_date` | **Multiple date/timestamp formats** | Staging batch data uses a completely different date structure than the historical baseline file. | `08/02/2026` vs `2026-08-01` | All Batch 2 rows | Apply a conditional date parser using `coalesce(to_date(col, 'yyyy-MM-dd'), to_date(col, 'MM/dd/yyyy'))`. |
| `transactions_batch_01.csv` | `amount` | **Missing / Null values** | Blanks or empty strings exist in critical monetary columns, which skews operational averages. | `""` or `NULL` | 14 rows | Filter out records where the amount is null or unparseable, routing defects to an error quarantine table. |
| `customers.csv` | `country` | **Inconsistent categorical values** | String values represent identical geographical entities using varying casing and formats. | `usa`, `USA`, `United States` | 42 rows | Standardize string data types using `upper(trim(country))` and apply an explicit case mapping lookup matrix. |
| `customers.csv` | `date_of_birth` | **Implausible values** | Dates of birth are logically impossible, such as dates set far into the future or indicating ancient ages. | `2099-12-31` or `1825-04-12` | 8 rows | Flag and quarantine rows where `date_of_birth > current_date()` or `date_of_birth < '1900-01-01'`. |
| `accounts.csv` | `customer_id` | **Orphaned foreign keys** | Account records point to a parent identifier that does not exist inside the customer master list. | `CUST-999999` (Missing in master) | 23 rows | Execute a left-anti join against the validated customer dimension table to quarantine orphaned account rows. |
| `accounts.csv` | `account_id` | **Duplicate business keys** | Multiple records share an identical account identifier, creating a data fan-out risk. | `ACC-77491` | 4 rows | Apply window function tracking to retain only the most recently updated or non-null record per account key. |

---

## 2. Precision Metrics: Duplicate Customer Identification

The profiling run revealed duplicate records within `customers.csv` on the business primary key (`customer_id`). To implement accurate deduplication logic in Sprint 2, we track three distinct, non-interchangeable metrics:

### Metric A: Duplicated Customer ID Values
* **Count:** **15 unique IDs**
* **Definition:** The specific count of unique `customer_id` strings that appear more than once in the dataset. 
* **Pipeline Action:** Represents the exact number of distinct customers impacted by duplicate keys.

### Metric B: Rows Involved in Duplicates
* **Count:** **32 total rows**
* **Definition:** The sum total of all physical rows in the file that contain those 15 duplicated keys. This includes both the original records and their matching duplicate entries.
* **Pipeline Action:** Identifies the total mass of messy rows that must be processed through a window function logic block.

### Metric C: Excess Rows (Rows to Drop)
* **Count:** **17 excess rows**
* **Definition:** Calculated as `Rows Involved (32) - Duplicated IDs (15)`. This is the exact number of redundant rows that will be completely discarded by the deduplication step.
* **Pipeline Action:** In Sprint 2, we will apply `row_number() OVER (PARTITION BY customer_id ORDER BY date_of_birth DESC)` and filter where `row_num = 1`. This safely drops exactly 17 rows, ensuring a single, accurate source of truth survives in the Silver tier.
---

## 3. Advanced Integrity Metrics: Transactions and Referential Alignment

### Activity 3: Orphan Transactions (Referential Integrity Scan)
We quantified how many `account_id` references in the raw transaction files (`transactions_batch_01.csv` and `transactions_batch_02.csv`) fail to map back to a valid parent record inside `accounts.csv`.

* **Distinct Missing Account IDs:** **8 unique accounts**
* **Total Impacted Rows:** **42 transaction rows**
* **Definition:** Transactions that are "orphaned" because they record balance movements against account structures that do not exist in our master dimension list.
* **Sprint 2 Proposed Fix:** Implement a left-semi join against `accounts.csv` during Silver processing to isolate valid transactions. Mismatched transactions will be diverted to an operational quarantine table in the `ops` schema for review rather than being dropped or processed into the main ledger.

### Activity 4: Transaction Batch Overlap Analysis
We quantified the physical key collisions between `transactions_batch_01.csv` and `transactions_batch_02.csv` by tracking identical business keys (`transaction_id`) inside the raw Bronze layer.

* **Overlapping Transaction IDs:** **15 duplicate transaction IDs**
* **Row Variance Assessment:** **Mismatched (Differing Data Fields)**
* **Detailed Finding:** A cross-file join confirmed that there are exactly 15 transaction IDs that exist in both files. An evaluation of their attributes reveals that **the overlapping rows are NOT identical**. While they share the exact same `transaction_id`, the underlying columns—such as `amount`, `txn_timestamp`, and `account_id`—contain completely different values. This indicates a severe upstream system issue (e.g., key recycling or sequence counter resets).
* **Sprint 2 Proposed Fix:** Because the data fields differ completely, a simple `MERGE INTO` table update statement using only `ON target.transaction_id = source.transaction_id` will destroy data by silently overwriting historical records. In Sprint 2, the cleaning pipeline must implement a compound business key (e.g., merging on both `transaction_id` AND `txn_timestamp`) or log these 15 collisions into a specialized operational quarantine table so that the historical ledger remains uncorrupted.
