## Solution
### A. Customer Nodes Exploration
#### 1. How many unique nodes are there on the Data Bank system?
```sql
select count(distinct(node_id)) from customer_nodes
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/1f2cdbb5-e0a3-4d9b-8d73-06b7d0f7f4d4)

#### 2. What is the number of nodes per region?
```sql
select count(distinct(order_id)) as Amount_unique_customer_ordered from #customer_orders
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/63fb6f06-298d-4cb1-9a70-139af388dbc5)

#### 3. How many customers are allocated to each region?
```sql
select r.region_id,r.region_name, count(distinct(c.customer_id)) as number_of_cus from customer_nodes as c join regions as r
on c.region_id = r.region_id
group by r.region_id,r.region_name
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/eb025219-bf8c-43d2-b4b8-1e25f7bd34d8)


#### 4. How many days on average are customers reallocated to a different node?
- DATEDIFF : Returns the difference between two time values based on the specified time period. The two time values must be dates or date and time expressions.
```sql
select AVG(datediff(day,start_date, end_date)) as days_on_average from customer_nodes
where end_date != '9999-12-31'
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/4f90f9d7-c1c0-48f9-9fde-19f260051ddc)

#### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
```sql
WITH CTE AS (SELECT region_id,
                    DATEDIFF(day,start_date,end_date) as allocation_days
            FROM customer_nodes
            WHERE end_date != '9999-12-31'
            )

SELECT distinct region_id , 
        PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY allocation_days) OVER (PARTITION BY region_id) AS median,
        PERCENTILE_DISC(0.8) WITHIN GROUP (ORDER BY allocation_days) OVER (PARTITION BY region_id) AS #80th_percentile,
        PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY allocation_days) OVER (PARTITION BY region_id) AS #95TH_percentile
FROM CTE
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/7a8137f9-78d5-4ddf-aeaa-4c2819e3b11c)


### B. Customer Transactions
#### 1. What is the unique count and total amount for each transaction type?
```sql
select txn_type,count(*) as unique_count, sum(txn_amount) as total_amount from customer_transactions
group by txn_type
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/532b6b0b-55d0-4290-adf0-49cd569036ca)


#### 2. What is the average total historical deposit counts and amounts for all customers?
```sql
WITH CTE AS (
SELECT  customer_id,
COUNT(customer_id) as count_deposit, 
AVG(txn_amount) as amount_deposit
FROM customer_transactions 
WHERE txn_type = 'deposit'
GROUP BY customer_id
 )   
SELECT AVG(count_deposit) AS avg_count, AVG(amount_deposit) AS avg_amount FROM CTE
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/be6b8547-2d88-4326-9a7d-1dff3a75a8f0)

#### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
- DATEPART : Returns a time value of the passed argument, which can be a day, month, year, quarter, hour, minute, second, millisecond... The return value is an integer (int).
```sql
with cte as (
select customer_id,  
DATEPART(month,txn_date) as monthh,
sum(case when txn_type = 'deposit' then 1 else 0 end) as count_deposit,
sum(case when txn_type = 'purchase' then 1 else 0 end) as count_purchase,
sum(case when txn_type = 'withdrawal' then 1 else 0 end) as count_withdrawal
from customer_transactions
group by customer_id, DATEPART(month,txn_date)
)
select monthh, count(*) as amount_customer from cte
where count_deposit > 1 and (count_purchase > 1 or count_withdrawal > 1) 
group by monthh
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/269cee21-c7a0-4678-9c9d-f5b535a80a34)


#### 4. What is the closing balance for each customer at the end of the month?
- DATEPART : Returns a time value of the passed argument, which can be a day, month, year, quarter, hour, minute, second, millisecond... The return value is an integer (int).
```sql
with 
cte1 as (
select customer_id,  
DATEPART(month,txn_date) as monthh,
sum(case when txn_type = 'deposit' then txn_amount else 0 end) as count_deposit,
sum(case when txn_type = 'purchase' then -txn_amount else 0 end) as count_purchase,
sum(case when txn_type = 'withdrawal' then -txn_amount else 0 end) as count_withdrawal
from customer_transactions
group by customer_id, DATEPART(month,txn_date)

),
cte2 as (
select customer_id, monthh,count_deposit + count_purchase + count_withdrawal as balance 
from cte1
),
cte3 as (
select customer_id, monthh, balance , 
lag(balance,1) over (partition by customer_id order by customer_id) as pre_balance
from cte2
)

select customer_id, monthh, balance , 
case when pre_balance is null then balance else balance - pre_balance end as chanced_balance
from cte3
order by customer_id
``` 
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/d2db48b9-670a-4750-9d11-5ae100bdbf63)


#### 5. What is the percentage of customers who increase their closing balance by more than 5%?
```sql
with 
cte1 as (
select customer_id,  
DATEPART(month,txn_date) as monthh,
sum(case when txn_type = 'deposit' then txn_amount else 0 end) as count_deposit,
sum(case when txn_type = 'purchase' then -txn_amount else 0 end) as count_purchase,
sum(case when txn_type = 'withdrawal' then -txn_amount else 0 end) as count_withdrawal
from customer_transactions
group by customer_id, DATEPART(month,txn_date)

),

cte2 as (
select customer_id, monthh,count_deposit + count_purchase + count_withdrawal as balance 
from cte1
),
cte3 as (
select customer_id, monthh, balance , 
FIRST_VALUE(balance) over (partition by customer_id order by customer_id) as opening_balance,
LAST_VALUE(balance) over (partition by customer_id order by customer_id desc) as ending_balance
from cte2
),
cte4 as (
SELECT *, 
((ending_balance - opening_balance) * 100 / abs(opening_balance)) as growing_rate
FROM cte3
WHERE ((ending_balance - opening_balance) * 100 / abs(opening_balance)) >= 5 AND ending_balance >opening_balance
)
select 
cast(count(distinct(customer_id)) as float)*100
/
(select cast(count(distinct(customer_id)) as float) from customer_transactions) as Percent_Customer
from cte4
```
![image](https://github.com/leevanhoc/SQL-CHALLENGE-8-WEEK/assets/173981700/6033ee68-5388-441b-b6eb-251b4c149dfb)
