# Credit_card_transactions_dataset  

### Data Set link on Kaggle  
https://www.kaggle.com/datasets/thedevastator/analyzing-credit-card-spending-habits-in-india  

### Overview  
This dataset contains insights into a collection of credit card transactions made in India, offering a comprehensive look at the spending habits of Indians across the nation. From the Gender and Card type used to carry out each transaction, to which city saw the highest amount of spending and even what kind of expenses were made, this dataset paints an overall picture about how money is being spent in India today.     
This project involves performing SQL queries of varying complexity(from medium to advanced). The primary goals of the project are to practice advanced SQL skills(using SQL Server) and generate valuable insights from the dataset.  

### SQL Queries    

--1. write a query to print top 5 cities with highest spends and their percentage contribution of total credit card spends.     
```sql
SELECT TOP 5 city, SUM(amount) city_wise_spend, ROUND(SUM(amount)*1.0/(SELECT SUM(amount) FROM credit_card_transcations)*100,2) percentage_spend
FROM credit_card_transcations
GROUP BY city
ORDER BY city_wise_spend DESC
```

--2. write a query to print highest spend month and amount spent in that month for each card type.    
```sql
WITH CTE1 AS
(
SELECT DATEPART(year, transaction_date) year, DATEPART(month, transaction_date) month, card_type, SUM(amount) total_amount
FROM credit_card_transcations
GROUP BY DATEPART(year, transaction_date), DATEPART(month, transaction_date), card_type
--ORDER BY DATEPART(year, transaction_date), DATEPART(month, transaction_date), card_type
), 

CTE2 AS
(
SELECT *, SUM(total_amount) over(partition by year, month) monthly_sum
FROM CTE1 
),

CTE3 AS
(
SELECT *, rank() over(order by monthly_sum DESC) rn
FROM CTE2
)

SELECT *
FROM CTE3
WHERE rn=1;
```

/*
3. write a query to print the transaction details(all columns from the table) for each card type when
it reaches a cumulative of 1000000 total spends(We should have 4 rows in the o/p one for each card type).
*/  

--for understanding  
```sql
SELECT *, SUM(amount) over(order by transaction_date rows between unbounded preceding and current row) cum_sum
FROM credit_card_transcations
WHERE card_type = 'Gold'
```

```sql
WITH CTE1 AS
(
SELECT *, SUM(amount) over(partition by card_type order by transaction_date rows between unbounded preceding and current row) cum_sum
FROM credit_card_transcations
),

CTE2 AS
(
SELECT *, rank() over(partition by card_type order by cum_sum) rn
FROM CTE1
WHERE cum_sum >= 1000000
)

SELECT *
FROM CTE2
WHERE rn =1;
```

--4. write a query to find city which had lowest percentage spend for gold card type.  
```sql
SELECT TOP 1 city, SUM(amount), SUM(amount)*1.0/(SELECT SUM(amount) FROM credit_card_transcations WHERE card_type = 'Gold')*100 percenatage_spend
FROM credit_card_transcations
WHERE card_type = 'Gold'
GROUP BY city
ORDER BY percenatage_spend 
```

--5. write a query to print 3 columns:  city, highest_expense_type , lowest_expense_type (example format : Delhi , bills, Fuel)  
```
WITH CTE1 AS
(
SELECT city, exp_type, SUM(amount) total, rank() over(partition by city order by SUM(amount) DESC ) rn1, rank() over(partition by city order by SUM(amount) ASC ) rn2
FROM credit_card_transcations
GROUP BY city, exp_type
--ORDER BY city, exp_type
)

,CTE2 AS
(
SELECT city, CASE WHEN rn1 = 1 THEN exp_type END highest_expense_type
FROM CTE1
WHERE CASE WHEN rn1 = 1 THEN exp_type END is not NULL
) 

,CTE3 AS
(
SELECT city, CASE WHEN rn2 = 1 THEN exp_type END lowest_expense_type
FROM CTE1
WHERE CASE WHEN rn2 = 1 THEN exp_type END is not NULL
) 

SELECT CTE2.city, highest_expense_type, lowest_expense_type
FROM CTE2
INNER JOIN CTE3 ON CTE2.city =CTE3.city
```

--6. write a query to find percentage contribution of spends by females for each expense type.  
```sql
SELECT exp_type, SUM(amount) total, SUM(CASE WHEN gender='F'THEN amount ELSE 0 END) female_contribution, SUM(CASE WHEN gender='F'THEN amount END)*1.0/SUM(amount)*100 percentage_female_contribution
FROM credit_card_transcations
GROUP BY exp_type
```

--7. which card and expense type combination saw highest month over month growth in Jan-2014.  
```sql
WITH CTE1 AS
(
SELECT card_type, exp_type, DATEPART(year,transaction_date) year, DATEPART(month,transaction_date) month, SUM(amount) total
FROM credit_card_transcations
GROUP BY card_type, exp_type, DATEPART(year,transaction_date),DATEPART(month,transaction_date)
--ORDER BY card_type, exp_type, DATEPART(year,transaction_date),DATEPART(month,transaction_date)
),

CTE2 AS
(
SELECT *, SUM(total) over(partition by card_type, exp_type order by year, month rows between 1 preceding and 1 preceding ) prev_month_amount
FROM CTE1
),

CTE3 AS
(
SELECT *, (total-prev_month_amount) MOM
FROM CTE2
WHERE year ='2014' and month ='1'
)

SELECT TOP 1 card_type, exp_type, MOM
FROM CTE3
ORDER BY MOM DESC
```


--for understanding purpose  
```sql
SELECT *, SUM(total) over(partition by card_type, exp_type order by year, month rows between 1 preceding and 1 preceding) prev_month_amount
FROM 
(
SELECT card_type, exp_type, DATEPART(year,transaction_date) year, DATEPART(month,transaction_date) month, SUM(amount) total
FROM credit_card_transcations
GROUP BY card_type, exp_type, DATEPART(year,transaction_date),DATEPART(month,transaction_date)
)A
```

--8. during weekends which city has highest total spend to total no of transcations ratio.   
```sql
WITH CTE AS
(
SELECT *, DATENAME(DW, transaction_date) day
FROM credit_card_transcations
)
SELECT TOP 1 city, SUM(amount), COUNT(*), SUM(amount)/COUNT(*) ratio
FROM CTE
WHERE day in ('saturday', 'sunday')
GROUP BY city
ORDER BY ratio DESC
```

--9. which city took least number of days to reach its 500th transaction after the first transaction in that city.  
```sql
WITH CTE1 AS
(
SELECT *, row_number() over (partition by city order by transaction_date) rn --and not rank()
FROM credit_card_transcations
),

CTE2 AS
(
SELECT city, transaction_date first_transaction_date
FROM CTE1 
WHERE rn=1 
),

CTE3 AS
(
SELECT city, transaction_date fivehundredth_transaction_date
FROM CTE1 
WHERE rn=500
)

SELECT TOP 1 CTE2.city, CTE2.first_transaction_date, CTE3.fivehundredth_transaction_date, DATEDIFF(day, CTE2.first_transaction_date, CTE3.fivehundredth_transaction_date ) days_to_500
FROM CTE2
INNER JOIN CTE3 ON CTE2.city =CTE3.city
ORDER BY days_to_500
```
### Technology Stack  
Database: **SQL Sever**    
SQL Queries: DML, DQL, Aggregations, Subqueries, Joins, Common table Expressions, Window Functions etc.   
Tools: SQL Server Management Studio 20  
