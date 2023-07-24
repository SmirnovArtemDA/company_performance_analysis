# Анализ эффективности компании с помощью SQL 

#### ЦЕЛИ И ЗАДАЧИ

[АНАЛИЗ ЭФФЕКТИВНОСТИ КОМПАНИИ Pens and Pencils](https://docs.google.com/spreadsheets/d/1N7rB4Hmvca2o5I_UcBrQJzVaIK8on__oN4mstj9VeMY/edit?usp=sharing) с точки зрения её эффективности и дать рекомендации по масштабированию бизнеса, а именно в каком штате лучше открыть офлайн-магазин.


#### КОНКРЕТНЫЕ ШАГИ 
1. Оценить динамику продаж и распределение выручки по товарам;
2. Составить портрет клиента, а для этого — выяснить, какие клиенты приносят больше всего выручки;
3. Проконтролировать логистику компании (определить, все ли заказы доставляются в срок и в каком штате лучше открыть офлайн-магазин).

##### Выручка по месяцам

```
WITH revenue_month AS
(SELECT 
date_trunc ('month', order_date) as month,
SUM (CASE WHEN discount>0 THEN (quantity*price)-(discount*(quantity*price))
     ELSE (quantity*price) END) as revenue
FROM  sql.store_products as sp
 JOIN sql.store_carts as sc on sp.product_id=sc.product_id
 JOIN sql.store_delivery as sd on sc.order_id=sd.order_id 
GROUP BY 1)

SELECT
to_char (month, 'yyyy-mm-dd') as date,
round (revenue,0) as revenue 
FROM revenue_month
order by 1
```
![суммa выручки по месяцам](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/077ab997-2524-4954-9f12-a4cd0c34c9ea)

Исходя из графика, можно видеть, что доходы компании растут. Но очень сильно зависят от сезона.

##### Выручка по категориям

```
WITH revenue_category AS
(SELECT 
     category,
     subcategory,
     SUM (CASE WHEN discount>0 THEN (quantity*price)-(discount*.       (quantity*price))
     ELSE (quantity*price) END) as revenue
FROM  sql.store_products as sp
 JOIN sql.store_carts as sc on sp.product_id=sc.product_id
 JOIN sql.store_delivery as sd on sc.order_id=sd.order_id 
GROUP BY 1,2)

SELECT 
     category,
     subcategory,
     round (revenue,0) as revenue
FROM revenue_category
ORDER BY 3 DESC
```
<img width="256" alt="Снимок экрана 2023-07-24 в 15 27 52" src="https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/b1ced119-3877-467b-87f9-ad4d222f26b9">

В данной таблице, отражена выручка по категориям и подкатегориям товаров.  Стулья и телефоны имеют наибольшую выручку в компании.

##### ТОП - 25 товаров.
```
WITH revenue_category AS
(SELECT 
      product_nm,
      SUM (quantity) as quantity,
      SUM (CASE WHEN discount>0 THEN (quantity*price)-(discount*(quantity*price))
      ELSE (quantity*price) END) as revenue
FROM  sql.store_products as sp
 JOIN sql.store_carts as sc on sp.product_id=sc.product_id
 JOIN sql.store_delivery as sd on sc.order_id=sd.order_id 
GROUP BY 1),

total_revenues AS
(SELECT 
     SUM (CASE WHEN discount>0 THEN (quantity*price)-(discount*(quantity*price))
     ELSE (quantity*price) END) as total_revenue
FROM  sql.store_products as sp
 JOIN sql.store_carts as sc on sp.product_id=sc.product_id)

SELECT 
   product_nm,
   ROUND (revenue,2) as revenue,
   quantity,
   ROUND (revenue/total_revenue*100,2) as percent_from_total
FROM revenue_category
 JOIN total_revenues on true
ORDER BY revenue desc
LIMIT 25
```
<img width="707" alt="Снимок экрана 2023-07-24 в 15 45 21" src="https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/94aa60c9-861c-481a-9574-6d67bb4f732e">

На странице посторена таблица с топ-25 товарами компании. Исходя из нее видно, что очень весомую долю занимают фотоаппараты и электроника в целом. 

##### Выручка по клиентам
```
SELECT  
   sc.category,
   COUNT (distinct sc.cust_id) as cust_cnt,
   round(sum(price * quantity * (1 - discount))) as revenue
FROM sql.store_customers as sc
  JOIN sql.store_delivery as sd on sc.cust_id=sd.cust_id
  JOIN sql.store_carts as sc1 on sd.order_id=sc1.order_id
  JOIN sql.store_products as sp on sc1.product_id=sp.product_id
GROUP BY 1
ORDER BY 3 DESC
```
![Соотношение категорий клиентов](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/b2e05eb4-6b72-427c-8be6-b79a40c03928)
![Выручка по категориям клиентов](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/e69ed0b4-cf24-49a1-b0b9-1e1bb3879a85)

Вы можете заметить, что B2B-клиентов намного больше, и выручки они приносят тоже в разы больше!

##### Количество новых корпоративных клиентов
```
with first_order as (
select
    sd.cust_id,
    min(date_trunc('month', order_date))::date first_date
from
    sql.store_delivery sd
    join sql.store_customers sc on sd.cust_id = sc.cust_id 
where category = 'Corporate'
group by 1)

select
    first_date,
    count(cust_id)
from first_order
group by 1
order by 1
```
![Кол-во новых корпоративных клиентов по месяцам](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/b5d678cc-3930-4bba-8997-77ab193d5587)

Из нашего графика видно, что новые корпоративные клиенты активно приходили в 2017 году, а в дальнейшем приток пошел на спад. Это объясняет то, что компания нашла свою аудиторию и реботает с ней. Стоит поощрять как новых клиентов, так и старых, что бы оттока было как можно меньше. 

##### Средняя выручка корп. заказчиков
```
WITH cnt_table AS 
(SELECT  
   DISTINCT (sc.cust_id),
   sd.order_id,
   cust_nm,
   COUNT (zip_code) as office,
   COUNT (subcategory) as sub,
   round(sum(price * quantity * (1 - discount))) revenue
FROM sql.store_customers as sc
  JOIN sql.store_delivery as sd on sc.cust_id=sd.cust_id
  JOIN sql.store_carts as sc1 on sd.order_id=sc1.order_id
  JOIN sql.store_products as sp on sc1.product_id=sp.product_id
WHERE sc.category = 'Corporate'
GROUP BY 1,2,3) 

SELECT 
     cust_id,
     cust_nm,
     ROUND (AVG (office),1) as avg_office,
     ROUND (AVG (sub),1) as avg_cnt,
     ROUND (AVG (revenue),1) as avg_revenue
FROM cnt_table 
GROUP BY  1,2
ORDER by 5 DESC
```
![chart](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/84611e52-5fba-46d9-a816-44e3afbcb1d5)

В таблице представлены все корпаративные заказчики и среднее количество их товаров заказе, выручки и офисов. На графике показаны топ -10 покупателей по средней выручке за заказ. Tamara Chand и Bill Shonely имеют самые высокие показатели.

##### Информаци о доставках 
Доля заказов по каждой категории
```
with cte1 as 
(select
    ship_mode,
    count (order_id) as orders_cnt,
    count (case when ship_mode='Standard Class' and ship_date-order_date <=6 then 'intime'
                when ship_mode='Second Class' and ship_date-order_date <=4 then 'intime'
                when ship_mode='First Class' and ship_date-order_date <=3 then 'intime'
                when ship_mode='Same Day' and ship_date-order_date <=0 then 'intime'
                else null end)::numeric as intime_orders_cnt
from sql.store_delivery
group by 1
order by 1)
select
    ship_mode,
    orders_cnt,
    orders_cnt-intime_orders_cnt::integer as late_orders_cnt,
    round (intime_orders_cnt/orders_cnt*100, 2) as  "% success"
from cte1
order by 4
```
![доля заказов по каждой категории](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/d4e3e6dc-496e-49fe-b4ea-339a3a446063)

Доставка second class по кварталам 
```
with cte1 as 
(select
    ship_mode,
    extract (QUARTER FROM ship_date) as quarter,
    count (order_id) as orders_cnt,
    count (case when ship_mode='Second Class' and ship_date-order_date <=4 then 'intime'
                else null end)::numeric as intime_orders_cnt
from sql.store_delivery
group by 1,2)

select
    quarter,
    ship_mode,
    orders_cnt,
    orders_cnt-intime_orders_cnt::integer as late_orders_cnt,
    round (intime_orders_cnt/orders_cnt*100, 2) as " % success"
from cte1
WHERE ship_mode='Second Class'
order by 3
```
![доставка Second Class  по кварталам](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/6849aa0f-4ac7-4224-b0a5-b2deebc7d01c)

Из графиков и таблиц мы видим, что самая эффективная доставка - это первым классом. Остальные тоже хорошо себя показывают, кроме второго класса, где процент доставок во время всего 79%. В разрезе по кварталам видно, что они практически одинаковые и причина не может быть не контролируемая. Возможно стоит ее модернизировать и разобраться в проблеме.

##### Регионы
```
SELECT 
    state,
    COUNT (sd.order_id) as cnt_order
FROM sql.store_delivery
GROUP BY 1
ORDER BY 2 DESC
```
![Количество доставок по штатам](https://github.com/SmirnovArtemDA/company_performance_analysis/assets/139784954/72bffe0a-c371-4bd1-a498-b2c48fe38543)

Из графика видно, что больше всего доставок идет в штат Калифорния, Лос-Анджелес и Сан-Франциско являются главными потребителями. Логично было бы открыть магазин там, что и будет правильным решением. Но если брать отдельно города, то всего больше в Нью-Йорке. И это будет тоже правильное решение. Логистика в данный штат стоит дороже а из 562 доставок столица штата забирает 450. Что позволит практически исключить доставку.  По моему решению, лучше открыть 2 магазина в обоих штатах, если позволяют средства. Если нет, то выбор - Калифорния.


