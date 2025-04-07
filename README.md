# 📁 Project: HIPAA-Compliant Healthcare Claims Forecasting Pipeline on Azure

# ───────────────────────────────
# Root Directory Structure
# ───────────────────────────────

# /hipaa-claims-pipeline
# ├── data/                   # Sample Synthea files (CSV/JSON)
# ├── etl/                    # Azure Function for ingesting and cleaning claims
# │   └── process_claims.py
# ├── sql/                    # SQL schema and aggregation views
# │   └── claims_schema.sql
# ├── forecasting/            # Python notebook or script for Prophet model
# │   └── forecast_claims.py
# ├── dashboards/             # Power BI .pbix files or report instructions
# ├── infra/                  # Bicep or Terraform templates (future add-on)
# ├── docs/                   # HIPAA compliance plan, architecture diagrams
# └── README.md

# ───────────────────────────────
# 📄 etl/process_claims.py
# ───────────────────────────────
    import hashlib
    import csv
    import pyodbc
    import os
    
    def hash_id(value):
        return hashlib.sha256(value.encode()).hexdigest()
    
    def process_claims(blob_bytes):
        decoded = blob_bytes.decode('utf-8').splitlines()
        reader = csv.DictReader(decoded)

        rows = []
        for row in reader:
            rows.append((
                hash_id(row['patient_id']),
                row['procedure_code'],
                row['diagnosis_code'],
                row['date_of_service'],
                row['provider_id'],
                row['payer'],
                float(row['total_billed']),
                float(row['amount_paid']),
                row['status']
            ))
        return rows
    
    def upload_to_sql(rows):
        conn = pyodbc.connect(os.getenv('SQL_CONN_STR'))
        cursor = conn.cursor()
        for r in rows:
            cursor.execute("""
                INSERT INTO claims
                (patient_id, procedure_code, diagnosis_code, date_of_service, provider_id, payer, total_billed, amount_paid, status)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """, r)
        conn.commit()

# ───────────────────────────────
# 📄 sql/claims_schema.sql
# ───────────────────────────────
    CREATE TABLE claims (
        claim_id INT IDENTITY PRIMARY KEY,
        patient_id VARCHAR(64),
        procedure_code VARCHAR(10),
        diagnosis_code VARCHAR(10),
        date_of_service DATE,
        provider_id VARCHAR(20),
        payer VARCHAR(50),
        total_billed DECIMAL(10,2),
        amount_paid DECIMAL(10,2),
        status VARCHAR(20)
    );
    
    -- Monthly aggregation view
    CREATE VIEW monthly_paid AS
    SELECT 
        FORMAT(date_of_service, 'yyyy-MM') AS month,
        SUM(amount_paid) AS total_paid
    FROM claims
    WHERE status = 'Paid'
    GROUP BY FORMAT(date_of_service, 'yyyy-MM');

# ───────────────────────────────
# 📄 forecasting/forecast_claims.py
# ───────────────────────────────
    import pandas as pd
    from prophet import Prophet
    import matplotlib.pyplot as plt
    
    # Load from SQL export
    df = pd.read_csv('monthly_paid.csv')
    df.rename(columns={"month": "ds", "total_paid": "y"}, inplace=True)
    
    model = Prophet()
    model.fit(df)
    
    future = model.make_future_dataframe(periods=6, freq='M')
    forecast = model.predict(future)
    
    model.plot(forecast)
    plt.title('Forecasted Claim Payouts')
    plt.xlabel('Month')
    plt.ylabel('Total Paid')
    plt.tight_layout()
    plt.show()
    
    # Save to CSV
    forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].to_csv('forecast_output.csv', index=False)

# ───────────────────────────────
# 📄 README.md (Summary)
# ───────────────────────────────
"""
# HIPAA-Compliant Healthcare Claims Forecasting Pipeline (Azure)

## 🧾 Overview
This project processes synthetic claims data from Synthea and builds a HIPAA-compliant claims ingestion, storage, analysis, and forecasting system on Azure.

## 🛠️ Tools Used
- **Azure Blob Storage**: Secure storage of uploaded claims files
- **Azure Function**: Serverless ETL with Python for ingestion + de-identification
- **Azure SQL Database**: Encrypted relational DB to store claims
- **Azure Key Vault**: Securely store database credentials
- **Power BI**: Build dashboards with role-based access control
- **Python (Prophet)**: Forecast future payouts based on historical claims

## ✅ HIPAA Compliance Features
- 🔐 **De-identification**: Patient identifiers are hashed using SHA-256
- 🔒 **Encryption**: Data encrypted at rest (TDE) and in transit (TLS)
- 👤 **Access Control**: Role-Based Access Control (RBAC) using Azure AD
- 📊 **Minimal Disclosure**: Only claim-essential fields are processed
- 🕵️‍♀️ **Audit Logging**: Diagnostic settings + Azure Monitor logs

## 📦 Execution Steps
1. Generate Synthea data and save to `/data/`
2. Upload CSV to Azure Blob Storage
3. Deploy Azure infrastructure:
   - Blob Storage with private endpoint
   - Azure SQL Database with TDE
   - Azure Function (Python runtime)
   - Key Vault with secure secrets
4. Configure Function to trigger on Blob upload
5. Function de-identifies and inserts data to SQL
6. Use Power BI to:
   - Connect to SQL
   - Build visual reports
   - Enable built-in forecasting
7. Run `forecast_claims.py` for Prophet-based predictions and export
8. Add documentation and architecture diagrams under `/docs`

## 📁 Structure
- `etl/`: Claim cleaning + ingestion
- `sql/`: Database schema and views
- `forecasting/`: Predictive modeling scripts
- `dashboards/`: Power BI (coming soon)
- `docs/`: HIPAA compliance summary + diagrams
- `infra/`: Infrastructure as code (optional)

"""
