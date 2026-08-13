# Data Defect Register

This register logs data quality anomalies found during the initial CSV profiling phase that would break basic calculations like `AVG(amount)` if left uncleaned.

| Source File | Defect Category | Found Field | Impact Description & Risk |
| :--- | :--- | :--- | :--- |
| `transactions_batch_02.csv` | **Numbers as Text** | `amount` | Financial numbers are wrapped in quotes or include symbol characters (e.g., `$1,200.50`), forcing Spark to interpret the column as a text String. A naïve `AVG(amount)` will fail or evaluate to 0. |
| `transactions_batch_02.csv` | **Missing / Null Data** | `amount` | Several transactions contain blank or empty string values in the amount column, which will skew balances or drop records during aggregation. |
| `transactions_batch_02.csv` | **Multiple Date Formats** | `transaction_date` | Dates use mixed configurations (e.g., `YYYY-MM-DD` on some rows and `MM/DD/YYYY` on others), which will break standard timestamp parsing engines. |
| `accounts.csv` | **Duplicate Business Keys** | `account_id` | Duplicate rows share identical account identifiers, creating a risk of artificially doubling account balances if not deduplicated in Silver. |
| `accounts.csv` | **Orphaned Rows** | `customer_id` | Accounts exist with `customer_id` values that do not correspond to any valid parent entry in the `Customers` master list. |
| `customers.csv` | **Inconsistent Casing** | `country` / `state` | Text fields contain mixed string styles (e.g., `USA`, `usa`, `United States`), which will cause downstream groupings and filters to split data inaccurately. |
