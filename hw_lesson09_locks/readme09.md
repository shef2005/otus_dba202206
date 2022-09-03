# Механизм блокировок

## 0 Подготовка

###  0.1 Создаем отдельную базу `locks` для отслеживания блокировок

```
sudo -i -u postgres psql

postgres=# create database locks;
CREATE DATABASE
postgres=# \c locks
You are now connected to database "locks" as user "postgres".
```

### 0.2 Создадим таблицу и наполним данными:

```
locks=# CREATE TABLE accounts(id integer, amount numeric);
CREATE TABLE
locks=# INSERT INTO accounts VALUES (1,2000.00), (2,2000.00), (3,2000.00);
INSERT 0 3
locks=# select * from accounts;
 id | amount
----+---------
  1 | 2000.00
  2 | 2000.00
  3 | 2000.00
(3 rows)
```

## 1 Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

-- посмотрим текущие настройки

```
select name, setting from pg_catalog.pg_settings ps where name in('log_min_duration_statement','log_lock_waits','deadlock_timeout') order by 1;
```

| name                       | setting |
| -------------------------- | ------- |
| deadlock_timeout           | 1000    |
| log_lock_waits             | off     |
| log_min_duration_statement | -1      |

Меняем параметры `log_lock_waits` и `deadlock_timeout`:

```
alter system set log_min_duration_statement = -1;
ALTER SYSTEM

alter system set log_lock_waits = on;
ALTER SYSTEM

alter system set deadlock_timeout = 200;
ALTER SYSTEM
```

## 2 Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

- В первом окне выполняем операцию с командой Update на поле id таблицы accounts:

```
SELECT pg_backend_pid();
 pg_backend_pid 
----------------
           29677
(1 row)
locks=# BEGIN;
BEGIN

locks=*# UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;
UPDATE 1
```

- Во втором и третьем окне выполним операцию Update:

  pg_backend_pid=39362

  pg_backend_pid=39402

- В четвертом окне посмотрим pg_locks

  ```
  SELECT 
    p.pid ,locktype ,relation::REGCLASS ,virtualxid AS virtxid ,transactionid AS xid ,mode
    ,granted
    ,pg_blocking_pids(p.pid) as wait_for 
  FROM pg_locks p  
  where 1=1
    and p.pid<>pg_backend_pid()
  order by pid ,locktype
  ;
  ```

- Результат в таблице ниже

| pid   | locktype      | relation | virtxid | xid    | mode             | granted | wait_for |
| ----- | ------------- | -------- | ------- | ------ | ---------------- | ------- | -------- |
| 29677 | relation      | accounts |         |        | RowExclusiveLock | true    | {}       |
| 29677 | transactionid |          |         | 446780 | ExclusiveLock    | true    | {}       |
| 29677 | virtualxid    |          | 8/114   |        | ExclusiveLock    | true    | {}       |
| 39362 | relation      | accounts |         |        | RowExclusiveLock | true    | {29677}  |
| 39362 | transactionid |          |         | 446781 | ExclusiveLock    | true    | {29677}  |
| 39362 | transactionid |          |         | 446780 | ShareLock        | false   | {29677}  |
| 39362 | tuple         | accounts |         |        | ExclusiveLock    | true    | {29677}  |
| 39362 | virtualxid    |          | 14/6    |        | ExclusiveLock    | true    | {29677}  |
| 39402 | relation      | accounts |         |        | RowExclusiveLock | true    | {39362}  |
| 39402 | transactionid |          |         | 446782 | ExclusiveLock    | true    | {39362}  |
| 39402 | tuple         | accounts |         |        | ExclusiveLock    | false   | {39362}  |
| 39402 | virtualxid    |          | 15/5    |        | ExclusiveLock    | true    | {39362}  |

Процесс 29677 ничего не ждёт (кроме commit), 39362 ждёт пока завершится 29677, а 39402 ждёт пока завершится 39362. 

39362,39402  не может получить ExclusiveLock для строки, поскольку его уже получил апдейт из 29677. 

RowExclusiveLock разрешены всем процессам, потому что они не конфликтуют между собой.

## 3 Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?



Да, по логу можно понять при каких транзакциях, в каких процессах и при какой последовательности возникли блокировки.

Смотрим журнал:  sudo tail -100 /var/log/postgresql/postgresql-13-main.log

```
2022-09-03 11:10:24.654 MSK [29677] postgres@locks STATEMENT:  begin 
	UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;

2022-09-03 11:11:53.169 MSK [39362] postgres@locks LOG:  process 39362 still waiting for ShareLock on transaction 446780 after 200.350 ms
2022-09-03 11:11:53.169 MSK [39362] postgres@locks DETAIL:  Process holding the lock: 29677. Wait queue: 39362.
2022-09-03 11:11:53.169 MSK [39362] postgres@locks CONTEXT:  while updating tuple (0,7) in relation "accounts"
2022-09-03 11:11:53.169 MSK [39362] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;

2022-09-03 11:12:59.492 MSK [39402] postgres@locks LOG:  process 39402 still waiting for ExclusiveLock on tuple (0,7) of relation 25013 of database 25012 after 200.295 ms
2022-09-03 11:12:59.492 MSK [39402] postgres@locks DETAIL:  Process holding the lock: 39362. Wait queue: 39402.
2022-09-03 11:12:59.492 MSK [39402] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount - 100.00 WHERE id = 1;
```

Запрос pg_locks - удобнее. См выше.

## 4 Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

**Да.**

### 4.1 UPDATE одной таблицы  без where в двух сеансах

CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00);

1-я сессия: UPDATE accounts SET acc_no = 10;
2-я сессия: UPDATE accounts SET acc_no = 10;

В этом случае второй апдейт будет заблокирован, поскольку будет ждать получения ShareLock на xid первого апдейта. 

Если бы был WHERE, то апдейты могли бы обновлять разные строки не мешая друг другу. 

### 4.2 UPDATE таблицы через курсор. 

Первый сеанс обновляет строки таблицы в прямом порядке: order by acc_no .

Второй сеанс обновляет строки таблицы в обратном порядке: order by acc_no desc.

