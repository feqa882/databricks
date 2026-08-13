# Data Dictionary & Structural Relationships

This document outlines the source data models and relationships established before ingestion into the Bronze Medallion layer.

## 1. Core Source Entities
*   **Customers:** 
    *   *Granularity:* One row represents a unique bank customer account holder.
    *   *Business Key:* `customer_id`
*   **Accounts:** 
    *   *Granularity:* One row represents an opened financial account (Checking, Savings, etc.).
    *   *Business Key:* `account_id`
*   **Branches:** 
    *   *Granularity:* One row represents a physical retail bank branch location.
    *   *Business Key:* `branch_id`
*   **Transactions:** 
    *   *Granularity:* One row represents a distinct debit, credit, or cash movement event.
    *   *Business Key:* `transaction_id`

## 2. Relational Entity Mapping (Entity-Relationship Hierarchy)
*   `Customers` relates to `Accounts` as a **1-to-Many** relationship (One customer can hold multiple bank accounts via `customer_id`).
*   `Branches` relates to `Accounts` as a **1-to-Many** relationship (One physical branch manages multiple accounts via `branch_id`).
*   `Accounts` relates to `Transactions` as a **1-to-Many** relationship (One account accumulates many transaction line items via `account_id`).
