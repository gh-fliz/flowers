# Databricks Hands-On Practice Guide
*Master the Platform Through Real-World Scenarios*

## 🚀 Getting Started: Platform Navigation (15 minutes)

### Step 1: Access Databricks SQL
1. Log into your Databricks workspace
2. Navigate to **SQL** in the left sidebar (or use the switcher in top-left)
3. Familiarize yourself with the main sections:
   - **Queries** - Where you write and save SQL
   - **Dashboards** - For visualization creation
   - **Data** - Schema browser and table explorer
   - **Warehouses** - Compute management

### Step 2: Create Your First SQL Warehouse
1. Go to **SQL Warehouses** → **Create SQL Warehouse**
2. Choose **Serverless** for instant startup (if available)
3. Or select **Pro** with **Small** size for cost efficiency
4. Name it `learning-warehouse`
5. Set **Auto Stop** to 10 minutes to save costs

---

## 📊 Phase 1: Table Creation & Data Loading (30 minutes)

### Scenario: Building a Sales Analytics Foundation

**Goal**: Create a medallion architecture for sales data analysis

#### Create Sample Data Tables

```sql
-- Bronze Layer: Raw Sales Data
CREATE OR REPLACE TABLE bronze_sales (
    transaction_id STRING,
    customer_id STRING,
    product_id STRING,
    transaction_date STRING,
    amount STRING,
    quantity STRING,
    sales_rep STRING,
    region STRING
) USING DELTA;

-- Insert sample data
INSERT INTO bronze_sales VALUES
('TXN001', 'CUST001', 'PROD001', '2024-01-15', '150.50', '2', 'John Smith', 'North'),
('TXN002', 'CUST002', 'PROD002', '2024-01-16', '75.25', '1', 'Jane Doe', 'South'),
('TXN003', 'CUST001', 'PROD003', '2024-01-17', '200.00', '3', 'John Smith', 'North'),
('TXN004', 'CUST003', 'PROD001', '2024-01-18', '150.50', '2', 'Bob Johnson', 'West'),
('TXN005', 'CUST002', 'PROD002', '2024-01-19', '225.75', '3', 'Jane Doe', 'South');

-- Create Customer Dimension Table
CREATE OR REPLACE TABLE dim_customers (
    customer_id STRING,
    customer_name STRING,
    customer_type STRING,
    registration_date DATE
) USING DELTA;

INSERT INTO dim_customers VALUES
('CUST001', 'Acme Corp', 'Enterprise', '2023-06-01'),
('CUST002', 'Tech Solutions', 'SMB', '2023-08-15'),
('CUST003', 'Global Industries', 'Enterprise', '2023-09-01');
```

#### Practice File Upload (Alternative Method)
1. Go to **Data** → **Create Table**
2. Click **Upload File**
3. Create a CSV with sample data:
   ```csv
   product_id,product_name,category,price
   PROD001,Widget A,Electronics,75.25
   PROD002,Widget B,Home,50.50
   PROD003,Widget C,Electronics,66.67
   ```
4. Upload and create table named `dim_products`

---

## 🔧 Phase 2: Complex Query Development (45 minutes)

### Silver Layer: Data Cleansing & Transformation

```sql
-- Silver Layer: Cleaned Sales Data
CREATE OR REPLACE TABLE silver_sales AS
SELECT 
    transaction_id,
    customer_id,
    product_id,
    CAST(transaction_date AS DATE) as transaction_date,
    CAST(amount AS DECIMAL(10,2)) as amount,
    CAST(quantity AS INT) as quantity,
    sales_rep,
    region,
    current_timestamp() as processed_at
FROM bronze_sales
WHERE amount IS NOT NULL 
  AND quantity > 0
  AND transaction_date IS NOT NULL;
```

### Advanced Query Patterns

#### 1. Window Functions for Rankings
```sql
-- Top customers by revenue with ranking
SELECT 
    customer_id,
    total_revenue,
    RANK() OVER (ORDER BY total_revenue DESC) as revenue_rank,
    DENSE_RANK() OVER (ORDER BY total_revenue DESC) as dense_rank,
    ROW_NUMBER() OVER (ORDER BY total_revenue DESC) as row_num
FROM (
    SELECT 
        customer_id,
        SUM(amount) as total_revenue
    FROM silver_sales
    GROUP BY customer_id
);
```

#### 2. Complex Aggregations with CUBE
```sql
-- Sales analysis by region and sales rep
SELECT 
    COALESCE(region, 'ALL_REGIONS') as region,
    COALESCE(sales_rep, 'ALL_REPS') as sales_rep,
    COUNT(*) as transaction_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_transaction_value
FROM silver_sales
GROUP BY CUBE(region, sales_rep)
ORDER BY region, sales_rep;
```

#### 3. Advanced JOIN with Analytics
```sql
-- Customer analysis with product preferences
SELECT 
    c.customer_name,
    c.customer_type,
    COUNT(s.transaction_id) as total_transactions,
    SUM(s.amount) as total_spent,
    AVG(s.amount) as avg_transaction,
    STRING_AGG(DISTINCT s.product_id, ', ') as products_purchased,
    DATEDIFF(CURRENT_DATE(), MAX(s.transaction_date)) as days_since_last_purchase
FROM dim_customers c
LEFT JOIN silver_sales s ON c.customer_id = s.customer_id
GROUP BY c.customer_id, c.customer_name, c.customer_type
ORDER BY total_spent DESC;
```

### Practice MERGE INTO Operations

```sql
-- Create target table for upserts
CREATE OR REPLACE TABLE customer_metrics (
    customer_id STRING,
    total_revenue DECIMAL(10,2),
    transaction_count INT,
    last_purchase_date DATE,
    updated_at TIMESTAMP
) USING DELTA;

-- MERGE INTO operation
MERGE INTO customer_metrics t
USING (
    SELECT 
        customer_id,
        SUM(amount) as total_revenue,
        COUNT(*) as transaction_count,
        MAX(transaction_date) as last_purchase_date
    FROM silver_sales
    GROUP BY customer_id
) s ON t.customer_id = s.customer_id
WHEN MATCHED THEN UPDATE SET
    t.total_revenue = s.total_revenue,
    t.transaction_count = s.transaction_count,
    t.last_purchase_date = s.last_purchase_date,
    t.updated_at = current_timestamp()
WHEN NOT MATCHED THEN INSERT (
    customer_id, total_revenue, transaction_count, 
    last_purchase_date, updated_at
) VALUES (
    s.customer_id, s.total_revenue, s.transaction_count, 
    s.last_purchase_date, current_timestamp()
);
```

---

## 🎨 Phase 3: Dashboard Creation (30 minutes)

### Create Business Intelligence Queries

#### 1. Revenue Trend Query
```sql
-- Save as "Revenue Trend"
SELECT 
    DATE_TRUNC('month', transaction_date) as month,
    SUM(amount) as monthly_revenue,
    COUNT(*) as transaction_count,
    AVG(amount) as avg_transaction_value
FROM silver_sales
GROUP BY DATE_TRUNC('month', transaction_date)
ORDER BY month;
```

#### 2. Regional Performance Query
```sql
-- Save as "Regional Performance"
SELECT 
    region,
    COUNT(*) as transactions,
    SUM(amount) as revenue,
    AVG(amount) as avg_transaction,
    COUNT(DISTINCT customer_id) as unique_customers
FROM silver_sales
GROUP BY region
ORDER BY revenue DESC;
```

#### 3. Top Products Query with Parameters
```sql
-- Save as "Top Products Analysis"
SELECT 
    product_id,
    SUM(amount) as total_revenue,
    SUM(quantity) as total_quantity,
    COUNT(*) as transaction_count,
    AVG(amount) as avg_sale_amount
FROM silver_sales
WHERE transaction_date >= '{{start_date}}'
  AND transaction_date <= '{{end_date}}'
GROUP BY product_id
ORDER BY total_revenue DESC
LIMIT {{top_n}};
```

### Build Interactive Dashboard

1. **Create New Dashboard**: Go to **Dashboards** → **Create Dashboard**
2. **Name it**: "Sales Performance Dashboard"
3. **Add Visualizations**:
   - **Revenue Trend**: Line chart showing monthly revenue
   - **Regional Performance**: Bar chart or pie chart
   - **Top Products**: Table with conditional formatting

4. **Add Parameters**:
   - `start_date`: Date picker (default: 30 days ago)
   - `end_date`: Date picker (default: today)
   - `top_n`: Number input (default: 10)

5. **Format Visualizations**:
   - Add titles and axis labels
   - Use appropriate colors
   - Enable hover tooltips
   - Format currency fields

---

## 🔄 Phase 4: ETL Pipeline Implementation (45 minutes)

### Create Gold Layer Analytics Tables

#### 1. Daily Sales Summary
```sql
-- Gold Layer: Daily business metrics
CREATE OR REPLACE TABLE gold_daily_sales AS
SELECT 
    transaction_date,
    COUNT(*) as daily_transactions,
    SUM(amount) as daily_revenue,
    AVG(amount) as avg_transaction_value,
    COUNT(DISTINCT customer_id) as unique_customers,
    COUNT(DISTINCT product_id) as unique_products,
    SUM(quantity) as total_quantity_sold
FROM silver_sales
GROUP BY transaction_date;
```

#### 2. Customer Segmentation
```sql
-- Gold Layer: Customer segments
CREATE OR REPLACE TABLE gold_customer_segments AS
SELECT 
    customer_id,
    customer_name,
    customer_type,
    total_revenue,
    transaction_count,
    avg_transaction_value,
    days_since_last_purchase,
    CASE 
        WHEN total_revenue >= 500 THEN 'High Value'
        WHEN total_revenue >= 200 THEN 'Medium Value'
        ELSE 'Low Value'
    END as value_segment,
    CASE 
        WHEN days_since_last_purchase <= 30 THEN 'Active'
        WHEN days_since_last_purchase <= 90 THEN 'At Risk'
        ELSE 'Inactive'
    END as engagement_status
FROM (
    SELECT 
        s.customer_id,
        c.customer_name,
        c.customer_type,
        SUM(s.amount) as total_revenue,
        COUNT(s.transaction_id) as transaction_count,
        AVG(s.amount) as avg_transaction_value,
        DATEDIFF(CURRENT_DATE(), MAX(s.transaction_date)) as days_since_last_purchase
    FROM silver_sales s
    JOIN dim_customers c ON s.customer_id = c.customer_id
    GROUP BY s.customer_id, c.customer_name, c.customer_type
);
```

### Create User-Defined Function

```sql
-- Create UDF for customer lifetime value calculation
CREATE OR REPLACE FUNCTION calculate_clv(
    avg_transaction_value DECIMAL(10,2),
    transaction_frequency INT,
    customer_lifespan_months INT
)
RETURNS DECIMAL(10,2)
LANGUAGE SQL
CONTAINS SQL
DETERMINISTIC
RETURN avg_transaction_value * transaction_frequency * customer_lifespan_months;

-- Use the UDF
SELECT 
    customer_id,
    customer_name,
    avg_transaction_value,
    transaction_count,
    calculate_clv(avg_transaction_value, transaction_count, 24) as estimated_clv
FROM gold_customer_segments
ORDER BY estimated_clv DESC;
```

### Implement Incremental Processing

```sql
-- Create staging table for new data
CREATE OR REPLACE TABLE staging_sales (
    transaction_id STRING,
    customer_id STRING,
    product_id STRING,
    transaction_date DATE,
    amount DECIMAL(10,2),
    quantity INT,
    sales_rep STRING,
    region STRING,
    load_timestamp TIMESTAMP DEFAULT current_timestamp()
) USING DELTA;

-- Incremental load process
MERGE INTO silver_sales target
USING (
    SELECT * FROM staging_sales 
    WHERE load_timestamp > (
        SELECT COALESCE(MAX(processed_at), '1900-01-01') 
        FROM silver_sales
    )
) source
ON target.transaction_id = source.transaction_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

---

## 🧪 Phase 5: Testing & Optimization (20 minutes)

### Data Quality Checks

```sql
-- Create data quality monitoring
CREATE OR REPLACE TABLE data_quality_checks AS
SELECT 
    'silver_sales' as table_name,
    COUNT(*) as total_records,
    COUNT(DISTINCT transaction_id) as unique_transactions,
    SUM(CASE WHEN amount IS NULL THEN 1 ELSE 0 END) as null_amounts,
    SUM(CASE WHEN quantity <= 0 THEN 1 ELSE 0 END) as invalid_quantities,
    MIN(transaction_date) as earliest_date,
    MAX(transaction_date) as latest_date,
    current_timestamp() as check_timestamp
FROM silver_sales;
```

### Performance Optimization

```sql
-- Optimize Delta table
OPTIMIZE silver_sales;

-- Z-order by commonly filtered columns
OPTIMIZE silver_sales ZORDER BY (transaction_date, region);

-- Analyze table for statistics
ANALYZE TABLE silver_sales COMPUTE STATISTICS;
```

### Test Delta Time Travel

```sql
-- View table history
DESCRIBE HISTORY silver_sales;

-- Query previous version
SELECT COUNT(*) FROM silver_sales VERSION AS OF 1;

-- Restore to previous version if needed
-- RESTORE TABLE silver_sales TO VERSION AS OF 1;
```

---

## 🎯 Practice Scenarios for Exam Prep

### Scenario 1: Data Loading Challenge
1. Upload a CSV file with intentionally messy data
2. Create bronze table with string columns
3. Transform to silver with proper data types
4. Handle nulls and invalid values
5. Create gold layer with business metrics

### Scenario 2: Dashboard with Parameters
1. Create a parameterized query for date ranges
2. Build dashboard with multiple visualizations
3. Add filters and interactive elements
4. Schedule automatic refresh
5. Share with stakeholders

### Scenario 3: ETL Pipeline
1. Design a complete medallion architecture
2. Implement incremental loading
3. Add data quality checks
4. Create monitoring queries
5. Optimize for performance

### Scenario 4: Complex Analytics
1. Write queries with window functions
2. Create customer cohort analysis
3. Build product recommendation logic
4. Calculate business KPIs
5. Handle slowly changing dimensions

---

## 📚 Key Takeaways

**Navigation Mastery**: Know where to find everything quickly - SQL editor, warehouse management, data explorer, and dashboard builder.

**SQL Proficiency**: Master JOINs, window functions, aggregations, and Delta-specific operations like MERGE INTO.

**Architecture Understanding**: Implement medallion architecture with proper data flow and transformation logic.

**Dashboard Skills**: Create interactive, parameterized dashboards with proper formatting and business context.

**Performance Awareness**: Understand when to optimize, how to use Z-ordering, and monitoring techniques.

**Real-World Application**: Focus on business scenarios rather than technical syntax memorization.

---

## 🚀 Next Steps

1. **Practice Daily**: Spend 30 minutes daily on hands-on exercises
2. **Build Projects**: Create complete end-to-end scenarios
3. **Join Community**: Participate in Databricks community forums
4. **Take Practice Tests**: Use official Databricks practice exams
5. **Schedule Exam**: Book your certification when confident

Remember: **The exam tests practical problem-solving, not memorization!**