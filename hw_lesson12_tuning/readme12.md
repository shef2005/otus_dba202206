Домашнее задание 12
Нагрузочное тестирование и тюнинг PostgreSQL

Цель:

# 1 Cделать нагрузочное тестирование PostgreSQL

настроить параметры PostgreSQL для достижения максимальной производительности

Описание/Пошаговая инструкция выполнения домашнего задания:

## 1.1 Развернуть виртуальную машину любым удобным способом

Выполнена по инструкции: [Установка PostgreSQL на Ubuntu VirtualBox](https://virtual-dba.com/blog/installing-postgresql-ubuntu-virtualbox/)

## 1.2 Поставить на неё PostgreSQL 14 любым способом

## Как установить и настроить Postgres 14 Ubuntu 20.04

Выполнена по инструкции: [Как установить и настроить Postgres 14 Ubuntu 20.04](https://citizix.com/how-to-install-and-configure-postgres-14-ubuntu-20-04/)

```
apt update
apt upgrade
sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
apt-get update
apt-get install postgresql
```

```
root@shef2005-VirtualBox:/home/shef2005# pg_lsclusters
```

Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

## 1.3 Настроить кластер PostgreSQL 14 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

### 1.3.1 Подготовка БД для тестирования нагрузки 

```
sudo -i -u postgres psql
create database configdb; 
```

### 1.3.2 Первоначальное тестирование с дефолтными настройками

```
pgbench -i configdb
```

ropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.35 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.26 s (drop tables 0.02 s, create tables 0.17 s, client-side generate 0.65 s, vacuum 0.21 s, primary keys 0.21 s).

```
pgbench -c8 -P 10 -T 600 -U shef2005 configdb
```

shef2005@shef2005-VirtualBox:~$ pgbench -c8 -P 10 -T 600 -U shef2005 configdb
pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 305.1 tps, lat 25.566 ms stddev 17.623
progress: 20.0 s, 304.8 tps, lat 25.799 ms stddev 12.508
progress: 30.0 s, 317.7 tps, lat 24.745 ms stddev 7.687
progress: 40.0 s, 332.5 tps, lat 23.631 ms stddev 7.398
...  

progress: 560.0 s, 350.9 tps, lat 22.422 ms stddev 7.357
progress: 570.0 s, 342.3 tps, lat 22.976 ms stddev 8.133
progress: 580.0 s, 237.9 tps, lat 33.190 ms stddev 18.828
progress: 590.0 s, 265.0 tps, lat 29.834 ms stddev 14.664
progress: 600.0 s, 352.6 tps, lat 22.308 ms stddev 6.646
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 182996
latency average = 25.779 ms
latency stddev = 11.465 ms
initial connection time = 79.588 ms
**tps = 305.010708 (without initial connection time)**

### 1.3.3 Тестирование с настройками от pgtune: 

###   https://pgtune.leopard.in.ua/

Параметры:

```
# DB Version: 14
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 2 GB
# CPUs num: 1
# Connections num: 50
# Data Storage: hdd
```

Рекомендации:

max_connections = 50
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 5242kB
min_wal_size = 1GB
max_wal_size = 4GB

```
pg_cluster 14 main restart
```

select 
   "name", setting ,unit,context
from pg_settings ps 
 where 1=1
  and ps."name"=any('{max_connections,shared_buffers,effective_cache_size,maintenance_work_mem,checkpoint_completion_target,wal_buffers,default_statistics_target,random_page_cost,effective_io_concurrency,work_mem,min_wal_size,max_wal_size}'::_varchar)
order by 1  
; 

| name                         | setting | unit | context    |
| ---------------------------- | ------- | ---- | ---------- |
| checkpoint_completion_target | 0.9     |      | sighup     |
| default_statistics_target    | 100     |      | user       |
| effective_cache_size         | 196608  | 8kB  | user       |
| effective_io_concurrency     | 2       |      | user       |
| maintenance_work_mem         | 131072  | kB   | user       |
| max_connections              | 50      |      | postmaster |
| max_wal_size                 | 4096    | MB   | sighup     |
| min_wal_size                 | 1024    | MB   | sighup     |
| random_page_cost             | 4       |      | user       |
| shared_buffers               | 65536   | 8kB  | postmaster |
| wal_buffers                  | 2048    | 8kB  | postmaster |
| work_mem                     | 5242    | kB   | user       |

```
postgres@shef2005-VirtualBox:/home/shef2005$ pgbench -i configdb
```

dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.32 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.24 s (drop tables 0.22 s, create tables 0.09 s, client-side generate 0.48 s, vacuum 0.25 s, primary keys 0.20 s).



```
postgres@shef2005-VirtualBox:/home/shef2005$ pgbench -c8 -P 10 -T 600 -U shef2005 configdb
```

pgbench (14.5 (Ubuntu 14.5-1.pgdg20.04+1))
starting vacuum...end.
progress: 10.0 s, 354.8 tps, lat 21.977 ms stddev 8.440
progress: 20.0 s, 315.1 tps, lat 24.924 ms stddev 12.157
progress: 30.0 s, 377.3 tps, lat 20.843 ms stddev 8.084

...

progress: 580.0 s, 385.8 tps, lat 20.383 ms stddev 7.579
progress: 590.0 s, 403.1 tps, lat 19.532 ms stddev 8.250
progress: 600.0 s, 384.6 tps, lat 20.469 ms stddev 8.746
transaction type: <builtin: TPC-B (sort of)>

scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 216545
latency average = 21.786 ms
latency stddev = 9.778 ms
initial connection time = 72.922 ms
tps = **360.906682** (without initial connection time)

### 1.3.4 Тестирование с custom настройками : 

```
shared_buffers = 2GB
maintenance_work_mem = 512MB
effective_cache_size = 3GB
random_page_cost = 1
max_wal_senders = 0
wal_level = minimal
min_wal_size = 4GB
max_wal_size = 16GB
archive_mode = off
checkpoint_completion_target = 0.9
autovacuum = off
full_page_writes = off
synchronous_commit = off
fsync = off
work_mem = 5242kB
wal_buffers = 16MB
checkpoint_timeout = 1h
```

Получаем следующие данные:

progress: 580.0 s, 552.8 tps, lat 14.390 ms stddev 5.392
progress: 590.0 s, 552.4 tps, lat 14.397 ms stddev 5.931
progress: 600.0 s, 550.7 tps, lat 14.440 ms stddev 5.928
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 319058
latency average = 14.905 ms
latency stddev = 6.138 ms
initial connection time = 93.642 ms
tps = **531.790113** (without initial connection time)

## 1.4 написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

Начали с tps = 305.0

pgtune tps = 360.9

Custom tps = 531.7

- shared_buffers - сколько выделенной памяти будет использоваться PostgreSQL для кеширования. 

- maintenance_work_mem - задаёт максимальный объём памяти для операций обслуживания БД, в частности VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY.

- max_wal_senders, wal_level, archive_mode - все эти параметры важны для настройки репликации. Изменили, чтобы отключить передачу изменений через WAL в процессе загрузки. Это не только поможет сэкономить время архивации и передачи WAL

- checkpoint_timeout, checkpoint_completion_target - увеличим время "отдыха" контрольной точки для меньшей нагрузки.

- autovacuum - отключаем автовакуум только для ускорения транзакций. Рекомендовано не отключать его на Prod среде.

- full_page_writes, synchronous_commit, fsync - все эти параметры отключаем исключительно для получения максимального показателя tps. Изменения не будут записаны на диск физически.
