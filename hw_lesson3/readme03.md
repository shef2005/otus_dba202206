1. Создал каталог docker в папке $HOME

2. Скачал и поставил docker

   -- https://docs.docker.com/engine/install/ubuntu/
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh

3. Добавил своего пользователя serge в группу docker

   sudo usermod -aG docker serge

4. Создал каталоги, сменил владельца и права

   /var/lib/postgre

   /var/lib/postgresql/data

   chown serge:docker /var/lib/postgre && chown serge:docker /var/lib/postgresql/data

   chmod -R 755 /var/lib/postgre && chmod -R 755 /var/lib/postgresql/data

5. Скачал готовый image контейнера PostgreSQL 14

   sudo docker pull postgres:14

6. Создал docker-сеть

   sudo docker network create pg-net

7. Запустил контейнер PostgreSQL pg-server

   sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -e PGDATA=/var/lib/postgresql/data -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

   sudo docker container start pg-server

8. Зашёл внутрь контейнера чтобы проверить его работу

   sudo docker exec -it pg-server bash

9. Развернул контейнер pg-client

   sudo docker run -it --rm --network pg-net --name pg-client -e POSTGRES_PASSWORD=postgres -d postgres:14 

10. Запустил контейнер pg-client

    sudo docker container start pg-client

11. Зашёл внутрь контейнера чтобы проверить его работу

    sudo docker exec -it pg-client bash

12. Получил список работающих контейнеров

    sudo docker ps -a

    ea03157e6a7b   postgres:14   "docker-entrypoint.s…"   50 minutes ago      Up 50 minutes      5432/tcp                                    pg-client
    5752dd94cbab   postgres:14   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

13. Подключился через psql к контейнеру pg-server

    sudo docker exec -it pg-server psql postgres postgres

    подключение прошло успешно

14. Сменил пароль пользователю postgres

    ALTER ROLE postgres WITH PASSWORD 'postgresotus2022';

15. Зашёл в контейнер pg-server и установил nano

    sudo docker exec -it pg-server bash

    apt update

    apt install nano

16. Включил шифрование пароля scram-sha-256 в /var/lib/postgresql/data/pg_hba.conf

17. Перезагрузил контейнер

    sudo docker container restart pg-server

18. Зашёл внутрь контейнера pg-client и попытался подключиться к контейнеру pg-server

    sudo docker exec -it pg-client psql -d postgres -h pg-server -U postgres

19. Создал базу данных с именем docker, создал в ней таблицу с именем persons и добавил в неё несколько строк

    CREATE DATABASE docker;

    CREATE TABLE persons (id serial, first_name text, second_name text); 

    INSERT INTO persons (first_name, second_name) VALUES ('ivan', 'ivanov'); 

    INSERT INTO persons (first_name, second_name) VALUES ('petr', 'petrov');

20. Для подключения к контейнеру снаружи в контейнере pg-server (небезопасный метод)

    sudo docker exec -it pg-server bash

    nano /var/lib/postgresql/data/pg_hba.conf

    host    all             all             0.0.0.0/0            scram-sha-256

    nano /var/lib/postgresql/postgresql.conf

    раскомментировал listen_addresses = '*'

    вышел из контейнера

    sudo docker container restart pg-server

    далее в GCP VPC network добавлено правило файервола для открытия снаружи порта 5432 для своего статического IP

21. Со своей локальной машины подключаюсь к инстансу GCP

    psql -d postgres -h 34.123.61.76 -U postgres

    Password for user postgres:

    postgresotus2022

    

    Результат

    psql (11.12, server 14.2 (Debian 14.2-1.pgdg110+1))
    WARNING: psql major version 11, server major version 14.
             Some psql features might not work.
    WARNING: Console code page (866) differs from Windows code page (1251)
             8-bit characters might not work correctly. See psql reference
             page "Notes for Windows users" for details.
    Type "help" for help.

    postgres=#

    

    Далее получаем список баз для проверки

    \l

       Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
    -----------+----------+----------+------------+------------+-----------------------
     docker    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
               |          |          |            |            | postgres=CTc/postgres
     template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
               |          |          |            |            | postgres=CTc/postgres

    

    Список баз корректный

22. Удаляю контейнер pg-server

    sudo docker container stop pg-server

    sudo docker container rm pg-server

23. Создаю контейнер заново

    sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -e PGDATA=/var/lib/postgresql/data -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

24. Получаем список контейнеров

    CONTAINER ID   IMAGE         COMMAND                  CREATED             STATUS             PORTS                                       NAMES
    60fdcc5ea5ea   postgres:14   "docker-entrypoint.s…"   10 seconds ago      Up 8 seconds       0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
    559428b989e6   postgres:14   "docker-entrypoint.s…"   About an hour ago   Up About an hour   5432/tcp                                    pg-client

25. Проверяем что данные остались на месте

    sudo docker exec -ti pg-server psql postgres postgres

    

    \l

                                     List of databases
       Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
    -----------+----------+----------+------------+------------+-----------------------
     docker    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
     template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
               |          |          |            |            | postgres=CTc/postgres
     template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
               |          |          |            |            | postgres=CTc/postgres

    

    \c docker

    \dt+

                                        List of relations
     Schema |  Name   | Type  |  Owner   | Persistence | Access method | Size  | Description
    --------+---------+-------+----------+-------------+---------------+-------+-------------
     public | persons | table | postgres | permanent   | heap          | 16 kB |

    

    SELECT * FROM persons;

     id | first_name | second_name
    ----+------------+-------------
      1 | ivan       | ivanov
      2 | petr       | petrov
    (2 rows)



