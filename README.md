# SQL-Bicycle-Sales-and-Inventory-Analysis
## Introduction
This project analyzes sales, inventory, customer, and discount data for a bicycle manufacturer to optimize inventory, improve production efficiency, and drive revenue growth. It addresses key metrics like stock-to-sales ratios, customer retention, and discount impact using SQL techniques such as CTEs, window functions, and aggregations.
### Objectives
- Optimize Inventory and Production: Analyze stock levels, stock-to-sales ratios, and growth trends to improve inventory utilization and align production with demand.
- Enhance Sales and Customer Retention: Identify high-performing subcategories and regions, assess the impact of discounts, and analyze retention patterns to boost sales and loyalty.
### Dataset Used
- This project uses UniGap's bicycle manufacturing database. The key tables include:
- Data table :
    | Table Name   | Description                                    |
    |-----------------------------|-------------------------------------------------|
    | adventureworks2019.Sales.SalesOrderDetail|Individual products associated with a specific sales order.    |
    | adventureworks2019.Sales.SalesOrderHeader|General sales order information.  |
    | adventureworks2019.Production.Product   | Products sold or used in the manfacturing of sold products.|
    | adventureworks2019.Production.ProductSubcategory |  stores information about product subcategories. |
    |adventureworks2019.Production.WorkOrder |Manufacturing work orders. |
    |adventureworks2019.Sales.SpecialOffer   |Sale discounts lookup table. |
## Data Exploration
#### 1. Calculate total quantity sold, sales value, and order count for each subcategory in the last 12 months.
Problem solving:
- Filter the last 12 months --> date(ModifiedDate) between   (date_sub(date(ModifiedDate), INTERVAL 12 month)) and '2014-06-30'
- Sum of itemxQty, LineTotal, order quantity.
```sql
select format_date('%b %Y', m.ModifiedDate) as Period
  ,pp.name as name
  ,sum(m.OrderQty) as Qty
  ,sum(m.LineTotal) as LineTotal
  ,count(distinct m.SalesOrderID)as Orde_cnt
from `adventureworks2019.Sales.SalesOrderDetail` m
left join `adventureworks2019.Production.ProductTable` p on m.productID = p.productID
left join `adventureworks2019.Production.ProductSubcategory` pp on cast(p.ProductSubcategoryID AS int) = pp.ProductSubcategoryID
where date(m.ModifiedDate) between date_sub('2014-06-30', INTERVAL 12 month) and '2014-06-30'
group by period, Name
order by period desc, name
```
<details>
  <summary>Results</summary>
    
|Period |	name|	Qty| LineTotal | Orde_cnt |
|----------------|	--------------|	----------------| ----------------| ----------------|
|Sep 2013|	Bike Racks|	312|	22828.512000000002|	71|
|Sep 2013|	Bike Stands|	26|	4134.0|	26|
|Sep 2013|	Bottles and Cages|	803|	4676.56280799994|	380|
|Sep 2013|	Bottom Brackets|	60|	3118.1400000000008|	19|
|Sep 2013|	Brakes|	100|	6390.0|	29|
|Sep 2013|	Caps|	440|	2879.4826160000002|	203|
</details>

#### 2. Calculate YoY growth rates for quantity sold and rank the top 3 subcategories with the highest growth rates.
Problem solving:
- current quantity --> sum(itemxQty)
-  previus quantity--> lead function
- Quantity different by SubCategory by year --> current quantity / previus quantity
- Filter top 3 category with highest grow rate -->dense rank order by quantity different

**Techniques Used**: Window functions (LEAD) for previous year comparisons.
 ```sql
with sale_info as ( 
select format_date('%Y', m.ModifiedDate) as period
  ,pp.name as name
  ,sum(m.OrderQty) as qty_item
from `adventureworks2019.Sales.SalesOrderDetail` m
left join `adventureworks2019.Production.ProductTable` p on m.productID = p.productID
left join `adventureworks2019.Production.ProductSubcategory` pp on cast(p.ProductSubcategoryID AS int) = pp.ProductSubcategoryID
group by period,name
order by name asc, period desc
)
, sale_diff as (
  select period
    ,name
    ,qty_item
    ,lead(qty_item) over (partition by name order by period desc) as prz_qty
    ,round(qty_item / (lead(qty_item) over (partition by name order by period desc)) -1,2) as qty_diff
  from sale_info
  order by qty_diff desc 
)
, rank_qty_diff as (
  select period
    ,name
    ,qty_item
    ,prz_qty
    ,qty_diff
    ,dense_rank() over (order by qty_diff desc) as qty_rank
  from sale_diff 
)

select distinct name
    ,qty_item
    ,prz_qty
    ,qty_diff
    ,qty_rank
from rank_qty_diff 
where qty_rank <= 3
order by qty_rank ;
```
<details>
  <summary>Results</summary>
    
|name |	qty_item|	prz_qty| qty_diff | qty_rank |
|----------------|	--------------|	----------------| ----------------| ----------------|
|Mountain Frames|	3168|	510|	5.21|	1|
|Socks|	2724|	523|	4.21|	2|
|Road Frames|	5564|	1137|	3.89|	3|
</details>

#### 3.  Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number
**Problem solving**:
- calculate Order quantity of each TeritoryID --> count(distinct m.SalesOrderID)
- Ranking Top 3 TeritoryID, do not skip the rank number --> dense rank

```sql

with product_count as (
  select format_date('%Y', o.ModifiedDate) as _year
    ,h.territoryID as territoryID
    ,sum(o.OrderQty) as order_count
    from `adventureworks2019.Sales.SalesOrderDetail` o 
  left join `adventureworks2019.Sales.SalesOrderHeader` h
   on h.SalesOrderID = o.SalesOrderID
  group by _year, territoryID
  order by _year desc 
)
  ,rank_territory as (
  select _year
    ,territoryID
    ,order_count
    ,dense_rank() over (partition by _year order by order_count desc) as order_count_rank
  from product_count
  order by _year desc, order_count_rank
)

select _year
  ,territoryID
  ,order_count
  ,order_count_rank
from rank_territory
where order_count_rank in (1,2,3)
```
<details>
  <summary>Results</summary>
    
|_year |	territoryID|	order_count| order_count_rank |
|----------------|	--------------|	----------------| ----------------|
|2014|	4|	11632|	1|
|2014|	6|	9711|	2|
|2014|	1|	8823|	3|
|2013|	4|	26682|	1|
|2013|	6|	22553|	2|
|2013|	1|	17452|	3|
|2012|	4|	17553|	1|
|2012|	6|	14412|	2|
|2012|	1|	8537|	3|
</details>

#### 4. Average Pageviews by Purchaser Type (June, July 2017)
- Compares the average number of pageviews between purchasers and non-purchasers.
```sql
with purchaser as (
  select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
    ,round((sum(totals.pageviews) / count(distinct fullVisitorId)),8) as avg_pageviews_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
  where (totals.transactions >=1) 
    and (product.productRevenue is not null) 
    and (_table_suffix between '0601' and '0731')
  group by month
  order by month
)
, non_purchaser as (
  select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
    ,round((sum(totals.pageviews) / count(distinct fullVisitorId)),8) as avg_pageviews_non_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
  where (totals.transactions is null) 
    and (product.productRevenue is  null) 
    and (_table_suffix between '0601' and '0731')
  group by month
  order by month
)
select p.month
  ,p.avg_pageviews_purchase
  ,np.avg_pageviews_non_purchase
from purchaser p
join non_purchaser np
on p.month = np.month
order by month;
```
<details>
  <summary>Results</summary>
    
|month |	avg_pageviews_purchase|	avg_pageviews_non_purchase|
|----------------|	--------------|	----------------|
|201706|	94.02050114|	316.86558846|
|201707|	124.23755187|	334.0565598|
</details>

### 5. Average Transactions per User (July 2017)
- Computes the average number of transactions per user who made a purchase in July 2017.
```sql
select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
  ,round((sum(totals.transactions) / count(distinct fullVisitorId)),9) as Avg_total_transactions_per_user
from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
where (totals.transactions >=1) 
  and (product.productRevenue is not null) 
  and (_table_suffix between '0701' and '0731')
group by month;
```
<details>
  <summary>Results</summary>
    
|month |	Avg_total_transactions_per_user|
|----------------|	--------------|
|201707|	4.163900415|
</details>

### 6. Average Revenue per Session (July 2017)
- Calculates the average revenue generated per session, considering only purchasing users.
```sql
select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
  ,round((SUM((product.productRevenue) / 1000000) / count(totals.visits)),2) as avg_revenue_by_user_per_visit
 from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits,
UNNEST (hits.product) product 
where (totals.transactions is not null) 
  and (product.productRevenue is not null)
  and  (_table_suffix between '0701' and '0731')
group by month;
```
<details>
  <summary>Results</summary>
    
|month |	avg_revenue_by_user_per_visit|
|----------------|	--------------|
|201707|	43.86|
</details>

### 7. Other Products Purchased by Customers Who Bought "YouTube Men's Vintage Henley" (July 2017)
- Identifies other products bought by customers who purchased the "YouTube Men's Vintage Henley" product.
```sql
with users_buy_the_product as (
  select  fullVisitorId
  from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product 
  where product.v2ProductName = "YouTube Men's Vintage Henley"
    and (_table_suffix between '0701' and '0731')
    and product.productRevenue is not null
  )

select product.v2ProductName as other_purchased_products
  ,sum(product.productQuantity) as quantity
from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
UNNEST (hits) hits,
UNNEST (hits.product) product 
where fullVisitorId in (select fullVisitorId from users_buy_the_product)
  and product.v2ProductName != "YouTube Men's Vintage Henley" 
  and (_table_suffix between '0701' and '0731') 
  and product.productRevenue is not null
group by other_purchased_products
order by quantity desc;
```
<details>
  <summary>Results</summary>
    
|other_purchased_products |	quantity|
|----------------|	--------------|
|Google Sunglasses|	20|
|Google Women's Vintage Hero Tee Black |	7|
|SPF-15 Slim & Slender Lip Balm |	6|
|Google Women's Short Sleeve Hero Tee Red Heather	|4|
|Google Men's Short Sleeve Badge Tee Charcoal |	3|
</details>

### 8. Cohort Analysis: Product View to Add to Cart to Purchase (Jan, Feb, Mar 2017)
- Computes the conversion rates from product views to cart additions and purchases at the product level.
```sql
with month_view_data as (
  select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
        ,count(hits.eCommerceAction.action_type) as num_product_view
  from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits
  where hits.eCommerceAction.action_type = "2"
  group by month
  order by month
)
  , addtocart_data as (
  select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
        ,count(hits.eCommerceAction.action_type) as num_addtocart
  from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits
  where hits.eCommerceAction.action_type = "3" 
  group by month
  order by month
)
, purchase_data as (
  select FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) as month
      ,count(hits.eCommerceAction.action_type) as num_purchase
  from `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product 
  where hits.eCommerceAction.action_type = "6" 
    and product.productRevenue is not null
  group by month
  order by month
)

select m. month
  ,m.num_product_view
  ,a.num_addtocart
  ,p.num_purchase
  ,round(100*(a.num_addtocart / m.num_product_view ),2) as add_to_cart_rate
  ,round(100*(p.num_purchase / m.num_product_view),2) as purchase_rate
from  month_view_data m
left join addtocart_data a 
on m.month = a.month
left join purchase_data p   
on m.month = p.month 
order by month;
```
<details>
  <summary>Results</summary>
    
|month |	num_product_view|	num_addtocart| num_purchase | add_to_cart_rate | purchase_rate |
|----------------|	--------------|	----------------| ----------------| ----------------| ----------------|
|201701|	25787|	7342|	2143|	28.47|	8.31|
|201702|    21489|	7360|	2060|	34.25|	9.59|
|201703|	23549|	8782|	2977|	37.29|	12.64|
|201704|	24587|	10291|	2906|	41.86|	11.82|
|201705|	25469|	10083|	3285|	39.59|	12.9|
|201706|	22148|	9020|    2785|	40.73|	12.57|
|201707|	28576|	11860|	3669|	41.5|    12.84|
|201708|	1267|	494|	186|	38.99|	14.68|
</details>

## Insights and Recommendations
### 1️. Optimize Traffic and Conversion Rates
#### Insights:
- Traffic volume fluctuates, but transactions do not always scale proportionally.
- High bounce rates from certain sources (e.g., YouTube) suggest engagement issues.
#### Recommendations:
- Invest in high-converting traffic sources while optimizing underperforming ones.
- Improve landing pages and reduce friction for high-bounce traffic to increase engagement.

### 2️. Revenue Optimization by Traffic Source
#### Insights:
- Direct traffic generates the highest revenue, while other sources contribute less.
- Weekly revenue fluctuations indicate possible marketing inefficiencies.
#### Recommendations:
- Allocate budget to the highest revenue-generating traffic sources.
- Implement time-based marketing campaigns to smooth out revenue inconsistencies.

### 3️. Enhancing the Purchase Journey
#### Insights:
- A significant drop-off occurs between product views, add-to-cart, and purchase.
- Purchasers browse fewer pages than non-purchasers, indicating decision-making hesitation.
#### Recommendations:
- Streamline checkout, offer incentives (e.g., discounts, free shipping) to encourage purchases.
- Use urgency tactics (limited stock, countdown timers) to increase add-to-cart conversions.

### 4️. Maximizing Order Value and Repeat Purchases
#### Insights:
- Customers who buy "YouTube Men's Vintage Henley" often purchase complementary products.
- Purchasers have a higher average transaction per user, indicating strong retention potential.
#### Recommendations:
- Introduce product bundles and targeted upselling strategies.
- Launch loyalty programs to retain high-value customers and drive repeat purchases.

### 5️. Data-Driven Decision Making for Long-Term Growth
#### Insights:
- Monthly and weekly trends reveal opportunities for proactive strategy adjustments.
- Revenue per session highlights the efficiency of the purchase funnel.
#### Recommendations:
- Monitor traffic source performance and adjust acquisition strategies accordingly.
- Continuously test and optimize pricing, UX, and promotional strategies to maximize revenue.
