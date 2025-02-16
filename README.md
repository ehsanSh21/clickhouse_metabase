# Fraud Detection Analysis using ClickHouse and Metabase

## Project Overview
This project focuses on analyzing fraud detection using a dataset obtained from Kaggle. The dataset consists of 23 columns and 550,000 rows of transaction data. The goal is to design, develop, and optimize a BI solution using **ClickHouse** for data storage and processing and **Metabase** for visualization and reporting.

## Technologies Used
- **Database**: ClickHouse
- **BI Tool**: Metabase
- **Programming Language**: Python (for ETL processes)
- **Libraries**: Pandas, SQLAlchemy

## Project Workflow
### 1. Data Import
The dataset was imported into ClickHouse by creating an initial raw transactions table:

```sql
CREATE TABLE IF NOT EXISTS credit_card_transactions (
    id Int32,  -- Unique ID for each row
    trans_date_trans_time String,
    cc_num String,
    merchant String,
    category String,
    amt Float32,
    first String,
    last String,
    gender String,
    street String,
    city String,
    state String,
    zip Int32,
    lat Float32,
    long Float32,
    city_pop Int32,
    job String,
    dob String,
    trans_num String,
    unix_time Int32,
    merch_lat Float32,
    merch_long Float32,
    is_fraud Int8
) ENGINE = MergeTree()
ORDER BY trans_num;
```

### 2. OLTP Data Model Creation
To simulate an OLTP system, the dataset was structured into normalized tables:

#### **City Table**
```sql
CREATE TABLE City
(
    City_id UInt32,
    City_name String,
    State String,
    City_population UInt32
) ENGINE = MergeTree()
ORDER BY City_id;
```

#### **Data Insertion**
```sql
INSERT INTO City (City_id, City_name, State, City_population)
SELECT
    rowNumberInAllBlocks() + 1 AS City_id,  -- generates sequential IDs starting from 1
    city, state, city_pop
FROM default.credit_card_transactions
GROUP BY city, state, city_pop
ORDER BY city;
```

#### **Address Table**
```sql
CREATE TABLE Address
(
    ADDR_id UInt32,
    Street String,
    Zip String,
    Lat Decimal(9, 6),
    Long Decimal(9, 6),
    City_id UInt32,
    INDEX idx_City_id (City_id) TYPE minmax GRANULARITY 3
) ENGINE = MergeTree()
ORDER BY ADDR_id;
```

```sql
INSERT INTO Address (ADDR_id, Street, Zip, Lat, Long, City_id)
SELECT
    rowNumberInAllBlocks() + 1 AS ADDR_id,
    street,
    zip,
    lat,
    long,
    b.City_id
FROM credit_card_transactions a
JOIN City b
    ON a.city = b.City_name
    AND a.state = b.State
GROUP BY street, zip, lat, long, a.city, a.state, b.City_id
ORDER BY b.City_id;
```

### 3. ETL Process & Analytical Data Model
A **denormalized data model** was created for optimized analytical queries, reducing the need for joins. The ETL (Extract, Transform, Load) process was implemented using Python and Pandas to efficiently process and transfer data.

#### **ETL Code Implementation**
```python
import os
import pandas as pd
from clickhouse_driver import Client

client = Client('localhost')  

# Extract Data from OLTP Tables
def extract_data():
    transactions = pd.DataFrame(client.execute("SELECT * FROM Transactions"), 
                                columns=['Transaction_ID', 'Amount', 'Transaction_TimeStamp', 'Unix_Time', 'Is_Fraud', 'Cust_id', 'Merchant_id'])
    
    customers = pd.DataFrame(client.execute("SELECT * FROM Customer"), 
                             columns=['Cust_ID', 'First_Name', 'Last_Name', 'Credit_Card_Number', 'Gender', 'Job', 'Date_Of_Birth', 'ADDR_id'])

    merchants = pd.DataFrame(client.execute("SELECT * FROM Merchant"), 
                             columns=['Merchant_ID', 'Merchant_Name', 'Lat', 'Long', 'Category_id'])

    categories = pd.DataFrame(client.execute("SELECT * FROM Category"), 
                              columns=['Category_id', 'Category_name'])

    addresses = pd.DataFrame(client.execute("SELECT * FROM Address"), 
                             columns=['ADDR_id', 'Street', 'Zip', 'Lat', 'Long', 'City_id'])

    cities = pd.DataFrame(client.execute("SELECT * FROM City"), 
                          columns=['City_id', 'City_name', 'State', 'City_population'])

    return transactions, customers, merchants, categories, addresses, cities


# Transform Data
def transform_data(transactions, customers, merchants, categories, addresses, cities):
    # Merge transactions with customers
    df = transactions.merge(customers, left_on='Cust_id', right_on='Cust_ID', how='left')

    # Merge with merchants
    df = df.merge(merchants, left_on='Merchant_id', right_on='Merchant_ID', how='left')

    # Merge with categories
    df = df.merge(categories, left_on='Category_id', right_on='Category_id', how='left')

    # Merge with addresses
    df = df.merge(addresses, left_on='ADDR_id', right_on='ADDR_id', how='left')

    # Merge with cities
    df = df.merge(cities, left_on='City_id', right_on='City_id', how='left')

    # Convert Date_Of_Birth to Age
    df['Age'] = (pd.to_datetime('today') - pd.to_datetime(df['Date_Of_Birth'], errors='coerce')).dt.days // 365

    # Select only necessary columns
    df = df[['Transaction_ID', 'Amount', 'Transaction_TimeStamp', 'Unix_Time', 'Is_Fraud', 
             'Cust_ID', 'First_Name', 'Last_Name', 'Gender', 'Age', 'Job',
             'Merchant_ID', 'Merchant_Name', 'Category_name',
             'City_name', 'State', 'Zip', 'Lat_x', 'Long_x']]

    # Rename Lat and Long columns
    df.rename(columns={'Lat_x': 'Lat', 'Long_x': 'Long'}, inplace=True)

    # Ensure Lat and Long are floats and handle missing values
    df['Lat'] = pd.to_numeric(df['Lat'], errors='coerce').fillna(0.0)
    df['Long'] = pd.to_numeric(df['Long'], errors='coerce').fillna(0.0)

    return df


# Load Data into ClickHouse
def load_data(df):
    # Convert dataframe to list of tuples for ClickHouse batch insert
    data_tuples = [tuple(row) for row in df.itertuples(index=False, name=None)]
    
    insert_query = """
    INSERT INTO transactions_analytics (
        Transaction_ID, Amount, Transaction_TimeStamp, Unix_Time, Is_Fraud,
        Cust_ID, First_Name, Last_Name, Gender, Age, Job,
        Merchant_ID, Merchant_Name, Category_name,
        City_name, State, Zip, Lat, Long
    ) VALUES
    """
    
    client.execute(insert_query, data_tuples)
    print("Data successfully loaded into ClickHouse!")

# Run ETL Pipeline
if __name__ == "__main__":
    print("Starting ETL process...")
    
    # Step 1: Extract Data
    transactions, customers, merchants, categories, addresses, cities = extract_data()
    print("Data extracted successfully.")

    # Step 2: Transform Data
    analytics_data = transform_data(transactions, customers, merchants, categories, addresses, cities)
    print("Data transformation completed.")

    # Step 3: Load Data
    load_data(analytics_data)
    print("ETL process finished successfully!")
```

A **denormalized data model** was created for optimized analytical queries, reducing the need for joins:

```sql
CREATE TABLE transactions_analytics
(
    Transaction_ID UInt64,
    Amount Float64,
    Transaction_TimeStamp DateTime,
    Unix_Time UInt64,
    Is_Fraud UInt8,

    -- Customer Information
    Cust_ID UInt64,
    First_Name String,
    Last_Name String,
    Gender String,
    Age UInt8,
    Job String,

    -- Merchant Information
    Merchant_ID UInt64,
    Merchant_Name String,
    Category_name String,  -- Directly denormalized from Category table

    -- Location Information
    City_name String,
    State String,
    Zip String,
    Lat Float64,
    Long Float64,

    -- Derived Time Dimensions
    Year UInt16 MATERIALIZED toYear(Transaction_TimeStamp),
    Month UInt8 MATERIALIZED toMonth(Transaction_TimeStamp),
    Day UInt8 MATERIALIZED toDayOfMonth(Transaction_TimeStamp),
    Hour UInt8 MATERIALIZED toHour(Transaction_TimeStamp),

    -- Index for performance
    INDEX idx_customer_gender Gender TYPE set(0) GRANULARITY 4,
    INDEX idx_merchant Merchant_ID TYPE set(0) GRANULARITY 4,
    INDEX idx_location State TYPE set(0) GRANULARITY 4
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(Transaction_TimeStamp)
ORDER BY (Transaction_TimeStamp, Cust_ID, Merchant_ID);

```

### 4. Data Visualization with Metabase
- Created **interactive dashboards** to track fraud trends.
- Designed visualizations to analyze transaction distribution, fraud frequency, and customer segmentation.
- Implemented **drill-down filters** for a more detailed view of fraud cases.

## Key Insights
- Identified transaction patterns that indicate potential fraud.
- Established KPIs for fraud risk assessment and anomaly detection.
- Improved data accessibility through well-structured BI dashboards.

## Future Enhancements
- Implement machine learning models for fraud prediction.
- Automate ETL workflows with **Apache Airflow**.
- Integrate real-time fraud detection alerts.

## Conclusion
This project successfully demonstrates the implementation of a **BI solution** using ClickHouse and Metabase to analyze fraud detection. By leveraging an optimized **OLTP and analytical data model**, data processing is significantly improved, and meaningful insights are generated for business decision-making.

---

