# Ingestion Formats & Data Sources (`SOURCES.md`)

This document details the standard formats, carbon coefficients, sample structures, and common ingestion failure modes analyzed for the Breathe ESG integration pipeline.

---

## 1. Emission Factors & Coefficients Used

My parsing engines leverage recognized environmental standards (primarily DEFRA and IEA):

### Scope 1: Direct Combustion (DEFRA)
* **Diesel (Automotive / Generator)**: $2.68\text{ kg CO2e / Liter}$
* **Heating Oil**: $2.54\text{ kg CO2e / Liter}$
* **Natural Gas**: $2.02\text{ kg CO2e / }m^3$

### Scope 2: Indirect Purchased Electricity (IEA Grid Coefficients)
* **Germany (DE)**: $0.401\text{ kg CO2e / kWh}$
* **United States (US)**: $0.385\text{ kg CO2e / kWh}$
* **Singapore (SG)**: $0.408\text{ kg CO2e / kWh}$

### Scope 3: Corporate Travel (DEFRA & ICAO)
* **Air Travel (Distance HAUL boundaries)**:
  * Short-Haul ($< 3,700\text{ km}$): $0.15\text{ kg CO2e / km}$
  * Long-Haul ($\ge 3,700\text{ km}$): $0.12\text{ kg CO2e / km}$
  * Cabin Class Multipliers: Economy = $\times 1.0$, Business = $\times 1.5$, First = $\times 2.0$
* **Hotel Lodging (per night)**:
  * Germany (DE): $11.2\text{ kg CO2e / night}$
  * United States (US): $15.4\text{ kg CO2e / night}$
  * Singapore (SG): $18.2\text{ kg CO2e / night}$
* **Car Rental (mileage)**:
  * Gasoline/Diesel car: $0.17\text{ kg CO2e / km}$

---

## 2. Inbound Data Formats & Sample Payloads

### A. SAP Procurement CSV Structure
Commonly extracted from SAP tables (like `EKPO` for purchase orders and `MSEG` for material movements).

* **Header (German)**: `Einkaufsbeleg,Buchungsdatum,Materialnummer,Werk,Menge,Basismengeneinheit,Betrag`
* **Header (English)**: `EBELN,BUDAT,MATNR,WERKS,MENG,MEINS,WRBTR`
* **Sample Raw Line**:
  ```csv
  45000123,20260510,1002,1000,150.5,L,300.00
  ```

### B. Utility Billing CSV Structure
Typically exported from corporate billing portals (e.g. Urjanet or Schneider Resource Advisor).

* **Headers**: `Account Number,Meter Number,Billing Start Date,Billing End Date,Usage kWh,Total Cost`
* **Sample Raw Line**:
  ```csv
  ACC-1000,MTR-1000,2026-01-15,2026-02-14,3100,620.00
  ```

### C. Concur Travel JSON Payload
Nested flight segment and hotel stay objects returned from corporate booking systems.

```json
{
  "id": "TRIP-001",
  "traveler": {"name": "Alice Smith"},
  "bookings": [
    {
      "type": "AIR",
      "departure_airport": "JFK",
      "arrival_airport": "LHR",
      "cabin_class": "BUSINESS",
      "date": "2026-05-01"
    },
    {
      "type": "HOTEL",
      "city": "US",
      "nights": 3,
      "check_in_date": "2026-05-02"
    }
  ]
}
```

---

## 3. Real-World Data Inconsistency & Failure Modes

During research, the following failure cases were identified and coded into our parser validation checks:

1. **Date Format Chaos**: SAP exports often use string compact dates (`YYYYMMDD` like `20260510`) or German dot separators (`10.05.2026`). The SAP parser handles this by testing both regular expressions and converting them into ISO dates.
2. **Missing Facility Codes**: When a transaction record lists a facility code (e.g. SAP `WERKS`) that is not pre-registered in our database, the parser flags the ingestion run status as `PARTIAL_FAILURE` and records the row in `RawRecord` with a status of `FAILED` with a validation warning: `Facility code 9999 not found`.
3. **Mass-to-Volume Unit Mismatch**: Liquid fuels are sometimes registered in Mass units (`KG` or `TO`) instead of volume (`L`). The parser resolves this by dividing the mass by the fuel's density coefficient before applying the volumetric emission factors.
4. **Billing Cycle Spans**: When a billing record spans two calendar months, the system daily-prorates the consumption rather than assigning the full carbon load to the end-date month, preventing artificial spikes.
5. **Missing IATA Code Coordinate**: When a traveler flies to a minor hub not in our database, the system logs a validation warning and applies a standard distance fallback rather than crashing the file import.
