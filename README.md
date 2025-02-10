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
- Purrent quantity --> sum(itemxQty)
- Previus quantity--> lead function
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
- Calculate Order quantity of each TeritoryID --> count(distinct m.SalesOrderID)
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

#### 4. Calculate the total cost of discounts belonging to seasonal promotions for each subcategory.
**Problem solving**:
- Calculate cost of discount --> (Discount Cost = Disct Pct * Unit Price * Item Qty)
- Filter type 'seasonal discount'
```sql
select format_date('%Y', ModifiedDate) as _year
  ,name
  ,round(sum(discount_cost),2) as total_discount_cost
from (
    select distinct o.ModifiedDate
    ,pp.name
    ,s.DiscountPct
    ,s.Type
    ,s.DiscountPct * o.UnitPrice * o.OrderQty as discount_cost
  from `adventureworks2019.Sales.SalesOrderDetail` o 
  left join `adventureworks2019.Production.ProductTable` p on o.productID = p.productID
  left join `adventureworks2019.Production.ProductSubcategory` pp on cast(p.ProductSubcategoryID AS int) = pp.ProductSubcategoryID
  left join `adventureworks2019.Sales.SpecialOffer` s on s.SpecialOfferID = o.SpecialOfferID
  where lower(Type) like '%seasonal discount%'
  order by ModifiedDate
)
group by _year,name
```
<details>
  <summary>Results</summary>
    
|_year |	name|	total_discount_cost|
|----------------|	--------------|	----------------|
|2012|	Helmets|	149.72|
|2013|	Helmets|	543.22|
</details>

### 5. Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)
**Problem solving**:
- Filter Successfully Shipped orders (Status = 5) in 2014 from the SalesOrderHeader table.
- Use ROW_NUMBER() to identify each customer's first order month (month_join).
- Calculate the difference between the first order month and subsequent months (month_diff).
- Group data by month_join and month_diff to count distinct returning customers.
- Analyze retention patterns over time for each customer cohort
```sql
with info as (
  select extract(month from ModifiedDate) as month_order
    ,extract (year from ModifiedDate) as year
    ,CustomerID
    ,count(distinct SalesOrderID) as sales_count
  from `adventureworks2019.Sales.SalesOrderHeader`
  where Status = 5 and extract (year from ModifiedDate) =2014
  group by month_order,year,CustomerID
)
, row_num as (
  select *
    ,row_number() over (partition by CustomerID order by month_order asc) row_nb
  from info
)
, first_order as (
  select distinct month_order as month_join
    ,year
    ,CustomerID
  from row_num
  where row_nb = 1
)
, all_join as (
  select distinct i.month_order
    ,i.year
    ,i.CustomerID
    ,f.month_join
    ,concat('M-',i.month_order - f.month_join) as month_diff
  from info i
  left join first_order f on i.CustomerID = f.CustomerID
  order by month_diff
)

select distinct month_join
  ,month_diff
  ,count(distinct CustomerID) as customer_count
from all_join
group by month_join, month_diff
order by month_join
```
<details>
  <summary>Results</summary>
    
|month_join |	month_diff| customer_count |
|----------------|	--------------| --------------|
|1|	M-0|	2076|
|1|	M-1|	78|
|1|	M-2|	89|
|1|	M-3|	252|
|1|	M-4|	96|
|1|	M-5|	61|
|1|	M-6|	18|
|2|	M-0|	1805|
|2|	M-2|	61|
|2|	M-3|	234|
</details>

### 6. Analyze month-over-month (MoM) stock levels and percentage changes for all products in 2011.
**Problem solving**:
- current stock quantity --> sum(StockedQty), in 2011
- previous stock quantity --> lead
- stock_level_diff --> (current stock quantity -  previous stock quantity) /  previous stock quantity
-  If %gr rate is null then 0 -- case when
```sql
with current_stock as (
select p.Name
  ,format_date('%m', w.ModifiedDate)  as month 
  ,format_date('%Y', w.ModifiedDate)  as year
  ,sum(w.StockedQty) as stock_qty
from adventureworks2019.Production.Product p
left join adventureworks2019.Production.WorkOrder w 
  on p.ProductID = w.ProductID
where format_date('%Y', w.ModifiedDate) = '2011'
group by name, month, year
order by stock_qty desc
)
, add_prv_stock as(
  select Name
  ,month
  ,year
  ,stock_qty
  ,lag(stock_qty) over (partition by Name order by month) as prz_stock
  from current_stock
  order by name, month desc
)

  select name
    ,month
    ,year
    ,stock_qty
    ,prz_stock
    ,case 
        when round((100*(stock_qty-prz_stock)/prz_stock),1) is null then 0
        else round((100*(stock_qty-prz_stock)/prz_stock),1) 
        end as stock_diff
  from add_prv_stock
  order by name, month desc
```
<details>
  <summary>Results</summary>
    
|name |	month| year| stock_qty| prz_stock| stock_diff|
|----------------|	--------------| --------------| --------------| --------------| --------------|
|BB Ball Bearing|	12|	2011|	8475|	14544|	-41.7|
|BB Ball Bearing|	11|	2011|	14544|	19175|	-24.2|
|BB Ball Bearing|	10|	2011|	19175|	8845|	116.8|
|BB Ball Bearing|	09|	2011|	8845|	9666|    -8.5|
|BB Ball Bearing|	08|	2011|	9666|	12837|	-24.7|
|BB Ball Bearing|	07|	2011|	12837|	5259|	144.1|
|BB Ball Bearing|	06|	2011|	5259|	null|	0.0|
|Blade|	12|	2011|	1842|	3598|	-48.8|
|Blade|	11|	2011|	3598|	4670|	-23.0|
|Blade|	10|	2011|	4670|	2122|	120.1|
</details>

### 7. Calculate the ratio of stock-to-sales for each product, by month, in 2011.
**Problem solving**:
- sum(sales and stock - sum(OrderQty), sum(StockedQty)
- ratio of Stock --> sum(StockedQty) /  sum(OrderQty)
- filter in 2011
```sql
with sale_info as (
  select extract (month from o.ModifiedDate) as month
    ,extract (year from o.ModifiedDate) as year
    ,p.ProductID
    ,p.name
    ,sum(o.OrderQty) as sales
  from`adventureworks2019.Sales.SalesOrderDetail` o
  left join `adventureworks2019.Production.Product` p
    on o.ProductID = p.ProductID
    where format_timestamp("%Y", o.ModifiedDate) = '2011'
  group by 1,2,3,4
)
, stock_info as(
  select extract (month from ModifiedDate) as month
    ,extract (year from ModifiedDate) as year
    ,ProductID
    ,sum(StockedQty) as stock
  from `adventureworks2019.Production.WorkOrder`
  where format_timestamp("%Y", ModifiedDate) = '2011'
  group by 1,2,3
)

select s.month
  ,s.year
  ,s.ProductID
  ,s.name
  ,s.sales
  ,ss.stock
  ,round(coalesce(ss.stock,0) / s.sales, 2) AS ratio
from sale_info s
 full join stock_info ss
on s.ProductID = ss.ProductID
and s.month = ss.month 
and s.year = ss.year
order by month desc, ratio desc
```
<details>
  <summary>Results</summary>
    
|month |	year|	ProductID| name | sales | stock | ratio |
|----------------|	--------------|	----------------| ----------------| ----------------| ----------------|  ----------------|
|12|	2011|	745|	HL Mountain Frame - Black, 48|	1|	27|	27.0|
|12|	2011|	743|	HL Mountain Frame - Black, 42|	1|	26|	26.0|
|12|	2011|	748|	HL Mountain Frame - Silver, 38|	2|	32|	16.0|
|12|	2011|	722|	LL Road Frame - Black, 58|	4|	47|	11.75|
|12|	2011|	747|	HL Mountain Frame - Black, 38|	3|	31|	10.33|
|12|	2011|	726|	LL Road Frame - Red, 48|	5|	36|	7.2|
|12|	2011|	738|	LL Road Frame - Black, 52|	10|	64|	6.4|
|12|	2011|	730|	LL Road Frame - Red, 62|	7|	38|	5.43|
|12|	2011|	741|	HL Mountain Frame - Silver, 48|	5|	27|	5.4|
|12|	2011|	725|	LL Road Frame - Red, 44|	12|	53|	4.42|
</details>

### 8. Calculate the number of pending orders and their total value in 2014.
**Problem solving**:
- sum(PurchaseOrderID) --> count(distinct PurchaseOrderID)
- sum(total sales) --> sum(TotalDue)
- status = 1(pending)
```sql
select extract(year from ModifiedDate) as _year
  ,status
  ,count(distinct PurchaseOrderID) as order_count
  ,sum(TotalDue) as value
from adventureworks2019.Purchasing.PurchaseOrderHeader
where status = 1 
  and extract(year from ModifiedDate) = 2014
group by _year, status
```
<details>
  <summary>Results</summary>
    
|_year |	status|	order_count| value | 
|----------------|	--------------|	----------------| ----------------| 
|2014|	1|	224|	3873579.0123000029|	
</details>

## Insights and Recommendations
### 1️. Sales Performance and Growth:
- Insights: Subcategories like Mountain Frames (5.21%), Socks (4.21%), and Road Frames (3.89%) showed the highest YoY growth.
- Recommendations: Prioritize production and marketing efforts for high-growth subcategories to maximize revenue.
### 2️. Inventory Management
- Insights: Products like BB Ball Bearing experienced significant MoM stock fluctuations (e.g., -41.7% in December 2011).
- Recommendations: Implement a dynamic inventory management system to stabilize stock levels and avoid overstock or stockouts.
### 3️. Stock-to-Sales Efficiency
- Insights: Products such as HL Mountain Frame (27.0 stock-to-sales ratio) indicate potential overstock issues.
- Recommendations: Adjust inventory levels based on sales trends to reduce holding costs and improve cash flow.
### 4️. Territory Performance
- Insights: Territory ID 4 consistently ranked as the highest-performing region (e.g., 11,632 orders in 2014).
- Recommendations: Focus marketing campaigns and resource allocation on top-performing territories to boost overall sales.
### 5️. Seasonal Discounts
- Insights: Subcategories like Helmets had significant seasonal discount costs (e.g., $543 in 2013).
- Recommendations: Optimize discount strategies by analyzing ROI for each subcategory and focusing on high-margin items.
### 6. Customer Retention
- Insights: Cohort analysis revealed retention drops significantly after the first month (e.g., M-0: 2,076 customers vs. M-1: 78).
- Recommendations: Implement loyalty programs and personalized offers to retain customers beyond their first purchase.
### 7. Pending Orders
- Insights: In 2014, 224 pending orders accounted for over $3.87M in value.
- Recommendations: Address operational bottlenecks to expedite pending orders and prevent delays in revenue recognition.
