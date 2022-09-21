# 1 Описание/Пошаговая инструкция выполнения домашнего задания:

## Секционировать большую таблицу из демо базы flights

## 1.1 Секционирование по диапазонам

## 1.1.0 Подготовка БД

Взял большую демо-базу demo-big postgrespro

```
sudo su - postgres
wget https://edu.postgrespro.ru/demo-big.zip
zcat demo-small.zip | psql 
```

### 1.1.1 Выполнение работы

Параметры работы:

v_tbl_src='bookings';              -- Исходная таблица
v_tbl_part='bookings_part';   -- Результирующая таблица 
v_tbl_fld='book_date';             -- Поле для деления

Написал скрипт, автоматически строит таблицы-разделы с делением по месяцам: 

```
do 
$$
declare 
   v_r record;
   v_cmd_loop varchar;
   v_cmd varchar; 
   v_tbl_src  varchar;
   v_tbl_part varchar;
   v_tbl_fld  varchar;
   v_suf      varchar;
   v_val_from varchar;
   v_val_to   varchar;
   v_limit_part varchar;  -- сколько разделов
   v_limit    varchar;  -- сколько записей  
begin
   set schema 'bookings';
   --v_limit_part='10';
   v_limit_part='NULL';
   --v_limit='100';
   v_limit='NULL';
   v_tbl_src='bookings';
   v_tbl_part='bookings_part';
   v_tbl_fld='book_date';
   --
   -- Создаем секционированную таблицу
   v_cmd=format
   (
	$x$   
drop table if exists %s;
CREATE table if not exists %s (LIKE %s) partition by range(%s);
$x$ ,v_tbl_part ,v_tbl_part, v_tbl_src ,v_tbl_fld   
   );
   RAISE NOTICE e'\n%', v_cmd;
   execute v_cmd;
   --
   --RAISE NOTICE e'table: \n%', v_tbl;
   v_cmd_loop=format
   (
   $x$
	   select 
		  date_trunc('month', t.%s::date)::date d1
		  from %s t
		 group by date_trunc('month', t.%s::date)
		 limit %s
   $x$ ,v_tbl_fld, v_tbl_src ,v_tbl_fld, v_limit_part  
   )
   ;
   --
   -- Цикл по месяцам(разделам)
   FOR v_r in
      execute v_cmd_loop 
   LOOP 	 
		--RAISE NOTICE 'cmd: %', v_r.d1;
	    v_suf=to_char(v_r.d1, '_YYYY_MM');
	    --RAISE NOTICE 'v_suf: %', v_suf;
	    v_val_from=to_char(v_r.d1, 'YYYY-MM-DD');
	    --RAISE NOTICE 'v_val_from: %', v_val_from;
	    v_val_to=to_char(v_r.d1+'1month'::interval , 'YYYY-MM-DD');
	    --RAISE NOTICE 'v_val_to: %', v_val_to;
	    --
	    v_cmd=format('drop table if Exists %s%s',v_tbl_part ,v_suf);
	    execute v_cmd;
	    -- 
	    v_cmd=format
	    (
	   'create table if not Exists %s%s PARTITION OF %s for VALUES FROM (%L) TO (%L);'
	    , v_tbl_part ,v_suf ,v_tbl_part ,v_val_from ,v_val_to
	    );
	    --
        RAISE NOTICE e'\n%', v_cmd;
        execute v_cmd;
	    --
   end loop
   ;
   --
   -- создание default раздела
   v_suf='_default';
   v_cmd=format('drop table if Exists %s%s;',v_tbl_part, v_suf );
   execute v_cmd;
   --  
   v_cmd=format('create table if not Exists %s%s PARTITION OF %s default;', v_tbl_part ,v_suf ,v_tbl_part);
   RAISE NOTICE e'\n%', v_cmd;
   execute v_cmd;
   --
   -- перенос данных 
   v_cmd=format
   (
   $x$insert into %s select * from %s limit %s 
   $x$
      ,v_tbl_part,v_tbl_src, v_limit 
   )
   ;
   RAISE NOTICE e'\n%', v_cmd;
   execute v_cmd;
   -- 
   RAISE NOTICE e'FINISH!!!';
end
$$
;
```

Фрагмент результата выполнения:

```
drop table if exists bookings_part;
CREATE table if not exists bookings_part (LIKE bookings) partition by range(book_date);

create table if not Exists bookings_part_2015_11 PARTITION OF bookings_part for VALUES FROM ('2015-11-01') TO ('2015-12-01');
create table if not Exists bookings_part_2015_12 PARTITION OF bookings_part for VALUES FROM ('2015-12-01') TO ('2016-01-01');
...
create table if not Exists bookings_part_2015_10 PARTITION OF bookings_part for VALUES FROM ('2015-10-01') TO ('2015-11-01');
create table if not Exists bookings_part_default PARTITION OF bookings_part default;
insert into bookings_part select * from bookings limit NULL 
   
FINISH!!!
```



### 1.1.3 Проверка результатов


--Записей в исходной таблице
select count(1) from bookings.bookings;
2111110

--Записей в результирующей таблице
select count(1) from bookings.bookings_part;
2111110

--Записей в таблице default
select count(1) from bookings.bookings_part_default;
0

Желаемые результаты достигнуты.

## 1.2 Секционирование по списку

Создадим новую таблицу ticket_flights_part по структуре копию таблицы ticket_flights

```
drop TABLE if exists bookings.ticket_flights_part;
CREATE TABLE if not exists ticket_flights_part (LIKE ticket_flights) PARTITION BY LIST (fare_conditions);
```

-- создадим секции

```
drop TABLE if exists fare_conditions_economy;
CREATE TABLE if not exists fare_conditions_economy PARTITION OF ticket_flights_part FOR VALUES IN ('Economy');

drop TABLE if exists fare_conditions_comfort;
CREATE TABLE if not exists fare_conditions_comfort PARTITION OF ticket_flights_part FOR VALUES IN ('Comfort');

drop TABLE if exists fare_conditions_business;
CREATE TABLE if not exists fare_conditions_business PARTITION OF ticket_flights_part FOR VALUES IN ('Business');
```

-- Перенесём в таблицу ticket_flights_part данные из таблицы ticket_flights

```
INSERT INTO ticket_flights_part SELECT * FROM ticket_flights; -- 8391852
```

Проверим результаты

```
select count(1) cnt from ticket_flights; -- 8391852

select count(1) cnt from ticket_flights_part; -- 8391852
```

**OK**

## 1.3 Секционирование по hash

Для работы выбрана таблица **flights**

-- Создаем таблицу flights_hash

```
drop table if exists flights_hash;
create table if not exists flights_hash (like flights) partition by hash(flight_id);
```

-- делаем секции 

```
create table flights_hash1 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 0);
create table flights_hash2 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 1);
create table flights_hash3 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 2);
create table flights_hash4 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 3);
create table flights_hash5 partition of flights_hash FOR VALUES WITH (MODULUS 5, REMAINDER 4);
```

-- Загружаем данные:

```
insert into flights_hash (flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival) select * from bookings.flights;
--214867
```

--Делаем проверку:

```
select count(*) from flights_hash1; -- 42740
select count(*) from flights_hash2; -- 43228
select count(*) from flights_hash3; -- 42944
select count(*) from flights_hash4; -- 42920
select count(*) from flights_hash5; -- 43035
```

-- Данные в таблице были разбиты на примерно равные части.

















