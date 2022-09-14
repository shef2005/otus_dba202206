## Выполнение домашнего задания по теме "Репликация"

### Подготовка инфраструктуры и первоначальная настройка

1. Для выполнения задания нам понадобится 4 виртуальные машины. Для удобства назовем их следующим образом:
- postgres-7-1
- postgres-7-2
- postgres-7-3
- postgres-7-4

2. На каждой машине отредактируем файл `/etc/hosts` и внесем дута IP адреса созданных машин, чтобы было удобнее обращаться к ним.
```
desmond@postgres-7-1:~$ sudo vim /etc/hosts
---
10.128.0.18 postgres-7-1
10.128.0.19 postgres-7-2
10.128.0.20 postgres-7-3
10.128.0.21 postgres-7-4
---
```
3. Устанавливаем PostgreSQL 13 на каждой ВМ:
```
desmond@postgres-7-1:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip
```
4. Для того, чтобы нам было проще отличить какие команды выполняются на каком сервере и в какой базе, создадим файл `~/.psqlrc` и внесем туда дополнительные параметры:
```
echo "\set PROMPT1 '%M %n@%/%R%# '" >> ~/.psqlrc
```
где:

- %M - название хоста, с которого мы подключаемся к базе. Если подключение локально через Unix, то будет указано [local].
- %n - название пользователя, под которым мы работаем в базе.
- %/ - название базы данных, в которой мы работаем.

5. Для того, чтобы сервера могли общаться друг с другом, необходимо внести следующие параметры в конфигурационные файлы:
```
desmond@postgres-7-2:~$ sudo vim /etc/postgresql/13/main/postgresql.conf
---
listen_addresses = '*'
---
### только на 1 и 2 ВМ:
wal_level = logical
---
```
```
desmond@postgres-7-1:~$ sudo vim /etc/postgresql/13/main/pg_hba.conf
---
# IPv4 local connections:
host    all             all             0.0.0.0/0            md5
---
# Allow replication connections from localhost, by a user with the
# replication privilege.
host    replication     all             0.0.0.0/0            md5
---
```
Обязательно перезагружаем наши кластера:
```
desmond@postgres-7-1:~$ sudo pg_ctlcluster 13 main restart
```

### Создание отдельной роли, базы и предоставление необходимых прав

1. На каждом сервере создадим отдельную роль для репликации с правами суперпользователя:
```
desmond@postgres-7-1:~$ sudo -i -u postgres
postgres@postgres-7-1:~$ createuser --interactive
Enter name of role to add: replica
Shall the new role be a superuser? (y/n) y
```
2. Подключимся к нашему пользователю и создадим отдельную базу данных. Подключаться будем с ключами -h, -p, -U, -W что указывает на название хоста, порта, роли и требованием пароля соответственно:
```
postgres@postgres-7-1:~$ psql -h postgres-7-1 -p 5432 -U replica -d postgres -W
Password:
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres-7-1 replica@postgres=#
```
Как видим, внесенные параметры в `~/.psqlrc` дают нам информацию о положении в psql - `postgres-7-1 replica@postgres`.

3. На каждом сервере создадим отдельную базу данных и дадим все права на нее для новой роли:
```
postgres-7-1 replica@postgres=# create database replicadb;
CREATE DATABASE
postgres-7-1 replica@postgres=# GRANT ALL PRIVILEGES ON DATABASE "replicadb" TO "replica";
GRANT
postgres-7-1 replica@postgres=# \c replicadb
Password:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "replicadb" as user "replica".
```

### Настройка логической репликации

1. На 1 BM создадим таблицу на запись и публикацию для нее:
```
postgres-7-1 replica@replicadb=# CREATE TABLE test(i int);
CREATE TABLE
postgres-7-1 replica@replicadb=# CREATE PUBLICATION test_pub FOR TABLE test;
CREATE PUBLICATION
```
2. На 2 ВМ создадим такую же таблицу и подпишемся на публикацию таблицы test с 1 ВМ:
```
postgres-7-2 replica@replicadb=# CREATE TABLE test(i int);
postgres-7-2 replica@replicadb=# CREATE SUBSCRIPTION test_sub
postgres-7-2 replica@replicadb-# CONNECTION 'host=postgres-7-1 port=5432 user=replica password=123$otus dbname=replicadb'
postgres-7-2 replica@replicadb-# PUBLICATION test_pub WITH (copy_data = false);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
3. На 1 ВМ внесем значения в таблицу и проверим информацию на 2 ВМ
```
postgres-7-1 replica@replicadb=# insert into test values(1);
INSERT 0 1
postgres-7-1 replica@replicadb=# select * from test;
 i
---
 1
(1 row)
---
postgres-7-2 replica@replicadb=# select * from test;
 i
---
 1
(1 row)
```
Как видим, логическая репликация настроена между таблицами test на 1 и 2 ВМ.
4. На 2 ВМ создадим таблицу test2 для записи и подпишемся на ее публикации с 1 ВМ. Сразу проверим нашу репликацию:
```
postgres-7-2 replica@replicadb=# CREATE TABLE test2(i int);
CREATE TABLE
postgres-7-2 replica@replicadb=# CREATE PUBLICATION test2_pub FOR TABLE test2;
CREATE PUBLICATION
---
postgres-7-1 replica@replicadb=# CREATE TABLE test2(i int);
CREATE TABLE
postgres-7-1 replica@replicadb=# CREATE SUBSCRIPTION test2_sub
postgres-7-1 replica@replicadb-# CONNECTION 'host=postgres-7-2 port=5432 user=replica password=123$otus dbname=replicadb'
postgres-7-1 replica@replicadb-# PUBLICATION test2_pub WITH (copy_data = false);
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
---
postgres-7-2 replica@replicadb=# insert into test2 values(1);
INSERT 0 1
---
postgres-7-1 replica@replicadb=# select * from test2;
 i
---
 1
(1 row)
```
5. На 3 ВМ сделаем подписки на таблицы test с 1 ВМ и test2 с 2 ВМ:
```
postgres-7-3 replica@replicadb=# CREATE SUBSCRIPTION test_sub
postgres-7-3 replica@replicadb-# CONNECTION 'host=postgres-7-1 port=5432 user=replica password=123$otu
s dbname=replicadb'
postgres-7-3 replica@replicadb-# PUBLICATION test_pub WITH (copy_data = false);
ERROR:  could not create replication slot "test_sub": ERROR:  replication slot "test_sub" already exists
```
Что произошло? дело в том, что подписка `test_sub` была создана на хосте `postgres-7-2`, о чем и говорит ошибка - такой слот уже есть.
Для каждой реплики нужен отдельный слот:
```
postgres-7-3 replica@replicadb=# CREATE SUBSCRIPTION test3_sub
CONNECTION 'host=postgres-7-1 port=5432 user=replica password=123$otus dbname=replicadb'
PUBLICATION test_pub WITH (copy_data = false);
NOTICE:  created replication slot "test3_sub" on publisher
CREATE SUBSCRIPTION
postgres-7-3 replica@replicadb=# CREATE SUBSCRIPTION test4_sub
CONNECTION 'host=postgres-7-2 port=5432 user=replica password=123$otus dbname=replicadb'
PUBLICATION test2_pub WITH (copy_data = false);
NOTICE:  created replication slot "test4_sub" on publisher
CREATE SUBSCRIPTION
```
6. Проведем небольшой тест. На 1 и 2 ВМ добавим данные и проверим, подтянулись ли они на 3 ВМ:
```
postgres-7-1 replica@replicadb=# insert into test values(2);
INSERT 0 1
postgres-7-1 replica@replicadb=# insert into test values(3);
INSERT 0 1
postgres-7-1 replica@replicadb=# select * from test;
 i
---
 1
 2
 3
(3 rows)
---
postgres-7-2 replica@replicadb=# select * from test;
 i
---
 1
 2
 3
(3 rows)
---
postgres-7-2 replica@replicadb=# insert into test2 values(2);
INSERT 0 1
postgres-7-2 replica@replicadb=# insert into test2 values(3);
INSERT 0 1
---
postgres-7-1 replica@replicadb=# select * from test2;
 i
---
 1
 2
 3
(3 rows)
---
postgres-7-3 replica@replicadb=# select * from test;
 i
---
 2
 3
(2 rows)

postgres-7-3 replica@replicadb=# select * from test2;
 i
---
 2
 3
(2 rows)
```
Как видим, данные есть только с последними транзакциями, так как при создании подписки мы указывали ключ `copy_data = false` при котором не копировались уже имеющиеся данные таблиц.

### Настройка физической репликации

1. Удаляем все содержимое кластера на 4 ВМ:
```
desmond@postgres-7-4:~$ sudo rm -rf /var/lib/postgresql/13/main
```
2. При помощи утилиты `pg_basebackup` копируем содержимое кластера с 3 ВМ:
```
sudo pg_basebackup -p 5432 -h postgres-7-3 -R -D /var/lib/postgresql/13/main -
U replica -W
Password:
```
3. Выполняем рестарт кластера:
```
desmond@postgres-7-4:~$ sudo pg_ctlcluster 13 main restart
Error: Data directory /var/lib/postgresql/13/main must not be owned by root
```
Упс, не сменили владельца директории :)
```
desmond@postgres-7-4:~$ sudo chown -R postgres:postgres /var/lib/postgresql/13/main
desmond@postgres-7-4:~$ sudo pg_ctlcluster 13 main restart
```
4. Проверим содержимое таблицы test:
```
postgres@postgres-7-4:~$ psql -h postgres-7-4 -U replica -d replicadb -W
Password:
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres-7-4 replica@replicadb=# select * from test;
 i
---
 2
 3
(2 rows)
```
5. Внесем еще одно значение в test и проверим нашу логическую и физическую реплику:
```
postgres-7-1 replica@replicadb=# insert into test values(4);
INSERT 0 1
---
postgres-7-2 replica@replicadb=# insert into test2 values(4);
INSERT 0 1
---
postgres-7-3 replica@replicadb=# select * from test2;
 i
---
 2
 3
 4
(3 rows)
---
postgres-7-4 replica@replicadb=# select * from test;
 i
---
 2
 3
 4
(3 rows)
```
Все работает!