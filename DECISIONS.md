# Architectural Decisions (`DECISIONS.md`)

This document outlines key technical decisions made during the design and implementation of the Breathe ESG platform, highlighting resolved engineering ambiguities and mathematical methodologies.

---

## 1. Billing Cycle Proration Math

**Problem**: Utility portal billing invoices do not align with clean calendar months (e.g., January 15th to February 14th).
* **Decision**: We perform **mathematical daily proration**.
* **Methodology**: 
  1. Determine the total number of days in the billing cycle ($D = \text{End Date} - \text{Start Date} + 1$).
  2. Compute daily consumption rate ($R = \frac{\text{Usage kWh}}{D}$).
  3. Map daily usage to each overlapping calendar month.
  4. Create separate, distinct `NormalizedEmissionRecord` instances for each month, using the 1st of that month as the `activity_date`.
* **Example**: A bill of $3,100\text{ kWh}$ from Jan 15 to Feb 14 ($31\text{ days}$) splits into:
  * January (17 days): $1,700\text{ kWh}$ ($1,700 \times \text{Grid Factor}$)
  * February (14 days): $1,400\text{ kWh}$ ($1,400 \times \text{Grid Factor}$)
* **Benefit**: Correctly reports monthly dashboard summaries without distorting peaks and valleys, while preserving the link to the single raw utility invoice record for audit trail.

---

## 2. Distance Calculations (Deterministic Airport Database)

**Problem**: Flight segments inside Concur payloads only provide IATA code strings (e.g., `JFK`, `LHR`, `SIN`). Querying live geocoding APIs is slow, rate-limited, non-deterministic, and breaks in offline environments.
* **Decision**: We embedded a **geodesic coordinate table of the top global hubs** directly in the parser.
* **Methodology**: The parser implements the **Haversine formula**:
  $$d = 2 R \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta \varphi}{2}\right) + \cos(\varphi_1)\cos(\varphi_2)\sin^2\left(\frac{\Delta \lambda}{2}\right)}\right)$$
  where $\varphi$ is latitude, $\lambda$ is longitude, and $R = 6371\text{ km}$.
* **Fallbacks**: If IATA codes are not in our pre-seeded database, the system logs a validation warning and falls back to a conservative default flight distance ($1,500\text{ km}$ per segment).

---

## 3. Outlier Flagging Algorithm

**Problem**: Ingested files may contain decimal typos (e.g., $10,000$ gallons instead of $1,000$) or billing errors.
* **Decision**: We flag records as `SUSPICIOUS` automatically if their cost-to-quantity ratio falls outside standard economic ranges.
* **Thresholds**:
  * **Diesel/Heating Oil**: $< \$0.20\text{ / Liter}$ or $> \$5.00\text{ / Liter}$.
  * **Electricity**: $< \$0.02\text{ / kWh}$ or $> \$1.50\text{ / kWh}$.
  * **Flights**: Mileage segment $> 15,000\text{ km}$ (longer than the longest commercial flight).
  * **Car Rental**: Rental distance $> 3,000\text{ km}$.
* **Benefit**: Suspicious records are clearly highlighted in yellow on the UI, giving analysts an immediate visual warning to review the raw record and correct errors before clicking "Approve".

---

## 4. Record State Locking

**Problem**: An approved record should not be subject to accidental edits or updates from re-uploading historical files.
* **Decision**: We introduced a Boolean field `is_locked` inside `NormalizedEmissionRecord`. Once the analyst approves a row, `is_locked` is set to `True`. Subsequent manual edits and parser uploads are blocked from modifying this row.

## 5. SAP Ingestion Pivot: Flat Files vs OData API

**Problem**: Live OData REST service is not available and SAP data needs to be ingested.
* **Decision**: The ingestion mechanism for SAP was pivoted from a live OData REST service to Flat File Ingestion (CSV/JSON dumps).
