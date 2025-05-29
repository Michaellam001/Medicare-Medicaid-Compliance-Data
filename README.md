# Medicare/Medicaid Compliance Data Analysis with SQL

This repository showcases SQL skills applied to Medicare/Medicaid compliance data, demonstrating abilities in data analysis, query optimization, and compliance reporting.

## Data Sources

1. [CMS Public Use Files](https://www.cms.gov/medicare/enrollment-renewal/health-plans/trends)
2. [Medicare Provider Data](https://data.cms.gov/provider-data/)
3. [NPPES NPI Registry](https://download.cms.gov/nppes/NPI_Files.html)
4. [HHS-OIG Workplan](https://oig.hhs.gov/reports-and-publications/workplan/)

## SQL Skills Demonstrated

- Complex joins across compliance datasets
- Window functions for trend analysis
- Subqueries and CTEs for hierarchical data
- Data quality assessment techniques
- Fraud detection heuristics
- Performance optimization approaches

## Setup Instructions (Step-by-Step)

### 1. Download Sample Data

```bash
# Create project directory
mkdir medicare-compliance-sql
cd medicare-compliance-sql

# Create data directory
mkdir data

# Download sample datasets (replace with actual files when available)
wget -O data/medicare_providers.csv https://example.com/sample_providers.csv
wget -O data/compliance_audits.csv https://example.com/sample_audits.csv
wget -O data/claims_sample.csv https://example.com/sample_claims.csv

##  Setting up the Database

# Install PostgreSQL (if needed)
# sudo apt-get install postgresql postgresql-contrib

# Create database
createdb medicare_compliance

# Connect to database
psql medicare_compliance

-- Create tables
CREATE TABLE providers (
    npi VARCHAR(10) PRIMARY KEY,
    provider_name VARCHAR(255),
    entity_type VARCHAR(50),
    address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(2),
    zip VARCHAR(10),
    specialty VARCHAR(255)
);

CREATE TABLE claims (
    claim_id VARCHAR(20) PRIMARY KEY,
    attending_npi VARCHAR(10) REFERENCES providers(npi),
    beneficiary_id VARCHAR(20),
    service_date DATE,
    procedure_code VARCHAR(10),
    payment_amount DECIMAL(10,2),
    submitted_charge DECIMAL(10,2)
);

CREATE TABLE compliance_audits (
    audit_id VARCHAR(20) PRIMARY KEY,
    npi VARCHAR(10) REFERENCES providers(npi),
    audit_date DATE,
    audit_type VARCHAR(100),
    findings VARCHAR(50),
    recovery_amount DECIMAL(10,2),
    corrective_action VARCHAR(255)
);

-- Import data
\copy providers FROM 'data/medicare_providers.csv' WITH CSV HEADER;
\copy claims FROM 'data/claims_sample.csv' WITH CSV HEADER;
\copy compliance_audits FROM 'data/compliance_audits.csv' WITH CSV HEADER;

## queries/compliance_trends.sql
-- Analyze compliance audit trends by year and type
SELECT 
    EXTRACT(YEAR FROM audit_date) AS year,
    audit_type,
    COUNT(*) AS total_audits,
    SUM(CASE WHEN findings = 'Non-compliant' THEN 1 ELSE 0 END) AS non_compliant,
    ROUND(SUM(CASE WHEN findings = 'Non-compliant' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS non_compliance_rate,
    AVG(recovery_amount) AS avg_recovery,
    SUM(recovery_amount) AS total_recovery,
    -- Year-over-year comparison
    LAG(COUNT(*), 1) OVER (PARTITION BY audit_type ORDER BY EXTRACT(YEAR FROM audit_date)) AS prev_year_audits,
    ROUND((COUNT(*) - LAG(COUNT(*), 1) OVER (PARTITION BY audit_type ORDER BY EXTRACT(YEAR FROM audit_date))) * 100.0 / 
        LAG(COUNT(*), 1) OVER (PARTITION BY audit_type ORDER BY EXTRACT(YEAR FROM audit_date)), 2) AS yoy_change_pct
FROM 
    compliance_audits
WHERE 
    audit_date BETWEEN '2018-01-01' AND '2022-12-31'
GROUP BY 
    EXTRACT(YEAR FROM audit_date), audit_type
ORDER BY 
    year, non_compliance_rate DESC;

## Running the Analysis
# Execute queries and save results
psql medicare_compliance -f queries/high_risk_providers.sql -o results/high_risk_providers.txt
psql medicare_compliance -f queries/compliance_trends.sql -o results/compliance_trends.txt
