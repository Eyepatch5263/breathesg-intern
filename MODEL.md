# Data Model Specification (`MODEL.md`)

This document outlines the database schema and normalization design choices implemented in the Breathe ESG prototype backend. The data model is constructed to support multi-tenancy, multi-site grouping, unit normalization, source-of-truth mapping, and robust audit trails.

---

## 1. Entity Relationship Schema

The database relies on an SQL schema structure implemented in Django's ORM:

    Organization ||--o{ Facility : "has sites"}
    Organization ||--o{ DataImport : "imports files"}
    Organization ||--o{ NormalizedEmissionRecord : "owns ledger"}
    
    DataImport ||--o{ RawRecord : "contains raw rows"}
    
    Facility ||--o{ NormalizedEmissionRecord : "associated facility (optional)"}
    RawRecord ||--o{ NormalizedEmissionRecord : "normalizes to (1-to-N)"}
    
    NormalizedEmissionRecord ||--o{ AuditLog : "audit timeline"}


---

## 2. Model Definitions & Multi-Tenancy

### `Organization` (Tenant)
Ensures full data isolation. In a multi-tenant enterprise app, all data access queries filter by `Organization` (via security middleware or request query-routing).
* `id` (BigAutoField, Primary Key)
* `name` (CharField)
* `created_at` (DateTimeField)

### `Facility` (Site location)
Represents physical sites, offices, or warehouses.
* `id` (BigAutoField, Primary Key)
* `organization` (ForeignKey -> `Organization`): Scopes site lists to tenants.
* `name` (CharField): E.g., "Munich HQ", "Boston Office".
* `code` (CharField): Plant Code inside SAP (e.g. `1000`, `2000`) or unique grid IDs. Unique per organization.
* `country` (CharField): Holds standard ISO country codes (e.g., `US`, `DE`, `FR`) used to resolve grid and hotel emission factor variables.

### `DataImport` (Ingestion Job)
Tracks the trace of data ingestion batches.
* `id` (BigAutoField, Primary Key)
* `organization` (ForeignKey -> `Organization`)
* `source_type` (CharField): `SAP`, `UTILITY`, or `TRAVEL`.
* `filename` (CharField): Original uploaded file name.
* `status` (CharField): `SUCCESS`, `PARTIAL_FAILURE`, or `FAILED`.
* `imported_by` (ForeignKey -> Auth `User`): Links to the analyst who uploaded the file.
* `created_at` (DateTimeField)

### `RawRecord` (Source of Truth)
Maintains raw, unmodified payloads exactly as they arrived from SAP CSV, Concur API JSON, or portal files.
* `id` (BigAutoField, Primary Key)
* `data_import` (ForeignKey -> `DataImport`)
* `raw_data` (JSONField): Stores the raw CSV row dictionary or JSON booking segment.
* `status` (CharField): `PENDING`, `VALIDATED`, or `FAILED`.
* `validation_errors` (JSONField): Holds array of parsing error messages.

### `NormalizedEmissionRecord` (The Core Ledger)
Represents a normalized activity entry mapped to its carbon footprint output. One `RawRecord` may map to multiple prorated monthly records.
* `id` (BigAutoField, Primary Key)
* `organization` (ForeignKey -> `Organization`)
* `facility` (ForeignKey -> `Facility`, Nullable): Linked for Scope 1 & 2; Nullable for corporate travel.
* `raw_record` (ForeignKey -> `RawRecord`): **Source-of-truth trackback link.**
* `scope` (IntegerField): `1` (Direct), `2` (Indirect electricity), or `3` (Travel).
* `category` (CharField): E.g., `stationary_combustion`, `purchased_electricity`, `flight`, `hotel`, `car`.
* `activity_date` (DateField): The calendar month/day when the footprint occurred.
* `original_quantity` (DecimalField) / `original_unit` (CharField)
* `normalized_quantity` (DecimalField) / `normalized_unit` (CharField): Unified tracking unit (e.g. Liters, kWh, km).
* `co2e_kg` (DecimalField): Calculated output footprint.
* `emission_factor_used` (JSONField): Documents the calculation metadata (factor value, source name, and proration notes) at the time of ingestion to insulate calculations from future factor mutations.
* `approval_status` (CharField): `PENDING`, `APPROVED`, `REJECTED`, or `SUSPICIOUS` (outlier flag).
* `approved_by` (ForeignKey -> Auth `User`, Nullable)
* `approved_at` (DateTimeField, Nullable)
* `is_locked` (BooleanField): Once `is_locked = True`, records are frozen and immutable, ready for external audit.

### `AuditLog` (Audit Trail)
Tracks manual adjustments, lifecycle status transitions, and user logs.
* `id` (BigAutoField, Primary Key)
* `normalized_record` (ForeignKey -> `NormalizedEmissionRecord`)
* `changed_by` (ForeignKey -> Auth `User`, Nullable): Links to the action taker (system or analyst).
* `field_name` (CharField): Name of field changed (e.g., `co2e_kg`, `status`).
* `old_value` (TextField)
* `new_value` (TextField)
* `action_type` (CharField): `IMPORT`, `EDIT`, `APPROVE`, or `REJECT`.
* `timestamp` (DateTimeField)

---

## 3. Normalization Rules

* **SAP Procurement (Scope 1)**: Base units are liters for liquid fuels (Diesel, Heating Oil) and cubic meters ($m^3$) for gases (Natural Gas). Density multipliers (e.g., Diesel: $0.84\text{ kg/L}$) are applied to convert mass units (KG/Tons) to volumes.
* **Utility (Scope 2)**: Normalized unit is kilowatt-hour ($\text{kWh}$). MWh is converted to kWh ($\times 1000$).
* **Travel (Scope 3)**: Flight segment distance is calculated in kilometers ($\text{km}$) using the Haversine formula based on airport coordinates. Hotel usage is tracked in nights; car rentals are normalized to $\text{km}$.
