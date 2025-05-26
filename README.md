# UK Online Retail Store – Customer Cohort Analysis

An end-to-end cohort analysis case study for a UK-based online gift retailer. We explore customer retention, revenue progression, and lifetime value across monthly cohorts to drive data-backed acquisition and retention strategies.

---

## Table of Contents

1. [Business Problem & Context](#business-problem--context)  
2. [Data Collection & Preparation](#data-collection--preparation)  
3. [Methodology & Implementation](#methodology--implementation)  
4. [Tableau Visualization Process](#tableau-visualization-process)  
5. [Business Insights & Recommendations](#business-insights--recommendations)  
6. [Challenges & Learnings](#challenges--learnings)  
7. [Repository Structure](#repository-structure)  
8. [How to Reproduce](#how-to-reproduce)  

---

## 1. Business Problem & Context

The UK-based online retailer specializes in unique, all-occasion gifts, with a significant portion of its customer base comprised of wholesale buyers. Over the period December 2010 to December 2011, the company faced:

- **High churn after first purchase**  
- **Variable revenue streams** across different customer cohorts  
- **Seasonal fluctuations** impacting customer acquisition and repeat buying  
- **Unclear lifetime value** and retention benchmarks  

**Objective:**  
Perform a cohort analysis to quantify retention and revenue progression by month of first purchase, uncover seasonal impacts, compute Customer Lifetime Value (CLV), and recommend strategies to improve acquisition and retention.

---

## 2. Data Collection & Preparation

### Source Dataset  
- **File:** `Online Retail.xlsx`  
- **Source:** UCI Machine Learning Repository  
- **Coverage:** December 2010 – December 2011, ~540,000 transaction records

### Initial Data Cleaning (Python / pandas)  
```python
import pandas as pd

# Load the Excel file
df = pd.read_excel('data/Online Retail.xlsx')

# 1. Remove exact duplicate rows
df.drop_duplicates(inplace=True)

# 2. Filter out invalid transactions
df = df[
    (df.Quantity > 0) & 
    (df.UnitPrice > 0) & 
    df.CustomerID.notnull()
]

# 3. Reset index after filtering
df.reset_index(drop=True, inplace=True)
```
### Feature engineering
##### 1. Compute total transaction amount
df['TotalAmount'] = df['Quantity'] * df['UnitPrice']

##### 2. Convert InvoiceDate to datetime and extract cohort period
df['InvoiceDate'] = pd.to_datetime(df['InvoiceDate'])
df['CohortMonth']  = df['InvoiceDate'].dt.to_period('M')

##### 3. Identify each customer’s first purchase date & cohort
df['FirstPurchaseDate'] = df.groupby('CustomerID')['InvoiceDate'].transform('min')
df['FirstCohortMonth']  = df['FirstPurchaseDate'].dt.to_period('M')

##### 4. Calculate months since first purchase for each transaction
df['MonthsFromFirstPurchase'] = (
    (df['InvoiceDate'].dt.year  - df['FirstPurchaseDate'].dt.year)  * 12 +
    (df['InvoiceDate'].dt.month - df['FirstPurchaseDate'].dt.month)
)


## 3. Methodology & Implementation

1. **Cohort Assignment**  
   - Determine each customer’s first purchase month (`FirstCohortMonth`) using:  
     ```python
     df['FirstPurchaseDate'] = df.groupby('CustomerID').InvoiceDate.transform('min')
     df['FirstCohortMonth']   = df['FirstPurchaseDate'].dt.to_period('M')
     ```
   - Assign every transaction to its customer’s cohort.

2. **Retention Matrix**  
   - Compute `MonthsFromFirstPurchase` for each record:  
     ```python
     df['MonthsFromFirstPurchase'] = (
         (df.InvoiceDate.dt.year  - df.FirstPurchaseDate.dt.year) * 12 +
         (df.InvoiceDate.dt.month - df.FirstPurchaseDate.dt.month)
     )
     ```
   - Pivot to create a retention table:  
     ```python
     retention = df.pivot_table(
         index='FirstCohortMonth',
         columns='MonthsFromFirstPurchase',
         values='CustomerID',
         aggfunc='nunique'
     )
     cohort_sizes = retention.iloc[:, 0]
     retention_rate = retention.divide(cohort_sizes, axis=0).round(3)
     ```

3. **Revenue & CLV Calculation**  
   - Aggregate total revenue per cohort-month:  
     ```python
     revenue = df.pivot_table(
         index='FirstCohortMonth',
         columns='MonthsFromFirstPurchase',
         values='TotalAmount',
         aggfunc='sum'
     )
     ```
   - Compute per-customer CLV and cumulative CLV:  
     ```python
     clv_per_customer = revenue.divide(cohort_sizes, axis=0)
     cumulative_clv   = clv_per_customer.cumsum(axis=1)
     ```

4. **Seasonality Comparison**  
   - Tag cohorts as **Winter** (Dec–Feb) vs. **Summer** (Jun–Aug):  
     ```python
     winter_cohorts = retention_rate.loc[retention_rate.index.month.isin([12,1,2])]
     summer_cohorts = retention_rate.loc[retention_rate.index.month.isin([6,7,8])]
     ```
   - Compare average retention and CLV between seasons.

5. **Export for Visualization**  
   - Save `retention_rate`, `revenue`, and `cumulative_clv` to CSV for Tableau:  
     ```python
     retention_rate.to_csv('output/cohort_retention.csv')
     revenue.to_csv('output/cohort_revenue.csv')
     cumulative_clv.to_csv('output/cohort_clv.csv')
     ```
   - Ensure output folder structure matches `/output/` directory in repo.




## 4. Tableau Visualization Process

- **Data Sources**  
  - `output/cohort_retention.csv` — retention rate per cohort by month  
  - `output/cohort_revenue.csv` — monthly revenue per cohort  
  - `output/cohort_clv.csv` — cumulative CLV per cohort  

- **Dashboard Layout**  
  1. **Cohort Retention Heatmap**  
     - Rows: `FirstCohortMonth`   
     - Columns: `MonthsFromFirstPurchase`   
     - Color scale: retention rate (green = high, red = low)  
  2. **Revenue Trend Lines**  
     - One line per cohort showing monthly revenue  
     - Dual axis to overlay cumulative CLV  
  3. **CLV Bar Chart**  
     - Bar chart of 12-month CLV by cohort  
  4. **Seasonality Filter**  
     - Parameter control to toggle Winter vs. Summer cohorts  

- **Key Tableau Features Used**  
  - **Calculated Fields** for retention percentage:  
    ```tableau
    Cohort Size = { FIXED [FirstCohortMonth] : COUNTD([CustomerID]) }
    Retention % = COUNTD([CustomerID]) / [Cohort Size]
    ```  
  - **Level-of-Detail (LOD) Expressions** to fix calculations at the cohort level  
  - **Dual-Axis Charts** for overlaying revenue and CLV curves  
  - **Parameter Controls** to switch views between seasonal cohort groups  
  - **Interactive Tooltips** displaying detailed metrics (e.g., cohort size, revenue, CLV)  



## 5. Business Insights & Recommendations

### Insights

- **Seasonal Acquisition Peaks**  
  - The **December 2010 cohort** achieved **37%** retention after the first month and **£570K** in initial revenue.  
  - **Winter cohorts** (Dec–Feb) consistently outperform **summer cohorts** (Jun–Aug) in both retention rates and cumulative revenue.

- **Retention Drop-Off**  
  - Retention rates drop sharply after Month 1, falling below **20%** by Month 2 and under **10%** by Month 6 for most cohorts.  
  - Early 2011 cohorts show slightly better Month 2–4 retention, but still experience rapid decline.

- **Customer Lifetime Value (CLV)**  
  - The **December 2010 cohort** reached a cumulative CLV of **£4.5M** over 12 months.  
  - Average per-customer CLV ranges between **£400–£700** across cohorts.

- **Revenue Patterns**  
  - Initial revenue spikes are followed by steep declines, indicating one-off purchases are common.  
  - Later cohorts display more sustained revenue in Months 2–4, suggesting improved repeat behavior.

### Recommendations

1. **Winter-Focused Marketing**  
   - Allocate a higher marketing budget to December–February campaigns.  
   - Introduce season-specific promotions and gift bundles to capitalize on high acquisition periods.

2. **Strengthen Early Retention**  
   - Deploy a **30-day onboarding** drip email series featuring discounts, usage tips, and loyalty program invitations.  
   - Offer time-limited incentives (e.g., free shipping on second purchase) to encourage repeat orders.

3. **Loyalty & Rewards Program**  
   - Implement tiered loyalty rewards for high-value customers from early cohorts to foster long-term engagement.  
   - Provide exclusive early access to new products or seasonal collections.

4. **Summer Engagement Strategies**  
   - Launch targeted promotions during slower summer months to mitigate seasonal drop-off.  
   - Test referral incentives for existing customers to bring in new buyers.

5. **Monitor & Iterate**  
   - Track cohort performance monthly to evaluate the effectiveness of interventions.  
   - Use A/B testing on promotional offers to determine optimal incentive structures.

By focusing on these strategies, the business can improve customer retention, optimize revenue streams, and enhance overall Customer Lifetime Value.  


## 6. Challenges & Learnings

- **Data Quality & Cleaning**  
  - Encountered **duplicate** and **invalid transactions** (negative quantities, missing `CustomerID`) requiring robust filtering.  
  - Standardizing **date formats** across records was essential for accurate cohort assignment.

- **Cohort Logic Complexity**  
  - Ensuring correct determination of each customer’s **first purchase month** involved iterative testing of `groupby` and transformation logic.  
  - Calculating **MonthsFromFirstPurchase** accurately across year boundaries demanded careful date arithmetic.

- **Analytical Computations**  
  - Constructing the **retention matrix** and computing **per-customer CLV** required multiple pivot and normalization steps.  
  - Managing large pivot tables in memory tested performance optimizations in pandas.

- **Tableau Visualization Challenges**  
  - Crafting **LOD expressions** for fixed-level calculations took several iterations to ensure correct aggregation.  
  - Designing an intuitive **heatmap color scale** and **dual-axis charts** for revenue vs. CLV required fine-tuning for clarity.

- **Stakeholder Communication**  
  - Translating technical findings into **actionable business strategies** demanded concise storytelling.  
  - Balancing **detail** with **simplicity** in the dashboard layout was key to stakeholder buy-in.

- **Key Learnings**  
  - Importance of **data validation** early in the workflow to prevent downstream errors.  
  - Value of **iterative development**: building and testing small components (e.g., retention logic, individual charts) before integrating.  
  - Effectiveness of **cohort analysis** in revealing both **seasonal trends** and **long-term customer value** for strategic decision-making.






