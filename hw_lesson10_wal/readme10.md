# Выполнение домашнего задания по теме "Работа с журналами"

## 1 Настройка выполнения контрольной точки
Настройте выполнение контрольной точки раз в 30 секунд.

Изменить настройки частоты выполнения контрольной точки можно двумя способами: через исправление файла `postrgesql.conf` либо через изменение настроек под psql `ALTER SYSTEM`. Выберем второй вариант:

```
student@xUbuntu-Spec:~$ sudo -u postgres psql
[sudo] password for student: 
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

-- посмотрим текущие настройки
postgres=# select name, setting from pg_catalog.pg_settings ps  where name like '%checkpoint%' order by 1;
             name             | setting 
------------------------------+---------
 checkpoint_completion_target | 0.5
 checkpoint_flush_after       | 32
 checkpoint_timeout           | 300
 checkpoint_warning           | 30
 log_checkpoints              | off
(5 rows)

postgres=# alter system set checkpoint_timeout='30s';
ALTER SYSTEM

-- изменим log_checkpoints='on';
postgres=# alter system set log_checkpoints='on';
ALTER SYSTEM

-- перезагрузим 
postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

-- посмотрим итоговые настройки
postgres=# select name, setting from pg_catalog.pg_settings ps  where name like '%checkpoint%' order by 1;
             name             | setting 
------------------------------+---------
 checkpoint_completion_target | 0.5
 checkpoint_flush_after       | 32
 checkpoint_timeout           | 30
 checkpoint_warning           | 30
 log_checkpoints              | on
(5 rows)


```

### 2.1 Создадим отдельную базу для тестирования нагрузки:

**CREATE DATABASE wal;**

```
CREATE DATABASE
```
### 2.2 Выполним инициализацию `pgbench`:

**pgbench -i wal**

```
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.31 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.93 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.49 s, vacuum 0.22 s, primary keys 0.20 s).
```

### 2.3 Прежде чем подать нагрузку на базу, сбросим статистику и запомним нынешнюю позицию в журнале:

**sudo -u postgres psql**

**\c wal**

```
You are now connected to database "wal" as user "postgres".
```

**wal=# SELECT pg_stat_reset_shared('bgwriter');**

```
pg_stat_reset_shared
----------------------

(1 row)
```

**wal=# SELECT pg_current_wal_insert_lsn();**

```
pg_current_wal_insert_lsn 
---------------------------
 0/B26BD58

(1 row)
```

### 2.4 Подаем нагрузку в течение 10 минут:

**^D**

**pgbench -P 10 -T 600 -U student wal**

```
starting vacuum...end.
progress: 10.0 s, 352.6 tps, lat 2.829 ms stddev 1.038
progress: 20.0 s, 326.8 tps, lat 3.059 ms stddev 1.258
progress: 30.0 s, 345.2 tps, lat 2.894 ms stddev 1.010
progress: 40.0 s, 368.2 tps, lat 2.715 ms stddev 1.648

progress: 530.0 s, 355.7 tps, lat 2.810 ms stddev 2.586
progress: 540.0 s, 354.2 tps, lat 2.821 ms stddev 1.665
progress: 550.0 s, 374.0 tps, lat 2.671 ms stddev 0.606
progress: 560.0 s, 358.9 tps, lat 2.785 ms stddev 2.254
progress: 570.0 s, 363.3 tps, lat 2.750 ms stddev 1.556
progress: 580.0 s, 374.8 tps, lat 2.666 ms stddev 0.521
progress: 590.0 s, 361.1 tps, lat 2.767 ms stddev 1.999
progress: 600.0 s, 356.6 tps, lat 2.802 ms stddev 1.603
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 214378
latency average = 2.797 ms
latency stddev = 1.592 ms
tps = 357.295968 (including connections establishing)
tps = 357.304538 (excluding connections establishing)
```

### 3 Просмотр статистики 

Проверим новую позицию в журнале:

**sudo -u postgres psql**

**wal=# SELECT pg_current_wal_insert_lsn();**

```
pg_current_wal_insert_lsn
---------------------------
0/21E6E820
(1 row)
```

Измерим, какой объем журнальных файлов был сгенерирован за время выполнения теста `pgbench`

**wal=# SELECT pg_size_pretty('0/21E6E820'::pg_lsn - '0/B26BD58'::pg_lsn);**

```
 pg_size_pretty 
----------------
 364 MB
(1 row)
```

Проверим статистику, сколько было пройдено контрольных точек:

**wal=# SELECT checkpoints_timed FROM pg_stat_bgwriter;**

```
 checkpoints_timed
-------------------
                 60
(1 row)
```
Видим, что за время тестирования нагрузки при помощи `pgbench` в среднем на 1 контрольную точку приходилось по 

364/60= 6,07MB данных.

Проверка общей статистики:

**wal=# SELECT * FROM pg_stat_bgwriter \gx**

```
SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 63
checkpoints_req       | 3
checkpoint_write_time | 329360
checkpoint_sync_time  | 481
buffers_checkpoint    | 41573
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3106
buffers_backend_fsync | 0
buffers_alloc         | 4383
stats_reset           | 2022-09-02 15:42:23.307349+03
```

С описанием параметров можно ознакомиться [здесь](https://postgrespro.ru/docs/postgrespro/12/monitoring-stats#PG-STAT-BGWRITER-VIEW).
Видим, что параметр `checkpoints_timed`, который соответствует контрольным точкам по расписанию (`checkpoint_timeout`) равен 63 и при этом для `checkpoints_req` равен 3. Объем данных  превысил `max_wal_size`, чтобы вызвать незапланированную контрольную точку.

## 4 Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

Нет, чуть раньше.

checkpoint_completion_target равен 0.5, тогда при checkpoint_timeout = 30 секунд контрольная точка будет выполняться каждые 15 секунд

### 5 Сравните tps в синхронном/асинхронном режиме утилитой pgbench

Объясните полученный результат.

1. По умолчанию используется синхронный режим:
```
sudo -u postgres psql
\c wal

wal=# SHOW synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)
```
2. Для тестирования, запустим снова `pgbench` на несколько секунд:
```
^D
postgres@postgres-5:~$ pgbench -P 1 -T 10 -U sudent wal
starting vacuum...end.
progress: 1.0 s, 349.9 tps, lat 2.805 ms stddev 0.882
progress: 2.0 s, 362.0 tps, lat 2.766 ms stddev 0.453
progress: 3.0 s, 294.0 tps, lat 3.390 ms stddev 4.585
progress: 4.0 s, 303.8 tps, lat 3.289 ms stddev 4.768
progress: 5.0 s, 366.2 tps, lat 2.734 ms stddev 0.693
progress: 6.0 s, 349.0 tps, lat 2.860 ms stddev 1.772
progress: 7.0 s, 357.0 tps, lat 2.800 ms stddev 0.963
progress: 8.0 s, 358.0 tps, lat 2.788 ms stddev 0.943
progress: 9.0 s, 364.0 tps, lat 2.752 ms stddev 0.918
progress: 10.0 s, 342.0 tps, lat 2.917 ms stddev 1.986
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 3447
latency average = 2.895 ms
latency stddev = 2.232 ms
tps = 344.645117 (including connections establishing)
tps = 345.190096 (excluding connections establishing)
```
3. Изменим режим записи на асинхронный и снова протестируем tps при помощи `pgbench`:
```
sudo -u postgres psql
\c wal

wal=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
wal=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```
```
^D
pgbench -P 1 -T 10 -U student wal
starting vacuum...end.
progress: 1.0 s, 722.8 tps, lat 1.357 ms stddev 0.625
progress: 2.0 s, 737.1 tps, lat 1.355 ms stddev 0.634
progress: 3.0 s, 770.8 tps, lat 1.296 ms stddev 0.524
progress: 4.0 s, 705.0 tps, lat 1.399 ms stddev 0.693
progress: 5.0 s, 741.0 tps, lat 1.366 ms stddev 0.843
progress: 6.0 s, 742.9 tps, lat 1.345 ms stddev 0.654
progress: 7.0 s, 735.1 tps, lat 1.359 ms stddev 0.683
progress: 8.0 s, 768.0 tps, lat 1.301 ms stddev 0.649
progress: 9.0 s, 710.0 tps, lat 1.407 ms stddev 0.677
progress: 10.0 s, 759.1 tps, lat 1.316 ms stddev 0.685
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 10 s
number of transactions actually processed: 7393
latency average = 1.349 ms
latency stddev = 0.671 ms
tps = 739.215861 (including connections establishing)
tps = 740.572020 (excluding connections establishing)
```
Как видим, в асинхронном режиме скорость возросла в 2 раза.
Однако, данные режим не гарантирует сохранность всех данных при падении сервера базы данных. Поэтому, данный режим не рекомендуется использовать при транзакциях с критичными данными.

### 6 Контрольные суммы страниц

### 6.1 По умолчанию, ведение контрольных сумм отключено:

```
postgres=# SHOW data_checksums;
 data_checksums
----------------
 off
(1 row)
```
### 6.2 Создадим отдельный кластер с включенным ведением контрольных сумм по умолчанию:

```
student@xUbuntu-Spec:~$ sudo pg_createcluster 13 test --  --data-checksums
Creating new PostgreSQL cluster 13/test ...
/usr/lib/postgresql/13/bin/initdb -D /var/lib/postgresql/13/test --auth-local peer --auth-host md5 --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locales
  COLLATE:  en_US.UTF-8
  CTYPE:    en_US.UTF-8
  MESSAGES: en_US.UTF-8
  MONETARY: ru_RU.UTF-8
  NUMERIC:  ru_RU.UTF-8
  TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/13/test ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Success. You can now start the database server using:

    pg_ctlcluster 13 test start
    
Ver Cluster Port Status Owner    Data directory              Log file
13  test    5433 down   postgres /var/lib/postgresql/13/test /var/log/postgresql/postgresql-13-test.log
```



```
student@xUbuntu-Spec:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
13  test    5433 down   postgres /var/lib/postgresql/13/test /var/log/postgresql/postgresql-13-test.log
```



```
student@xUbuntu-Spec:~$ sudo pg_ctlcluster 13 test start
student@xUbuntu-Spec:~$ sudo -i -u postgres psql -p 5433
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
(1 row)
```
### 6.3 Создадим таблицу и наполним ее данными:

```
postgres=# create table test_text(t text);
CREATE TABLE

postgres=# INSERT INTO test_text SELECT 'строка '||s.id FROM generate_series(1,10) AS s(id);
INSERT 0 10

postgres=# select * from test_text;
     t
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
 строка 6
 строка 7
 строка 8
 строка 9
 строка 10
(10 rows)
```
### 6.4 Перед тем, как поменять несколько байт на странице, необходимо проверить, где физически в базе расположена таблица:

```
postgres=# SELECT pg_relation_filepath('test_text');
 pg_relation_filepath
----------------------
base/13494/16384
(1 row)
```
### 6.5 Изменяем заголовок LNS для журнальной записи:

Выключаем кластер

```
^D
sudo pg_ctlcluster 13 test stop
```

Измените пару байт в таблице.

Из /dev/zero набираем 8 нулевых байт и вставляем в файл, где хранится наша тестовая таблица. затирая часть информации. Соответственно, при запросе postgres ругается на неверную контрольную сумму



```
sudo dd if=/dev/zero of=/var/lib/postgresql/13/test/base/13494/16384 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00803994 s, 1.0 kB/s
```

```
sudo pg_ctlcluster 13 test start
```



```
sudo dd if=/dev/zero of=/var/lib/postgresql/13/test/base/13414/16384 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00803994 s, 1.0 kB/s
```
### 6.6 Подключаемся к нашей базе и пробуем получить данные таблицы:

```
sudo pg_ctlcluster 13 test start
```

```
sudo -i -u postgres psql -p 5433
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.
```

```
postgres=# select * from test_text;
WARNING:  page verification failed, calculated checksum 30809 but expected 22930
ERROR:  invalid page in block 0 of relation base/13494/16384
```

Ошибка. Данные таблицы повреждены.
Чтобы увидеть наши данные необходимо выполнить следующую команду:

```
postgres=# SET ignore_checksum_failure = on;
SET
```

```
postgres=# select * from test_text;
WARNING:  page verification failed, calculated checksum 30809 but expected 22930
     t     
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
 строка 6
 строка 7
 строка 8
 строка 9
 строка 10
(10 rows)
```

После того, как мы отключим `ignore_checksum_failure`, мы все равно сможем увидеть необходимые данные:
```
postgres=# SET ignore_checksum_failure = off;
SET
postgres=# select * from test_text;
     t
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
 строка 6
 строка 7
 строка 8
 строка 9
 строка 10
(10 rows)
```