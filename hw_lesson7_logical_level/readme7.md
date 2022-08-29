

Описание/Пошаговая инструкция выполнения домашнего задания:

# Выполнение домашнего задания по теме "Логический уровень PostgreSQL"

## 1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL, VM)

```
Использовал ранее созданную VM
```

## 2 Зайдите в созданный кластер под пользователем postgres

```
sudo -u postgres psql
```

## 3 Создайте новую базу данных testdb
```
postgres=# create database testdb;
CREATE DATABASE
```

## 4 Зайдите в созданную базу данных под пользователем postgres

```
postgres=# \c testdb
Вы подключены к базе данных "testdb" как пользователь "postgres".
```

## 5 Создайте новую схему testnm

```
testdb=# create schema testnm;
CREATE SCHEMA
```

## 6 Создайте новую таблицу t1 с одной колонкой c1 типа integer

```
testdb=# create table t1(c1 int);
CREATE TABLE
```

## 7 Вставьте строку со значением c1=1

```
testdb=# insert into t1(c1) values (1);
INSERT 0 1
testdb=# select * from t1;
 c1
----
  1
  select * from t1;
(1 строка)
```

## 8 Создайте новую роль readonly

```
testdb=# create role readonly;
CREATE ROLE
```

## 9 дайте новой роли право на подключение к базе данных testdb

```
testdb=# grant connect on database testdb to readonly;
GRANT
```

## 10 дайте новой роли право на использование схемы testnm

```
testdb=# grant usage on schema testnm to readonly;
GRANT
```

## 11 дайте новой роли право на select для всех таблиц схемы testnm

```
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```

## 12 Создайте пользователя testread с паролем test123

```
testdb=# create role testread with password 'test123';
CREATE ROLE
```

## 13 Дайте роль readonly пользователю testread

```
testdb=# grant readonly to testread;
GRANT ROLE
```

## 14 Зайдите под пользователем testread в базу данных testdb

```
testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=>
```

## 15 Сделайте select * from t1;

```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```

## 16 Получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

```
ERROR:  permission denied for table t1
```

## 17 Напишите что именно произошло в тексте домашнего задания

```
Исправляем ситуацию
```

- ## *предоставляем пользователю права на `login`:*

```
testdb=# alter role testread with login;
ALTER ROLE
```

- изменяем тип подключения для пользователей с `peer` на `md5`:

  ```
  sudo nano /etc/postgresql/13/main/pg_hba.conf
  
    local   all             all                                     md5 
  ```

- Перегружаем кластер   

  ```
  sudo pg_ctlcluster 13 main reload
  ```

## 18 У вас есть идеи почему? ведь права то дали?

```
таблица создана в схеме public 
```

## 19 Посмотрите на список таблиц

```
testdb=> \dt
       List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

## 20 Подсказка в шпаргалке под пунктом 20

```
таблица создана в схеме public,а не testnm и прав на public для роли readonly не давали
```

## 21 А почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)

```
testdb=> show search_path;
   search_path   
-----------------
 "$user", public
(1 row)
```

```
 Согласно `search_path`, таблицы сначала создаются в схеме `"$user"`, соответствующей имени пользователя, под которым мы работаем в базе данных, и, если одноименной схемы нет, то таблицы попадают в схему `public`.
Чтобы мы получили доступ к таблице `t1` под пользователем `testread` нам необходимо было либо создавать нашу таблицу в схеме `testnm`, либо предоставлять дополнительные права на таблицы схемы `public`.
```

## 22 Вернитесь в базу данных testdb под пользователем postgres

```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
```

## 23 Удалите таблицу t1

```
testdb=# drop table t1;
DROP TABLE
testdb=# \dt
Did not find any relations.
```

## 24 создайте ее заново но уже с явным указанием имени схемы testnm

```
testdb=# create table testnm.t1(c1 int);
CREATE TABLE
```

## 25 Вставьте строку со значением c1=1

```
testdb=# insert into testnm.t1(c1) values (1);
INSERT 0 1
```

## 26 Зайдите под пользователем testread в базу данных testdb

```
testdb=# \c testdb testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=>
```

## 27 Сделайте select * from testnm.t1;

```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

## 28 Получилось?

```
нет доступа
```

## 29 Есть идеи почему? если нет - смотрите шпаргалку

```
потому что grant SELECT on all TABLEs in SCHEMA testnm TO readonly дал доступ только для существующих на тот момент времени таблиц, а t1 пересоздавалась
```

## 30 Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

```
\c testdb postgres; 
ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly; 
\c testdb testread;
```

## 31 Сделайте select * from testnm.t1;

```
testdb=> select * from testnm.t1;
ERROR:  relation "testnm.t1" does not exist
СТРОКА 1: select * from testnm.t1;
```

## 32 Получилось?

```
testdb=> select * from testnm.t1;
 c1 
----
  1
(1 row)
```



## 33 ура!

36 есть идеи как убрать эти права? если нет - смотрите шпаргалку
37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
39 расскажите что получилось и почему



## 34 Теперь попробуйте выполнить команду create table t2(c1 integer); insert into insert into t2 values (2); 

```
estdb=> create table t2(c1 integer); insert into 
CREATE TABLE
testdb-> 
testdb-> t2 values (2);
INSERT 0 1
```

## 35 А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

```
это все потому что search_path указывает в первую очередь на схему public. А схема public создается в каждой базе данных по умолчанию. И grant на все действия в этой схеме дается роли public. А роль public добавляется всем новым пользователям.
```

## 36 Есть идеи как убрать эти права? если нет - смотрите шпаргалку

Уберем право у роли `public` на `create` в схеме `public`, а также все права в базе данных в базе данных `testdb` для пользователя `public`:

```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# revoke create on schema public from public;
REVOKE
testdb=# revoke all on database testdb from public;
REVOKE
```

## 37 Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните, что сделали и почему выполнив указанные в ней команды

```
Делал сам + смотрел шпаргалку
```

## 38 Теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

```
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
СТРОКА 1: create table t2(c1 integer);
                       ^
INSERT 0 1
```

## 39 Расскажите что получилось и почему

```
Создать таблицу t3 не удалось, так как ранее убрали права на создание таблицы в схеме `public`.  
Наполнить таблицу у нас получилось, поскольку права на INSERT не забирали
```

