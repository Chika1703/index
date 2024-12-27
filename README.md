# Домашнее задание к занятию «Индексы» Dmitry Kolb

## Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### решение 
немного не понял что именно нужно посчитать поэтому выполнил два запроса, а именно:
* первый запрос, если нужно посчитать процент индексов только по отношению к данным:

```sql
SELECT
  ROUND((SUM(stat.INDEX_LENGTH) / SUM(stat.DATA_LENGTH)) * 100, 2) AS percentage 
FROM 
  information_schema.TABLES stat
WHERE 
  stat.TABLE_SCHEMA = 'sakila';
```

![image 1](png/1.png)

* второй запрос, если нужно посчитать процент индексов к размеру всей таблицы:
```sql
SELECT ##второй запрос, если нужно посчитать процент индексов к размеру всей таблицы
  ROUND((SUM(stat.INDEX_LENGTH) / SUM(stat.DATA_LENGTH + stat.INDEX_LENGTH)) * 100, 2) AS percentage 
FROM 
  information_schema.TABLES stat
WHERE 
  stat.TABLE_SCHEMA = 'sakila';

```
![image 2](png/2.png)

## Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### решение

#### узкие места
выполнив ```EXPLAIN ANALYZE``` в неоптимизированном запросе получил результат: (actual time=3389..3389 rows=391 loops=1).
* Использование ```,``` для соединения таблиц (старый синтаксис):
  * Этот метод менее эффективен и сложнее для оптимизатора. Лучше использовать ```JOIN```.
* Функция ```DATE(p.payment_date)``` в ```WHERE```:
  * Применение ```DATE()``` к столбцу ```p.payment_date``` приводит к тому, что индекс на ```payment_date``` не используется.
  * Это вызывает полное сканирование таблицы
* Отсутствие индексов на столбцах, участвующих в JOIN:
  * Соединения по ```rental_date```, ```customer_id```, ```inventory_id``` и ```film_id``` без индексов могут привести к Nested Loop Join с полной проверкой записей.
* ```DISTINCT```:
  * ```DISTINCT``` требует сортировки или хеширования большого объема данных, что замедляет выполнение.
* ```OVER (PARTITION BY)```:
  * Окно, основанное на двух столбцах ```customer_id``` и ```title```, может требовать временных таблиц для выполнения, особенно если таблицы большие.

![image 3](png/3.png)

#### оптимизация запроса
1. Создание индексов для ускорения соединений и фильтрации:
2. ```JOIN``` вместо ```,```
3. Удаляем ```DATE()``` в ```WHERE``` и переписываем фильтр через интервал
4. Удаление ```DISTINCT```

в ходе оптимизации я создал индексы и улучшил запрос:
```sql
CREATE INDEX idx_payment_date ON payment (payment_date);
CREATE INDEX idx_rental_date ON rental (rental_date);
CREATE INDEX idx_customer_id ON customer (customer_id);
CREATE INDEX idx_inventory_id ON inventory (inventory_id);
CREATE INDEX idx_film_title ON film (title);
CREATE INDEX idx_film_id ON inventory (film_id);
```

```sql
SELECT DISTINCT 
  CONCAT(c.last_name, ' ', c.first_name) AS full_name,
  SUM(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS total_amount
FROM 
  payment p
JOIN 
  rental r ON p.payment_date = r.rental_date
JOIN 
  customer c ON r.customer_id = c.customer_id
JOIN 
  inventory i ON r.inventory_id = i.inventory_id
JOIN 
  film f ON i.film_id = f.film_id
WHERE 
  p.payment_date >= '2005-07-30 00:00:00' 
  AND p.payment_date < '2005-07-31 00:00:00';
```
и выводит результат ```EXPLAIN ANALYZE```: (actual time=9.15..9.19 rows=602 loops=1)

![image 4](png/4.png)















