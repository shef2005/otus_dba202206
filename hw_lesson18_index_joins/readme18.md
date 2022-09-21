Домашнее задание 18
Работа с индексами, join'ами, статистикой

Цель:
знать и уметь применять основные виды индексов PostgreSQL строить и анализировать план выполнения запроса
уметь оптимизировать запросы для с использованием индексов, знать и уметь применять различные виды join'ов
строить и анализировать план выполнения запроса, оптимизировать запрос
уметь собирать и анализировать статистику для таблицы

Описание/Пошаговая инструкция выполнения домашнего задания:

# 1 вариант:

Создать индексы на БД, которые ускорят доступ к данным. В данном задании тренируются навыки: определения узких мест написания запросов для создания индекса оптимизации

Необходимо:

## 1.0 Подготовка БД

Взял маленькую demo базу flights с postgrespro.

```
sudo su - postgres
wget https://edu.postgrespro.ru/demo-small.zip
zcat demo-small.zip | psql
```

## 1.1 Создать индекс к какой-либо из таблиц вашей БД

Хотим посчитать суммарную стоимость купленных билетов бизнес и комфорт класса. 

Без индекса. Выполняется параллельное последовательное сканирование.

```
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(amount) FROM ticket_flights WHERE fare_conditions = 'Business' OR fare_conditions = 'Comfort';

Finalize Aggregate  (cost=16379.92..16379.93 rows=1 width=32) (actual time=447.594..453.832 rows=1 loops=1)
  Buffers: shared hit=2208 read=6507
  ->  Gather  (cost=16379.70..16379.91 rows=2 width=32) (actual time=447.539..453.780 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        Buffers: shared hit=2208 read=6507
        ->  Partial Aggregate  (cost=15379.70..15379.71 rows=1 width=32) (actual time=423.881..423.883 rows=1 loops=3)
              Buffers: shared hit=2208 read=6507
              ->  Parallel Seq Scan on ticket_flights  (cost=0.00..15250.79 rows=51564 width=6) (actual time=0.084..384.555 rows=41644 loops=3)
                    Filter: (((fare_conditions)::text = 'Business'::text) OR ((fare_conditions)::text = 'Comfort'::text))
                    Rows Removed by Filter: 306931
                    Buffers: shared hit=2208 read=6507
Planning:
  Buffers: shared hit=78 read=2
Planning Time: 1.007 ms
Execution Time: 454.060 ms
```

Добавляем индекс fare_conditions.

```
CREATE index concurrently if not exists ticket_flights_fare_conditions_idx ON bookings.ticket_flights (fare_conditions);
```

Результат команды explain, в которой используется этот индекс.

Запрос исполнился быстрее.

План:

```
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(amount) FROM ticket_flights WHERE fare_conditions = 'Business' OR fare_conditions = 'Comfort';

Finalize Aggregate  (cost=12059.83..12059.84 rows=1 width=32) (actual time=291.218..301.667 rows=1 loops=1)
  Buffers: shared hit=2231 read=5331
  ->  Gather  (cost=12059.60..12059.81 rows=2 width=32) (actual time=290.884..301.643 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        Buffers: shared hit=2231 read=5331
        ->  Partial Aggregate  (cost=11059.60..11059.61 rows=1 width=32) (actual time=269.225..269.229 rows=1 loops=3)
              Buffers: shared hit=2231 read=5331
              ->  Parallel Bitmap Heap Scan on ticket_flights  (cost=1431.62..10930.69 rows=51564 width=6) (actual time=22.860..239.567 rows=41644 loops=3)
                    Recheck Cond: (((fare_conditions)::text = 'Business'::text) OR ((fare_conditions)::text = 'Comfort'::text))
                    Heap Blocks: exact=2397
                    Buffers: shared hit=2231 read=5331
                    ->  BitmapOr  (cost=1431.62..1431.62 rows=125452 width=0) (actual time=26.964..26.966 rows=0 loops=1)
                          Buffers: shared hit=112
                          ->  Bitmap Index Scan on ticket_flights_fare_conditions_idx  (cost=0.00..1191.23 rows=109174 width=0) (actual time=25.799..25.799 rows=107642 loops=1)
                                Index Cond: ((fare_conditions)::text = 'Business'::text)
                                Buffers: shared hit=95
                          ->  Bitmap Index Scan on ticket_flights_fare_conditions_idx  (cost=0.00..178.51 rows=16278 width=0) (actual time=1.159..1.159 rows=17291 loops=1)
                                Index Cond: ((fare_conditions)::text = 'Comfort'::text)
                                Buffers: shared hit=17
Planning:
  Buffers: shared hit=19 read=1
Planning Time: 1.067 ms
Execution Time: 301.755 ms
```

## 1.2 Реализовать индекс для полнотекстового поиска

Найду всех однофамильцев из таблицы tickets. 

**Без индекса:**

```
DROP index if exists gin_idx;
```

План

```
EXPLAIN (ANALYZE, BUFFERS) select  * FROM tickets WHERE passenger_name ~ 'IVANOV';
Gather  (cost=1000.00..10164.47 rows=11104 width=104) (actual time=7.024..608.945 rows=14040 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  Buffers: shared hit=2610 read=3534
  ->  Parallel Seq Scan on tickets  (cost=0.00..8054.07 rows=4627 width=104) (actual time=0.152..569.433 rows=4680 loops=3)
        Filter: (passenger_name ~ 'IVANOV'::text)
        Rows Removed by Filter: 117564
        Buffers: shared hit=2610 read=3534
Planning Time: 0.425 ms
Execution Time: 610.410 ms
```

**С индексом**

```
CREATE INDEX if not exists gin_idx ON tickets USING gin(to_tsvector('english',passenger_name));
```

План

```
Bitmap Heap Scan on tickets  (cost=26.46..4910.08 rows=1834 width=104) (actual time=3.359..38.015 rows=7539 loops=1)
  Recheck Cond: (to_tsvector('english'::regconfig, passenger_name) @@ to_tsquery('IVANOV'::text))
  Heap Blocks: exact=4377
  Buffers: shared hit=2742 read=1641
  ->  Bitmap Index Scan on gin_idx  (cost=0.00..26.00 rows=1834 width=0) (actual time=2.077..2.077 rows=7539 loops=1)
        Index Cond: (to_tsvector('english'::regconfig, passenger_name) @@ to_tsquery('IVANOV'::text))
        Buffers: shared hit=6
Planning:
  Buffers: shared hit=19
Planning Time: 0.554 ms
Execution Time: 39.002 ms
```

Однофамильцы нашлись значительно быстрее

## 1.3 Реализовать индекс на часть таблицы или индекс на поле с функцией

Делаем индекс на наибольшую часть таблицы. Наибольшая - это fare_conditions = 'Economy':

SELECT count(*) FROM ticket_flights WHERE fare_conditions = 'Business'; -- 107 642  
SELECT count(*) FROM ticket_flights WHERE fare_conditions = 'Comfort'; --   17 291 
SELECT count(*) FROM ticket_flights WHERE fare_conditions = 'Economy'; --  920 793 

Используя индекс, сможем отсеять все ненужные нам данные.

```
drop INDEX  if exists fare_conditions_not_economy_idx;

CREATE INDEX CONCURRENTLY if not exists fare_conditions_not_economy_idx ON ticket_flights (fare_conditions) WHERE fare_conditions <> 'Economy';
```

Запрос

```
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(amount) FROM ticket_flights WHERE fare_conditions <> 'Economy';
```

План 

```
Aggregate  (cost=4129.23..4129.24 rows=1 width=32) (actual time=41.494..41.497 rows=1 loops=1)
  Buffers: shared hit=482
  ->  Index Only Scan using fare_conditions_not_economy_idx on ticket_flights  (cost=0.42..3815.60 rows=125452 width=6) (actual time=0.038..22.245 rows=124933 loops=1)
        Heap Fetches: 0
        Buffers: shared hit=482
Planning:
  Buffers: shared hit=13 read=1 dirtied=2
Planning Time: 0.391 ms
Execution Time: 41.550 ms 
```

Запрос получился дешевле по стоимости, план запроса проще, но по времени чуть дольше работает. 

## 1.4 Создать индекс на несколько полей 

Создать индекс на несколько полей.

Нам не так много данных нужно из таблицы, поэтому если мы добавим в индекс колонку amount, то это позволит всю нужную информацию для выполнения запроса получить из индекса:

DROP INDEX if exists fare_conditions_not_economy_idx;

CREATE INDEX concurrently if not exists fare_conditions_not_economy_idx ON ticket_flights (fare_conditions) INCLUDE (amount) WHERE fare_conditions <> 'Economy';

Действительно, запрос стал ещё дешевле и быстрее за счёт Index Only Scan:

EXPLAIN (ANALYZE, BUFFERS) SELECT sum(amount) FROM ticket_flights WHERE fare_conditions <> 'Economy';

## 1.5 Описать, что и как делали и с какими проблемами столкнулись

Использовал маленькую demo базу flights с postgrespro.

Большую big demo базу flights загрузить ВМ VirtualBox не удалось.

Вариантов два: 

- Создание/расширение диска ВМ
- Подключение Общей папки (share) папки, как дополнительного tablespace

# 2 вариант:

В результате выполнения ДЗ вы научитесь пользоваться  различными вариантами соединения таблиц.
В данном задании тренируются навыки: написания запросов с различными типами соединений

Поскольку примеры делаются в качестве иллюстраций, будем использовать конструкции CTE для генерации данных 

## 2.0 Подготовка исходных  данных

**t1** 

```
with t1 as 
(
select 
   generate_series(1,10,3) id,
   'x'||generate_series(1,10,3) x
   ,'y_'||generate_series(1,10,3) y
)
select * from t1;
```

Получим

| id   | x    | y    |
| ---- | ---- | ---- |
| 1    | x1   | y_1  |
| 4    | x4   | y_4  |
| 7    | x7   | y_7  |
| 10   | x10  | y_10 |

**t2**

```
with t2 as 
(
select 
   generate_series(4,12,2) id,
   'x'||generate_series(3,12,2) x
   ,'y_'||generate_series(3,12,2) y
)
select * from t2;
```

Получим

| id   | x    | y    |
| ---- | ---- | ---- |
| 4    | x3   | y_3  |
| 6    | x5   | y_5  |
| 8    | x7   | y_7  |
| 10   | x9   | y_9  |
| 12   | x11  | y_11 |

## 2.1 Реализовать прямое соединение двух или более таблиц

Визуально раздели запрос на две части: общую и решение задачи

```
with t1 as 
(
select 
   generate_series(1,10,3) id,
   'x'||generate_series(1,10,3) x
   ,'y_'||generate_series(1,10,3) y
)
,t2 as 
(
select 
   generate_series(4,12,2) id,
   'x'||generate_series(3,12,2) x
   ,'y_'||generate_series(3,12,2) y
)
```

```
select
   t1.id t1_id ,t1.x t1_x ,t1.y t1_y ,t2.id t2_id ,t2.x t2_x ,t2.y t2_y from t1 
   join t2 on t2.id=t1.id
;   
```

Результат

| t1_id | t1_x | t1_y | t2_id | t2_x | t2_y |
| ----- | ---- | ---- | ----- | ---- | ---- |
| 4     | x4   | y_4  | 4     | x3   | y_3  |
| 10    | x10  | y_10 | 10    | x9   | y_9  |

## 2.2 Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

```
with t1 as (select generate_series(1,10,3) id, 'x'||generate_series(1,10,3) x ,'y_'||generate_series(1,10,3) y)
,t2 as (select generate_series(4,12,2) id,  'x'||generate_series(3,12,2) x   ,'y_'||generate_series(3,12,2) y)
select t1.id t1_id ,t1.x t1_x ,t1.y t1_y ,t2.id t2_id ,t2.x t2_x ,t2.y t2_y from t1 
  left join t2 on t2.id=t1.id;  
```

Результат

| t1_id | t1_x | t1_y | t2_id | t2_x | t2_y |
| ----- | ---- | ---- | ----- | ---- | ---- |
| 4     | x4   | y_4  | 4     | x3   | y_3  |
| 10    | x10  | y_10 | 10    | x9   | y_9  |
| 1     | x1   | y_1  |       |      |      |
| 7     | x7   | y_7  |       |      |      |

## 2.3 Реализовать кросс соединение двух или более таблиц

```
with t1 as (select generate_series(1,10,3) id, 'x'||generate_series(1,10,3) x ,'y_'||generate_series(1,10,3) y)
,t2 as (select generate_series(4,12,2) id,  'x'||generate_series(3,12,2) x   ,'y_'||generate_series(3,12,2) y)
select t1.id t1_id ,t1.x t1_x ,t1.y t1_y ,t2.id t2_id ,t2.x t2_x ,t2.y t2_y from t1 
cross join t2;  
```

Результат

| t1_id | t1_x | t1_y | t2_id | t2_x | t2_y |
| ----- | ---- | ---- | ----- | ---- | ---- |
| 1     | x1   | y_1  | 4     | x3   | y_3  |
| 4     | x4   | y_4  | 4     | x3   | y_3  |
| 7     | x7   | y_7  | 4     | x3   | y_3  |
| 10    | x10  | y_10 | 4     | x3   | y_3  |
| 1     | x1   | y_1  | 6     | x5   | y_5  |
| 4     | x4   | y_4  | 6     | x5   | y_5  |
| 7     | x7   | y_7  | 6     | x5   | y_5  |
| 10    | x10  | y_10 | 6     | x5   | y_5  |
| 1     | x1   | y_1  | 8     | x7   | y_7  |
| 4     | x4   | y_4  | 8     | x7   | y_7  |
| 7     | x7   | y_7  | 8     | x7   | y_7  |
| 10    | x10  | y_10 | 8     | x7   | y_7  |
| 1     | x1   | y_1  | 10    | x9   | y_9  |
| 4     | x4   | y_4  | 10    | x9   | y_9  |
| 7     | x7   | y_7  | 10    | x9   | y_9  |
| 10    | x10  | y_10 | 10    | x9   | y_9  |
| 1     | x1   | y_1  | 12    | x11  | y_11 |
| 4     | x4   | y_4  | 12    | x11  | y_11 |
| 7     | x7   | y_7  | 12    | x11  | y_11 |
| 10    | x10  | y_10 | 12    | x11  | y_11 |

## 2.4 Реализовать полное соединение двух или более таблиц

```
with t1 as (select generate_series(1,10,3) id, 'x'||generate_series(1,10,3) x ,'y_'||generate_series(1,10,3) y)
,t2 as (select generate_series(4,12,2) id,  'x'||generate_series(3,12,2) x   ,'y_'||generate_series(3,12,2) y)
select t1.id t1_id ,t1.x t1_x ,t1.y t1_y ,t2.id t2_id ,t2.x t2_x ,t2.y t2_y from t1 
full join t2 on t2.id=t1.id
order by t1.id, t2.id;
```

| t1_id | t1_x | t1_y | t2_id | t2_x | t2_y |
| ----- | ---- | ---- | ----- | ---- | ---- |
| 1     | x1   | y_1  |       |      |      |
| 4     | x4   | y_4  | 4     | x3   | y_3  |
| 7     | x7   | y_7  |       |      |      |
| 10    | x10  | y_10 | 10    | x9   | y_9  |
|       |      |      | 6     | x5   | y_5  |
|       |      |      | 8     | x7   | y_7  |
|       |      |      | 12    | x11  | y_11 |

## 2.5 Реализовать запрос, в котором будут использованы разные типы соединений

```
with t1 as (select generate_series(1,10,3) id, 'x'||generate_series(1,10,3) x ,'y_'||generate_series(1,10,3) y)
,t2 as (select generate_series(4,12,2) id,  'x'||generate_series(3,12,2) x   ,'y_'||generate_series(3,12,2) y)
select t1.id t1_id ,t1.x t1_x ,t1.y t1_y ,t2.id t2_id ,t2.x t2_x ,t2.y t2_y ,t3.id t3_id ,t3.x t3_x ,t3.y t3_y
from t1 join t2 on t2.id=t1.id
  right join t2 t3 on t3.id=t2.id
;
```

| t1_id | t1_x | t1_y | t2_id | t2_x | t2_y | t3_id | t3_x | t3_y |
| ----- | ---- | ---- | ----- | ---- | ---- | ----- | ---- | ---- |
| 4     | x4   | y_4  | 4     | x3   | y_3  | 4     | x3   | y_3  |
|       |      |      |       |      |      | 6     | x5   | y_5  |
|       |      |      |       |      |      | 8     | x7   | y_7  |
| 10    | x10  | y_10 | 10    | x9   | y_9  | 10    | x9   | y_9  |
|       |      |      |       |      |      | 12    | x11  | y_11 |

## 2.6 К работе приложить структуру таблиц, для которых выполнялись соединения

Описаны в **2.0 Подготовка исходных  данных**

## 2.7 Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке

Метрики:

- **OLTP** - tps - число транзакций в сек

- **OLAP** - qps - число запросов в сек 

- **На работе** (OLAP) при построении витрин по договорам:  время в сек обработки порции в 1тыс договоров.

​        Дополнительный параметр - размер порции 15000-100000 записей

​        Размер порции устанавливаю в зависимости от количества полей витрины (сложность запроса).

​        Простой запрос 100000 записей. Сложный запрос 15000 записей.

