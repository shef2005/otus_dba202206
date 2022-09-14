

# Настройка autovacuum с учетом оптимальной производительности

Цель:
запустить нагрузочный тест pgbench с профилем нагрузки DWH
настроить параметры autovacuum для достижения максимального уровня устойчивой производительности

Описание/Пошаговая инструкция выполнения домашнего задания:

## 1 создать GCE инстанс типа e2-medium и standard disk 10GB

##     установить на него PostgreSQL 13 с дефолтными настройками

**Использовал VM PostgreSQL 14**

## 2 Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

**Предложены начальные настройки:**

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

Поскольку max_connections, shared_buffers, wal_buffers имеют context='postmaster' редактируем файл `postgresql.conf`.

Перегружаем  кластер

```
shef2005@shef2005-VirtualBox:~$ sudo pg_ctlcluster 14 main restart
```

[sudo] password for shef2005:

## 3 запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres дать отработать до конца

**Заходим и создаем базу configdb**

```
sudo -i -u postgres psql psql
postgres=# create database configdb;
```

CREATE DATABASE

**Начальный запуск pgbench**

```
\q
pgbench -i configdb
```

dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.33 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.98 s (drop tables 0.28 s, create tables 0.13 s, client-side generate 0.73 s, vacuum 0.33 s, primary keys 0.52 s).
postgres@shef2005-VirtualBox:~$

**Запуск pgbench** 

```
pgbench -c8 -P 60 -T 3600 -U postgres configdb
```

pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 330.0 tps, lat 23.802 ms stddev 11.263

progress: 120.0 s, 337.2 tps, lat 23.341 ms stddev 22.282

....

progress: 3060.0 s, 403.3 tps, lat 19.515 ms stddev 7.479
progress: 3120.0 s, 401.4 tps, lat 19.603 ms stddev 5.651
progress: 3180.0 s, 408.5 tps, lat 19.265 ms stddev 5.711
progress: 3240.0 s, 404.3 tps, lat 19.466 ms stddev 5.875
progress: 3300.0 s, 337.2 tps, lat 23.313 ms stddev 9.912
progress: 3360.0 s, 386.2 tps, lat 20.375 ms stddev 7.277
progress: 3420.0 s, 384.8 tps, lat 20.453 ms stddev 6.081
progress: 3480.0 s, 390.1 tps, lat 20.175 ms stddev 5.840
progress: 3540.0 s, 385.0 tps, lat 20.452 ms stddev 26.727
progress: 3600.0 s, 392.3 tps, lat 20.061 ms stddev 6.486
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1310708
latency average = 21.616 ms
latency stddev = 10.790 ms
initial connection time = 72.525 ms
tps = 364.087879 (without initial connection time)

## 3 зафиксировать среднее значение tps в последней ⅙ части работы

**389,31**

## 4  дальше настроить autovacuum максимально эффективно так, чтобы получить максимально ровное значение tps на горизонте часа

**Редактируем файл `postgresql.conf`.**

log_autovacuum_min_duration = '0'
autovacuum_max_workers = '6' 
autovacuum_naptime = '10s' 
autovacuum_vacuum_threshold = '250'
autovacuum_vacuum_scale_factor = '0.05'
autovacuum_vacuum_cost_delay = '10'
autovacuum_vacuum_cost_limit = '1000'

**Перегружаем  кластер**

shef2005@shef2005-VirtualBox:~$ sudo pg_ctlcluster 14 main restart
shef2005@shef2005-VirtualBox:~$ sudo -i -u postgres psql
psql (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
Type "help" for help.

**Проверка ожидающих настроек**

```
select 
   "name", setting ,unit,context, pending_restart 
from pg_settings ps 
 where 1=1
   --and ps."name" like '%vacuum%'
  and pending_restart
order by 1
;
```

postgres=# \q

**Начальный запуск pgbench**

shef2005@shef2005-VirtualBox:~$ sudo su postgres
postgres@shef2005-VirtualBox:/home/shef2005$ pgbench -i configdb
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.37 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.27 s (drop tables 0.08 s, create tables 0.04 s, client-side generate 0.45 s, vacuum 0.28 s, primary keys 0.42 s).

**Запуск pgbench**

```
postgres@shef2005-VirtualBox:/home/shef2005$ pgbench -c8 -P 60 -T 3600 -U postgres configdb
```

pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 344.8 tps, lat 22.808 ms stddev 14.446
progress: 120.0 s, 359.3 tps, lat 21.905 ms stddev 7.213

...

progress: 3060.0 s, 346.6 tps, lat 22.698 ms stddev 7.202
progress: 3120.0 s, 347.7 tps, lat 22.631 ms stddev 7.425
progress: 3180.0 s, 337.9 tps, lat 23.288 ms stddev 7.055
progress: 3240.0 s, 328.9 tps, lat 23.927 ms stddev 11.230
progress: 3300.0 s, 332.3 tps, lat 23.693 ms stddev 9.332
progress: 3360.0 s, 358.8 tps, lat 21.936 ms stddev 6.615
progress: 3420.0 s, 335.2 tps, lat 23.482 ms stddev 11.516
progress: 3480.0 s, 351.0 tps, lat 22.429 ms stddev 7.922
progress: 3540.0 s, 332.7 tps, lat 23.652 ms stddev 12.217
progress: 3600.0 s, 314.0 tps, lat 25.052 ms stddev 11.901
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1259184
latency average = 22.499 ms
latency stddev = 8.669 ms
initial connection time = 86.674 ms
tps = 349.776429 (without initial connection time)

В итоге среднее за последние 10 мин

**tps= 338.51**

