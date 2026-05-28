# Engineering Tradeoffs (`TRADEOFFS.md`)

This document outlines three deliberate engineering omissions made during the implementation of the Breathe ESG prototype, explaining the rationale behind each choice.

---

## 1. Static Emission Factors vs. Live Grid Factor API Integration

* **Omission**: I didn't integrate with live electricity grid factor APIs (e.g., ElectricityMap, eGRID, or Entso-E) to query real-time marginal grid intensities.
* **Why**:
  1. **Auditability**: External ESG auditors require carbon footprint calculations to be fully reproducible. Dynamic APIs that return hourly real-time marginal carbon intensities can change their historical datasets post-facto. Static grid factors (from DEFRA and IEA) ensure calculations remain stable, repeatable, and easily verifiable.
  2. **Reliability & Performance**: Relying on external network requests during bulk ingestion runs creates a single point of failure and slows down parsing execution. Hardcoding standard country factors ensures sub-second ingestion rates.

---

## 2. Structured CSV Ingestion vs. Automated PDF Invoice OCR

* **Omission**: I didn't implement Optical Character Recognition (OCR) tools (e.g., AWS Textract or Tesseract OCR) to parse scanned PDF or image copies of utility bills.
* **Why**:
  1. **Complexity & Failure Modes**: OCR extraction is non-deterministic. A simple wrinkle in a scan can lead to reading a `3` as an `8` or missing a decimal point (turning a $1,000$ kWh bill into $10,000$ kWh). Correcting these mistakes creates more operational overhead than upload spreadsheets.
  2. **Efficiency**: Most enterprise utility portals offer structured CSV exports. Designing our Utility Ingestor to consume these CSVs allows us to process thousands of bills instantly with 100% accuracy.

---

## 3. Pre-converted Local Currency vs. Live Currency Exchange API

* **Omission**: I didn't integrate with a live currency conversion API (such as OpenExchangeRates) to handle transactions billed in non-base currencies.
* **Why**:
  1. **Historical Currency Fluctuations**: Carbon calculations look at historical records spanning months or years. If a bill from 2025 is parsed, converting it using current exchange rates would distort the cost-to-quantity ratio.
  2. **Pragmatic Scope**: Enterprise ERP systems (like SAP) already pre-calculate conversions and export values in uniform local or corporate currencies. For this prototype, assuming costs are pre-aligned avoids adding redundant exchange-rate layers.

## 4. OData vs. Alternatives

* **Omission**: I didn't chose OData Integration Service(REST API) over legacy methods like IDocs(Too complex and highly), Flat Files like (CSV/TXT)(Too fragile and requires schedule batch jobs, manual file management and frequently breaks when column structure changes),BAPIs(Reliable but requires specialized SAP-specific protocols(RFC) and proprietary connectors to extract data), Whereas OData rurns SAP Tables into standarized RESTful JSON APIs and also supports native filtering using ($filter), hanldes pagination. 
* **Why**:
  1. **Limited Time**: Building a API Integration requires testing against a mock SAP server that accurately replicates SAP's proprietary network behaviours, paging, delta-token tracking. 