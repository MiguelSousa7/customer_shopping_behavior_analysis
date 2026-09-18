# 🛍️ Customer Shopping Behavior Analysis

> An end-to-end data analytics project simulating a corporate BI workflow, from raw data ingestion to interactive dashboarding, using Python, SQL and Power BI.

---

## 📌 Project Overview

This project analyzes customer shopping behavior using transactional data from **3,900 purchases** across multiple product categories. The goal is to uncover actionable insights on spending patterns, customer segmentation, product preferences, and subscription behavior to support data-driven business decisions.

The pipeline covers three distinct layers, replicating the typical responsibilities of a Data Analyst in a business environment:

| Layer | Tool | Description |
|---|---|---|
| 🐍 ETL & EDA | Python · pandas | Data cleaning, transformation and feature engineering |
| 🗄️ Business Analysis | PostgreSQL | 10 structured queries to answer key business questions |
| 📊 Visualization | Power BI | Interactive dashboard with KPIs and dynamic filters |

---

## 🔧 Data Pipeline

### 1 · ETL & Exploratory Data Analysis - Python

The notebook `initial_eta_eda.ipynb` implements the following steps:

**Column normalization**
```python
df.columns = df.columns.str.lower().str.replace(" ", "_")
df = df.rename(columns={"purchase_amount_(usd)": "purchase_amount"})
```

**Missing data imputation**
```python
# Impute review_rating with the median per product category
df["review_rating"] = df.groupby("category")["review_rating"] \
    .transform(lambda x: x.fillna(x.median()))
```
> Strategy: category-level median preserves the distribution and avoids global bias.

**Feature engineering**
```python
# Age segmentation into 4 equal-frequency groups
labels = ["Young Adult", "Adult", "Middle-aged", "Senior"]
df["age_group"] = pd.qcut(df["age"], q=4, labels=labels)

# Convert purchase frequency text to numeric (days)
freq_mapping = {"Weekly": 7, "Fortnightly": 14, "Monthly": 30,
                "Quarterly": 90, "Every 3 Months": 90, "Annually": 365}
df["purchase_frequency_days"] = df["frequency_of_purchases"].map(freq_mapping)
```

**Redundancy check**
```python
# discount_applied and promo_code_used are 100% identical → drop one
(df["discount_applied"] == df["promo_code_used"]).all()  # True
df = df.drop("promo_code_used", axis=1)
```

**PostgreSQL integration**
```python
from sqlalchemy import create_engine
engine = create_engine("postgresql+psycopg2://user:pass@localhost:5432/customer_behavior")
df.to_sql("customer", engine, if_exists="replace", index=False)
```

---

### 2 · Business Analysis - SQL (PostgreSQL)

10 queries structured around real business questions:

| # | Question |
|---|---|
| Q1 | What is the total revenue by gender? |
| Q2 | Are repeat buyers (>5 purchases) more likely to subscribe? |
| Q3 | What is the revenue contribution of each age group? |
| Q4 | Which customers used a discount but still spent above average? |
| Q5 | What are the top 5 products by average review rating? |
| Q6 | How do average purchase amounts compare between Standard and Express shipping? |
| Q7 | Do subscribers spend more? Average spend and total revenue comparison. |
| Q8 | Which 5 products have the highest discount dependency rate? |
| Q9 | Customer segmentation: New / Returning / Loyal |
| Q10 | Top 3 most purchased products per category (window function) |

**Example - Window function (Q10):**
```sql
WITH item_counts AS (
    SELECT category, item_purchased,
           COUNT(customer_id) AS total_orders,
           ROW_NUMBER() OVER (
               PARTITION BY category
               ORDER BY COUNT(customer_id) DESC
           ) AS item_rank
    FROM customer
    GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```

**Example — CTE + CASE for segmentation (Q9):**
```sql
WITH customer_type AS (
    SELECT customer_id, previous_purchases,
           CASE
               WHEN previous_purchases = 1 THEN 'New'
               WHEN previous_purchases BETWEEN 2 AND 10 THEN 'Returning'
               ELSE 'Loyal'
           END AS customer_group
    FROM customer
)
SELECT customer_group, COUNT(*) AS num_customers
FROM customer_type
GROUP BY customer_group;
```

---

### 3 · Dashboard - Power BI

An interactive dashboard built to communicate insights visually to stakeholders, featuring:

- **KPI cards** — Total customers (3.9K), Average Purchase Amount ($59.76), Average Review Rating (3.75)
- **Donut chart** — Subscription status breakdown (27% Yes / 73% No)
- **Bar charts** — Revenue and sales volume by category and age group
- **Slicers** — Filter by Subscription Status, Gender, Category, Shipping Type

---

## 📈 Key Findings

| Insight | Finding |
|---|---|
| 💰 Revenue by gender | Male customers generated $157,890 vs. $75,191 from female customers |
| 🎯 High-value discount users | 839 customers used discounts and still spent above average |
| 📋 Subscription gap | 73% of customers are non-subscribers — significant growth opportunity |
| 👶 Top revenue age group | Young Adults lead with $62,143, followed by Middle-aged ($59,197) |
| 🚚 Shipping & spend | Express shipping users spend slightly more on average ($60.48 vs $58.46) |
| 🏆 Customer loyalty | 80% of customers (3,116) classified as "Loyal" based on purchase history |
| ⭐ Top-rated products | Gloves (3.86), Sandals (3.84), Boots (3.82) |
| 🏷️ Discount-heavy products | Hat (50%), Sneakers (49.66%), Coat (49.07%) discount rates |

---

## 💡 Business Recommendations

1. **Boost subscriptions** - With 73% non-subscribers, targeted campaigns with exclusive perks could drive significant conversion.
2. **Loyalty programs** - Incentivize "Returning" customers to cross into the "Loyal" segment, which already represents 80% of the base.
3. **Review discount policy** - Products like Hat, Sneakers, and Coat show ~50% discount rates; assess whether margins justify this dependency.
4. **Product positioning** - Highlight top-rated products (Gloves, Sandals, Boots) in marketing campaigns to leverage social proof.
5. **Targeted marketing** - Focus acquisition spend on Young Adults and Express Shipping users, both associated with higher revenue.

---

*Project developed as a demonstration of end-to-end data analytics capabilities.*
