# Data Dictionary: Raw Source Files (Bronze Layer)

This document contains the structural schema documentation for all five source files uploaded to our landing volume. It serves as the engineering baseline for building our Bronze DDL tables.

---

## 1. customers.csv
* **Folder Location:** `/Volumes/bluepeak/bronze/landing_volume/customers/`
* **Granularity:** One row represents a unique bank customer

| Column Name | Business Meaning | Example Value | Key Status |
| :--- | :--- | :--- | :--- |
| `customer_id` | Unique internal identifier for a customer | `CUST-001928` | **Primary Key** |
| `first_name` | Legal first name of the account holder | `Sarah` | None |
| `last_name` | Legal last name of the account holder | `Jenkins` | None |
| `email` | Primary email address for notifications | `s.jenkins@email.com` | None |
| `date_of_birth` | Date of birth used for identity verification | `1988-11-14` | None |

---

## 2. accounts.csv
* **Folder Location:** `/Volumes/bluepeak/bronze/landing_volume/accounts/`
* **Granularity:** One row per opened financial bank account.

| Column Name | Business Meaning | Example Value | Key Status |
| :--- | :--- | :--- | :--- |
| `account_id` | Unique identifier for a bank account | `ACC-77491` | **Primary Key** |
| `customer_id` | Link back to the account owner | `CUST-001928` | **Foreign Key** (Points to customers) |
| `branch_id` | Link to the branch where the account opened | `BR-04` | **Foreign Key** (Points to branches) |
| `account_type` | The financial product category | `Checking` | None |

---

## 3. branches.csv
* **Folder Location:** `/Volumes/bluepeak/bronze/landing_volume/branches/`
* **Granularity:** One row per physical retail bank branch location.

| Column Name | Business Meaning | Example Value | Key Status |
| :--- | :--- | :--- | :--- |
| `branch_id` | Unique identifier for a retail branch | `BR-04` | **Primary Key** |
| `branch_name` | Name or location description of the branch | `Downtown Calgary` | None |
| `country` | Country where the branch operates | `Canada` | None |

---

## 4. transactions_batch_01.csv
* **Folder Location:** `/Volumes/bluepeak/bronze/landing_volume/transactions/`
* **Granularity:** One row per financial ledger event (base load tracking).

| Column Name | Business Meaning | Example Value | Key Status |
| :--- | :--- | :--- | :--- |
| `transaction_id` | Unique identifier for a transaction record | `TXN-8839201` | **Primary Key** |
| `account_id` | Link to the target bank account | `ACC-77491` | **Foreign Key** (Points to accounts) |
| `transaction_date`| Date and time when the ledger event occurred | `2026-08-01` | None |
| `amount` | Financial value of the event (stored as text) | `$1,250.00` | None |
| `type` | Code direction of fund movement | `Credit` | None |

---

## 5. transactions_batch_02.csv
* **Folder Location:** `/Volumes/bluepeak/bronze/landing_volume/staging/`
* **Granularity:** One row per financial ledger event (isolated delta batch).

| Column Name | Business Meaning | Example Value | Key Status |
| :--- | :--- | :--- | :--- |
| `transaction_id` | Unique identifier for a transaction record | `TXN-8841005` | **Primary Key** |
| `account_id` | Link to the target bank account | `ACC-55210` | **Foreign Key** (Points to accounts) |
| `transaction_date`| Date and time when the ledger event occurred | `08/02/2026` | None |
| `amount` | Financial value of the event (stored as text) | `$45.25` | None |
| `type` | Code direction of fund movement | `Debit` | None |

---

## Entity Relationships and Join Mapping

This section outlines how the four core entities link together across the data pipeline. To maintain relational integrity during transformations, use these column mappings:

### 1. Customers to Accounts
* **Relationship:** **1-to-Many** (One customer can open multiple financial accounts).
* **Join Condition:** `customers.customer_id = accounts.customer_id`
* **Key Role:** `customer_id` is the Primary Key in the customers file and acts as a Foreign Key in the accounts file.

### 2. Branches to Accounts
* **Relationship:** **1-to-Many** (One physical bank branch manages many customer accounts).
* **Join Condition:** `branches.branch_id = accounts.branch_id`
* **Key Role:** `branch_id` is the Primary Key in the branches file and acts as a Foreign Key in the accounts file.

### 3. Accounts to Transactions
* **Relationship:** **1-to-Many** (One financial account accumulates multiple transaction entries over time).
* **Join Condition:** `accounts.account_id = transactions.account_id`
* **Key Role:** `account_id` is the Primary Key in the accounts file and acts as a Foreign Key in the transactions files (both Batch 1 and Batch 2).
