# Bank-Fraud-Detection-Analysis
End-to-end Bank Fraud Detection Analysis using Python, SQL and Power BI. Includes data cleaning, exploratory data analysis and interactive dashboards.

<img width="1407" height="792" alt="image" src="https://github.com/user-attachments/assets/2aeca19a-4117-421e-b32f-40869d732322" />
<img width="1408" height="797" alt="image" src="https://github.com/user-attachments/assets/1d5cbc95-3093-4a33-a6e0-9b25f26ce107" />



# 🏦 Bank Fraud Detection & Transaction Monitoring Analytics

> End-to-end analytics project: **SQL Server** (cleaning) → **Python** (EDA & feature prep) → **Power BI** (dashboard) for a retail bank's FY 2025–26 transaction monitoring.

![SQL Server](https://img.shields.io/badge/SQL%20Server-Cleaning-CC2927?logo=microsoftsqlserver&logoColor=white)
![Python](https://img.shields.io/badge/Python-pandas%20%7C%20seaborn-3776AB?logo=python&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)

---

## 📌 Project Overview

Banks lose money to fraud, and also lose time chasing alerts that turn out to be legitimate. This project analyses **75,034 transactions** and **5,025 customer records** to answer:

1. How big is the fraud problem (volume and value)?
2. Where is fraud concentrated: channel, merchant category, location, time of day?
3. Is the monitoring system catching fraud, and at what cost in false alerts?
4. Are high-value transactions crossing approval thresholds?

📄 Full requirements are documented in the [Business Requirements Document](<./BRD – Bank Fraud Detection.pdf>). | [📊 Dashboard Screenshots](./Screenshots)

---

## 🗂️ Repository Structure

```
├── data/
│   ├── Transactions_Cleaned.csv
│   ├── Customers_Cleaned.csv
│   ├── location_master.csv
│   ├── merchant_categories.csv
│   └── fraud_data.csv              # final dataset used in Power BI
├── sql/
│   └── Bank_Fraud_Data_Cleaning.sql
├── notebooks/
│   └── Bank_Fraud-Detection_EDA.ipynb
├── dashboard/
│   └── Bank_Fraud_Detection.pbix
├── images/
│   ├── fraud_landscape.png
│   └── detection_controls.png
├── docs/
│   └── BRD
└── README.md
```

---

## 🧰 Tech Stack

| Layer | Tools |
|---|---|
| Data cleaning | SQL Server Express, SSMS (T-SQL) |
| EDA & feature prep | Python, pandas, NumPy, seaborn, Matplotlib |
| Visualisation | Power BI (DAX, bookmarks, buttons) |
| Documentation | BRD, GitHub |

---

## 📊 Datasets

| Table | Rows | Description |
|---|---|---|
| `transactions` | 75,034 | Transaction ID, date, type, channel, location, account type, merchant category, amount, approver, approval status, alert flag, investigation status, fraud flag, remarks |
| `customers` | 5,025 | Home city, region, account type, monthly income, account open date, KYC status, risk rating, account status, status date |
| `location_master`, `merchant_categories` | — | Reference tables used as trusted lookups |

Data cleaning and EDA focus on the **Transactions** and **Customers** tables.

---

## 🧹 Step 1: Data Cleaning in SQL Server

Script: [`Bank_Fraud_Data_Cleaning.sql`](./sql/Bank_Fraud_Data_Cleaning.sql)

Raw CSVs were imported into the `BankFraudDetection` database, then cleaned with T-SQL.

### 1.1 Data quality checks
- Row counts after import for both tables
- Missing `Customer_ID`, `Transaction_ID` and `KYC_Status`
- Zero or negative `Transaction_Amount`
- Fraud-flagged transactions with no investigation status
- Duplicate detection on both tables

### 1.2 Fixing the amount column
`Transaction_Amount` arrived as text with symbols and shorthand (`₹`, `Rs`, commas, spaces, `K` suffix).
- Stripped currency symbols, commas and spaces with nested `REPLACE` inside `CROSS APPLY`
- Converted `25K` → `25000` with a `CASE` expression
- Used `TRY_CONVERT` to `DECIMAL(18,2)` so bad values become `NULL` instead of failing
- Added a clean column, dropped the old one, renamed with `sp_rename`
- Applied the same logic to `Monthly_Income` in the customers table

### 1.3 Standardising categorical values

| Column | Examples of fixes |
|---|---|
| `Transaction_Type` | `PYMT` → Payment, `WDRL` → Withdrawal, `DEP.` → Deposit, `TXFR` → Transfer |
| `Channel` | `NETBANKING`/`INTERNET` → Net Banking, `MOB` → Mobile, `BR.` → Branch, `P.O.S` → POS |
| `Location` / `Home_City` | `Bangalore`/`Bengalore` → Bengaluru, `Bombay` → Mumbai, `Gurgaon` → Gurugram, `AMD` → Ahmedabad |
| `Account_Type` | `SAV` → Savings, `CA` → Current |
| `KYC_Status` | `VERFIED` → Verified, `PEND.` → Pending, `EXP` → Expired |
| `Risk_Rating` | `L`/`MED`/`H` → Low / Medium / High |
| `Account_Status` | `ACTV` → Active |
| `Approval_Status` | `N/R` → Not Required, `AUTO APPROVED` → Auto-Approved |
| `Investigation_Status` | `FP` → False Positive, `CONFIRMED` → Confirmed Fraud |
| `Alert_Flag`, `Fraud_Flag` | `Yes/Y/TRUE` → 1, `No/N/FALSE` → 0 |

> 💡 Every `CASE` ends with `ELSE LTRIM(RTRIM(column))`, so unrecognised values are kept for review instead of silently turning into `NULL`.

### 1.4 Handling nulls, dates and duplicates
- `NULLIF(NULLIF(LTRIM(RTRIM(col)), ''), 'NULL')` converts blank strings and the text `'NULL'` into real `NULL`s across both tables
- `Account_Open_Date`: parsed with `TRY_CONVERT(DATE)`; future or invalid dates set to `NULL`
- `Status_Date`: set to `NULL` if unparseable or earlier than the account open date
- Created `customers_backup` before the destructive steps
- **Duplicates removed** with `ROW_NUMBER() OVER (PARTITION BY ... )` CTEs, keeping `RowNum = 1` for customers and deleting `RowNum > 1` for transactions

---

## 🐍 Step 2: Cleaning & EDA in Python

Notebook: [`Bank_Fraud-Detection_EDA.ipynb`](./notebooks/Bank_Fraud-Detection_EDA.ipynb)

### 2.1 Light-touch cleaning in pandas
- Parsed mixed-format dates with `pd.to_datetime(..., format='mixed')`
- Converted `Alert_Flag` and `Fraud_Flag` to nullable integers with `pd.to_numeric(errors='coerce')`
- Built `fraud_data` by dropping rows where `Fraud_Flag` is missing (**75,034 → 73,114 rows**) so every rate is calculated on labelled data only
- **Merchant category:** lower-cased and trimmed, then a mapping dictionary fixed typos (`grocry`, `electronis`, `resturants`, `entertainmnt`, `crypto exchnage` …). Non-merchant values (cash withdrawal, salary credit, peer transfer …) became `Not Applicable`; missing and `?` became `Unknown`
- **Location:** same approach with a city mapping (`bombay`, `poona`, `hyderbad`, `amd` …) and `Unknown` for blanks
- **Transaction type:** `fund transfer` standardised to `Transfer`

### 2.2 EDA questions and findings

| # | Business question | Key finding |
|---|---|---|
| Q1 | Which channels have the highest fraud rate? | **Online 4.93%** and **Net Banking 4.59%** lead; ATM (0.66%) and POS (0.71%) are lowest |
| Q2 | Which merchant categories carry the most exposure? | **Crypto Exchange (19.08%)** and **Online Gambling (18.45%)** have the highest rates; **Jewellery** has the highest fraud value (₹2.38M) |
| Q3 | Where is fraud concentrated geographically? | **Nagpur 4.68%**, Ahmedabad 3.74%, Pune 3.44%, Mumbai 3.32%; Mumbai has the highest fraud value (₹12.4M) |
| Q4 | Are threshold breaches controlled? | Thresholds applied per type (Transfer ₹2L, Payment ₹1L, Withdrawal ₹50K, Deposit ₹5L). Transfers breach most often (**1.26%**) |
| Q5 | How big is the problem? | **2.20% fraud rate**, **₹88.4M fraud value** |

The final dataset is exported to `fraud_data.csv` and loaded into Power BI as `fraud_data_clean`.

---

## 📈 Step 3: Power BI Dashboard

Two pages, switched with bookmarks and buttons: **Fraud Landscape** and **Detection & Controls**.

### Page 1: Fraud Landscape
![Fraud Landscape](./images/fraud_landscape.png)

| KPI | Value |
|---|---|
| Fraud Value | ₹88.4M |
| Fraud Transactions | ~1.6K |
| Fraud Rate | 2.20% |
| Affected Customers | 551 |

**Visuals:** monthly fraud-rate trend, fraud rate by hour of day (heatmap), riskiest merchant categories, fraud rate by channel, fraud rate by location.

**What it shows**
- 🌙 Fraud spikes between **midnight and 5 AM** (rates of roughly 10–16% per hour vs about 1% in the daytime)
- 📅 Fraud rate peaks in **July (3.5%)** and dips in May (1.2%) and November (1.5%)
- 💳 **Crypto Exchange (18.9%)** and **Online Gambling (18.0%)** are the riskiest merchant categories
- 🌐 **Online (4.9%)** and **Net Banking (4.6%)** are the riskiest channels
- 📍 **Nagpur (4.7%)** is the highest-risk location

### Page 2: Detection & Controls
![Detection & Controls](./images/detection_controls.png)

| KPI | Value |
|---|---|
| Alerts Raised | 3,085 |
| Alert Precision | 40.05% |
| Threshold Breaches | 393 |

**Visuals:** fraud investigation funnel, investigation remarks distribution, fraud-vs-alert status matrix, key insights panel.

| | Alerted | Not alerted |
|---|---|---|
| **Fraud** | 1,170 | 439 |
| **Legit** | 1,915 | 69,491 |

**What it shows**
- 1,915 legitimate transactions were wrongly flagged, so there is room to cut false positives
- 439 fraud transactions raised no alert, which points to detection gaps
- **Key focus:** improve detection coverage while reducing false-positive alerts

---

## 💡 Key Insights & Recommendations

1. Add **stricter step-up authentication** for Online and Net Banking transactions
2. Put **enhanced monitoring rules** on Crypto Exchange and Online Gambling merchants
3. Introduce **night-time (00:00–05:59) velocity limits** or additional verification
4. Tune alert rules to **reduce false positives** and close the gap on 439 missed frauds
5. Review **approval workflows** for transactions that breach type-wise thresholds
6. Run location-focused reviews in high-rate cities such as Nagpur, Ahmedabad and Pune

---

## ⚠️ Data Limitations
- Some `Transaction_Amount` values are missing, and negative amounts remain in the data and need business confirmation
- Rows with no fraud label were excluded from rate calculations
- Thresholds were assumed for analysis and should be replaced with the bank's actual policy limits

---

## ▶️ How to Reproduce
1. Create a SQL Server database and import the raw CSVs
2. Run `Bank_Fraud_Data_Cleaning.sql` and export the cleaned tables
3. Run the notebook to generate `fraud_data.csv`
4. Open the `.pbix` file and refresh the data source to `fraud_data.csv`

---

## 👤 Author
**Amartya Prasad**: Data Analyst
[LinkedIn](https://linkedin.com/in/amartyaprasaddatascientist25) · Bhopal, India
