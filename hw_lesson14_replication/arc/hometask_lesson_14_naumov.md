Имеются 2 виртуальные машины vm1 (postgres (10.128.0.6)) и  vm2 (postgresreplica (10.128.0.8)) в которых мы создадим базы testlogical с таблицами и настроим логическую репликацию. 

1. Создаём кластер на ВМ1, настроим файл pg_hba.conf и запустим кластер

   sudo pg_createcluster 14 vm1

   Ver Cluster Port Status Owner    Data directory             Log file
   14  vm1     5433 down   postgres /var/lib/postgresql/14/vm1 /var/log/postgresql/postgresql-14-vm1.log

   Изменим pg_hba.conf

   host    testlogical    all             10.128.0.8/0            scram-sha-256

   Стартуем кластер

   sudo pg_ctlcluster 14 vm1 start

2. Подключимся к кластеру и создадим базу с таблицами

   psql postgres postgres -p 5433

   CREATE DATABASE testlogical;

   \c testlogical

   CREATE TABLE test1 (column1 text);

   CREATE TABLE test2 (column1 text);

3. Повторим действия на ВМ2, отличие только в настройке pg_hba.conf

   host    testlogical    all             10.128.0.6/0            scram-sha-256

4. Изменим уровень журналирования на обоих машинах и перезагрузим кластер

   ALTER SYSTEM SET wal_level = logical;

   sudo pg_ctlcluster 14 vm1 restart

   sudo pg_ctlcluster 14 vm2 restart

5. Создадим публикацию таблицы test1 на vm1 и проверим подписку

   CREATE PUBLICATION vm1_test1_pub FOR TABLE test1;

   CREATE PUBLICATION

   \dRp+

                            Publication vm1_test1_pub
     Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
   ----------+------------+---------+---------+---------+-----------+----------
    postgres | f               | t          | t              | t            | t                | f

6. Создадим публикацию таблицы test2 на vm2 и проверим подписку

   CREATE PUBLICATION vm2_test2_pub FOR TABLE test2;

   CREATE PUBLICATION

   \dRp+

                            Publication vm2_test2_pub
     Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
   ----------+------------+---------+---------+---------+-----------+----------
    postgres | f              | t           | t              | t            | t                | f
   Tables:
       "public.test2"

   

7. Создадим на vm1 подписку к таблице test2 из vm2 и проверим подписку

   CREATE SUBSCRIPTION vm2_test2_sub CONNECTION 'host= 10.128.0.8 port=5433 user=postgres password=OTUS dbname=testlogical' PUBLICATION vm2_test2_pub WITH (copy_data = false);

   CREATE SUBSCRIPTION

   \dRs

                   List of subscriptions
        Name      |  Owner   | Enabled |   Publication
   ---------------+----------+---------+-----------------
    vm2_test2_sub | postgres     | t               | {vm2_test2_pub}
   (1 row)

   

   Посмотрим на состояние подписки

   SELECT * FROM pg_stat_subscription \gx

   -[ RECORD 1 ]---------+------------------------------
   subid                 | 16405
   subname               | vm2_test2_sub
   pid                   | 8889
   relid                 |
   received_lsn          | 0/1719E78
   last_msg_send_time    | 2022-04-28 21:52:04.946361+00
   last_msg_receipt_time | 2022-04-28 21:52:04.947054+00
   latest_end_lsn        | 0/1719E78
   latest_end_time       | 2022-04-28 21:52:04.946361+00

8. Cоздадим на vm2 подписку к таблице test1 из vm1 и проверим подписку

   CREATE SUBSCRIPTION vm1_test1_sub CONNECTION 'host= 10.128.0.6 port=5433 user=postgres password=OTUS dbname=testlogical' PUBLICATION vm1_test1_pub WITH (copy_data = false);

   CREATE SUBSCRIPTION

   \dRs

                   List of subscriptions
        Name      |  Owner   | Enabled |   Publication
   ---------------+----------+---------+-----------------
    vm1_test1_sub  | postgres     | t              | {vm1_test1_pub}
   (1 row)

   

   Посмотрим на состояние подписки

   SELECT * FROM pg_stat_subscription \gx

   -[ RECORD 1 ]---------+------------------------------
   subid                 | 16398
   subname               | vm1_test1_sub
   pid                   | 8775
   relid                 |
   received_lsn          | 0/171D470
   last_msg_send_time    | 2022-04-28 21:49:25.092118+00
   last_msg_receipt_time | 2022-04-28 21:49:25.092309+00
   latest_end_lsn        | 0/171D470
   latest_end_time       | 2022-04-28 21:49:25.092118+00

   

9. Проверим работу подсписки vm1_test1_sub к публикации vm1_test1_pub вставив строку в таблицу test1 на vm1 и прочитав данные на vm2 в таблице test1

   vm1: 

   INSERT INTO test1 VALUES ('line1');

   INSERT 0 1

   SELECT * FROM test1; \gx

   -[ RECORD 1 ]--
   column1 | line1

   

   Проверим состояние репликации на vm1

   SELECT * FROM pg_stat_replication; \gx

   testlogical=# -[ RECORD 1 ]----+------------------------------
   pid              | 8926
   usesysid         | 10
   usename          | postgres
   application_name | vm1_test1_sub
   client_addr      | 10.128.0.8
   client_hostname  |
   client_port      | 55810
   backend_start    | 2022-04-28 21:48:17.899319+00
   backend_xmin     |
   state            | streaming
   sent_lsn         | 0/171D6E8
   write_lsn        | 0/171D6E8
   flush_lsn        | 0/171D6E8
   replay_lsn       | 0/171D6E8
   write_lag        |
   flush_lag        |
   replay_lag       |
   sync_priority    | 0
   sync_state       | async
   reply_time       | 2022-04-28 21:59:23.963452+00

   

   vm2

   Проверим работу подписки

   SELECT * FROM pg_stat_subscription \gx

   -[ RECORD 1 ]---------+------------------------------
   subid                 | 16398
   subname               | vm1_test1_sub
   pid                   | 8775
   relid                 |
   received_lsn          | 0/171D6E8
   last_msg_send_time    | 2022-04-28 22:00:44.060635+00
   last_msg_receipt_time | 2022-04-28 22:00:44.061394+00
   latest_end_lsn        | 0/171D6E8
   latest_end_time       | 2022-04-28 22:00:44.060635+00

   Как видно, номера LSN на vm1 и vm2 совпадают

   

   Теперь просто сделаем выборку из таблицы test1 на vm2

   SELECT * FROM test1;

   -[ RECORD 1 ]--
   column1 | line1

10. Проведём аналогичный тест с подпиской vm2_test2_sub к публикации vm2_test2_pub на vm2 вставив строку в таблицу test2 на vm2 и прочитав данные на vm1 в таблице test2

    vm2:

    INSERT INTO test2 VALUES ('line1');

    INSERT 0 1

    SELECT * FROM test2; \gx

    -[ RECORD 1 ]--
    column1 | line1

    

    Проверим состояние репликации на vm2

    SELECT * FROM pg_stat_replication; \gx

    -[ RECORD 1 ]----+------------------------------
    pid              | 8774
    usesysid         | 10
    usename          | postgres
    application_name | vm2_test2_sub
    client_addr      | 10.128.0.6
    client_hostname  |
    client_port      | 55018
    backend_start    | 2022-04-28 21:48:10.00192+00
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/171A1C8
    write_lsn        | 0/171A1C8
    flush_lsn        | 0/171A1C8
    replay_lsn       | 0/171A1C8
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 0
    sync_state       | async
    reply_time       | 2022-04-28 22:08:52.781521+00

    

    Проверим работу подписки на vm1

    SELECT * FROM pg_stat_subscription; \gx

    -[ RECORD 1 ]---------+------------------------------
    subid                 | 16405
    subname               | vm2_test2_sub
    pid                   | 8889
    relid                 |
    received_lsn          | 0/171A1C8
    last_msg_send_time    | 2022-04-28 22:10:52.929136+00
    last_msg_receipt_time | 2022-04-28 22:10:52.929326+00
    latest_end_lsn        | 0/171A1C8
    latest_end_time       | 2022-04-28 22:10:52.929136+00

    Как видно, номера LSN на vm2 и vm1 совпадают

    

    Теперь просто сделаем выборку из таблицы test2 на vm1

    SELECT * FROM test2; \gx

    -[ RECORD 1 ]--
    column1 | line1

11. Cоздадим ещё одну виртуальную машину и создадим на ней кластер PostgreSQL

    sudo pg_createcluster 14 vm3 -p 5433

    pg_lsclusters

    14  vm3     5433 down   postgres /var/lib/postgresql/14/vm3 /var/log/postgresql/postgresql-14-vm3.log

    sudo pg_ctlcluster 14 vm3 start

12. Подключимся к клаcтеру и создадим в нём базу testlogical с таблицами test1 и test2

    psql postgres postgres -p 5433

    CREATE DATABASE testlogical;

    \c testlogical

    CREATE TABLE test1 (column1 text);

    CREATE TABLE test2 (column1 text);

13. На машинах vm1 и vm2 настроим правила для репликации на машину vm3

    Исправим pg_hba.conf на обоеих машинах добавив строку 

    host    testlogical     all             10.128.0.9/0            md5

14. Создадим подписку на таблицы в кластере vm3 выставляя флаг copy_data в true чтобы в таблицы заполнились данными

    CREATE SUBSCRIPTION vm3_vm2_test2_sub CONNECTION 'host= 10.128.0.8 port=5433 user=postgres password=OTUS dbname=testlogical' PUBLICATION vm2_test2_pub WITH (copy_data = true);

    CREATE SUBSCRIPTION

    CREATE SUBSCRIPTION vm3_vm1_test1_sub CONNECTION 'host= 10.128.0.6 port=5433 user=postgr
    es password=OTUS dbname=testlogical' PUBLICATION vm1_test1_pub WITH (copy_data = false);
    NOTICE:  created replication slot "vm3_vm1_test1_sub" on publisher

    CREATE SUBSCRIPTION

    Проверим состояние подписок на машинt vm3

    \dRs

                      List of subscriptions
           Name        |  Owner   | Enabled |   Publication
    -------------------+----------+---------+-----------------
     vm3_vm1_test1_sub | postgres | t       | {vm1_test1_pub}
     vm3_vm2_test2_sub | postgres | t       | {vm2_test2_pub}

    Проверим заполнение данных в таблицах

    \x

     SELECT * FROM test1;

    -[ RECORD 1 ]--
    column1 | line1

    SELECT * FROM test2;

    -[ RECORD 1 ]--
    column1 | line1

15. Проверим состояние реплицации на vm1 и vm2

    vm1:

    SELECT * FROM pg_stat_replication; \x

    -[ RECORD 1 ]----+------------------------------
    pid              | 1421
    usesysid         | 10
    usename          | postgres
    application_name | vm1_test1_sub
    client_addr      | 10.128.0.8
    client_hostname  |
    client_port      | 42736
    backend_start    | 2022-04-30 18:24:30.700269+00
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/171DAD0
    write_lsn        | 0/171DAD0
    flush_lsn        | 0/171DAD0
    replay_lsn       | 0/171DAD0
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 0
    sync_state       | async
    reply_time       | 2022-04-30 19:15:14.008136+00
    -[ RECORD 2 ]----+------------------------------
    pid              | 5493
    usesysid         | 10
    usename          | postgres
    application_name | vm3_vm1_test1_sub
    client_addr      | 10.128.0.9
    client_hostname  |
    client_port      | 48912
    backend_start    | 2022-04-30 19:08:53.521504+00
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/171DAD0
    write_lsn        | 0/171DAD0
    flush_lsn        | 0/171DAD0
    replay_lsn       | 0/171DAD0
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 0
    sync_state       | async
    reply_time       | 2022-04-30 19:15:14.07411+00

    Видно, что появился ещё один слот репликации

    

    vm2

    SELECT * FROM pg_stat_replication; \x

    -[ RECORD 1 ]----+------------------------------
    pid              | 1279
    usesysid         | 10
    usename          | postgres
    application_name | vm2_test2_sub
    client_addr      | 10.128.0.6
    client_hostname  |
    client_port      | 49096
    backend_start    | 2022-04-30 18:24:26.072921+00
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/171A3D0
    write_lsn        | 0/171A3D0
    flush_lsn        | 0/171A3D0
    replay_lsn       | 0/171A3D0
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 0
    sync_state       | async
    reply_time       | 2022-04-30 19:18:32.863439+00
    -[ RECORD 2 ]----+------------------------------
    pid              | 3760
    usesysid         | 10
    usename          | postgres
    application_name | vm3_vm2_test2_sub
    client_addr      | 10.128.0.9
    client_hostname  |
    client_port      | 60760
    backend_start    | 2022-04-30 18:52:40.989134+00
    backend_xmin     |
    state            | streaming
    sent_lsn         | 0/171A3D0
    write_lsn        | 0/171A3D0
    flush_lsn        | 0/171A3D0
    replay_lsn       | 0/171A3D0
    write_lag        |
    flush_lag        |
    replay_lag       |
    sync_priority    | 0
    sync_state       | async
    reply_time       | 2022-04-30 19:18:33.017491+00

    Видно, что появился ещё один слот репликации

16. Проверим работу репликации добавив новую строку в таблицу test1 кластера vm1

    INSERT INTO test1 (colum1) VALUES ('line2');

    INSERT 0 1

    \x

    SELECT * FROM test1;

    -[ RECORD 1 ]--
    column1 | line1
    -[ RECORD 2 ]--
    column1 | line2

17. Прверим как запись отреплицировалась на машинах vm2 и vm3

    vm2

    SELECT * FROM test1;

    -[ RECORD 1 ]--
    column1 | line1
    -[ RECORD 2 ]--
    column1 | line2

    vm3

    SELECT * FROM test1;

    -[ RECORD 1 ]--
    column1 | line1
    -[ RECORD 2 ]--
    column1 | line2

    

    

    

    

