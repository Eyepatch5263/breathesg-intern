# Breathe ESG Platform Prototype

An end-to-end carbon accounting and ESG reporting prototype. It ingests, normalizes, and calculates CO2e emissions for Scope 1 (Direct fuel), Scope 2 (Indirect electricity), and Scope 3 (Business travel) activities, offering audit trails and anomaly detection.

---

## 🛠️ Technology Stack

* **Backend**: Django & Django REST Framework (Python), SQLite database.
* **Frontend**: React (Vite), Vanilla CSS, responsive dashboard.
* **Environments**: Python Virtual Environment (`venv`), Node.js ecosystem.

---

## 📁 Project Directory Structure

* **`backend/`**: Django project containing APIs for data ingestion, audit logging, facility scoping, and record processing.
* **`frontend/`**: React application featuring data ingestion uploads, a normalized ledger table, and analytics dashboards.
* **`venv/`**: Python virtual environment storing application dependencies.
* **`run.sh`**: Helper shell script to migrate backend databases and start both servers concurrently.
* **`sample_*.csv/json`**: Sample input data for SAP (Scope 1), Utility (Scope 2), and Travel (Scope 3) flows.

---

## 🚀 Quick Start

1. **Prerequisites**: Ensure you have `python3` (with `venv`), `node` and `npm` installed.
2. **Setup Dependencies**:
   * **Backend**: Ensure the `venv` directory exists and has dependencies (`Django`, `djangorestframework`, etc.) installed.
   * **Frontend**: Navigate to `frontend/` and run `npm install`.
3. **Run Application**:
   Execute the root startup script to automatically migrate the database and spin up the backend and frontend dev servers:
   ```bash
   ./run.sh
   ```
   * **Backend REST API**: http://localhost:8000
   * **Frontend Application**: http://localhost:5173 (or as indicated in the terminal)

---

## 📖 Key Documentation

* **[`DECISIONS.md`](DECISIONS.md)**: Architectural and mathematical decisions (e.g., daily proration, Haversine formula, outlier flagging thresholds).
* **[`MODEL.md`](MODEL.md)**: Database ER schema and normalization rules.
* **[`SOURCES.md`](SOURCES.md)**: Details on carbon emission factors and data sources.
* **[`TRADEOFFS.md`](TRADEOFFS.md)**: Implemented tradeoffs and engineering decisions.
