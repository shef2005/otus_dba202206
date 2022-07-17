Исходный  текст ДЗ: 

```
Домашнее задание
Работа с уровнями изоляции транзакции в PostgreSQL

Цель:

1. научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
 научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read
 Описание/Пошаговая инструкция выполнения домашнего задания:
 создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере , например postgres2022-, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально на уровне GCP)
 далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами
 добавить свой ssh ключ в GCE metadata
 зайти удаленным ssh (первая сессия), не забывайте про ssh-add

 поставить PostgreSQL
 зайти вторым ssh (вторая сессия)

 2. запустить везде psql из под пользователя postgres для обоих сессий
 3. выключить везде auto commit
 4. сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
 5. посмотреть текущий уровень изоляции: show transaction isolation level
 6. начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
 7. в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
 8. сделать select * from persons во второй сессии
 9. видите ли вы новую запись и если да то почему?
10. завершить первую транзакцию - commit;
11. сделать select * from persons во второй сессии
12. видите ли вы новую запись и если да то почему?
13. завершите транзакцию во второй сессии

14. начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
15. в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
16. сделать select * from persons во второй сессии
17. видите ли вы новую запись и если да то почему?
18. завершить первую транзакцию - commit;
19. сделать select * from persons во второй сессии
20. видите ли вы новую запись и если да то почему?
21. завершить вторую транзакцию
22. сделать select * from persons во второй сессии
23. видите ли вы новую запись и если да то почему? 

ДЗ сдаем в виде миниотчета в markdown в гите

Критерии оценки:
Критерии оценивания:
Выполнение ДЗ: 10 баллов

+2 балл за красивое решение
-2 балл за рабочее решение, и недостатки указанные преподавателем не устранены
```


# Решение ДЗ: Работа с уровнями изоляции транзакции в PostgreSQL
Принятые обозначения:  
1. Исходные строки задания ДЗ пронумерованы и перенесены в решение. 
1. Выполняемые команды выводятся жирным текстом без приглашения. Пример: __psql__
1. Ответы системы выводятся обычным текстом. Пример:
```
   CREATE DATABASE
```

# 1 Установка Oracle VM VirtualBox, PostgreSQL
Была выполнена согласно инструкции: [Установка PostgreSQL на Ubuntu VirtualBox](https://virtual-dba.com/blog/installing-postgresql-ubuntu-virtualbox/)

Запуск VM -> делаем запуск Oracle VM VirtualBox и выбираем установленную машину

# 2 Запустить везде psql из под пользователя postgres. 

## 2.1 Запуск первого терминала (Т1) - сессия 1 
• Запуск первого терминала

__ctrl+alt+t__

• Запуск psql из под пользователя postgres

__sudo su postgres__
 
__psql__

## 2.2 Запуск второго терминала (Т2)  сессия 2 
• Запуск второго терминала

__ctrl+alt+t__

• Запуск psql из под пользователя postgres

__sudo su postgres__
 
__psql__


## 2.3 Доп команды для Т1, T2
• Создание базы l2 -- добавлена только для T1 !!!
 
__CREATE DATABASE l2;__

>CREATE DATABASE

• Установка соединения с базой -- добавлена -- добавлена для T1, T2 !!!!!!!

__\c l2;__

>You are now connected to database "l2" as user "postgres".

## 3 Выключить везде auto commit для T1 и Т2
 
• Выключить auto commit   -- добавил вывод значения :AUTOCOMMIT !

__\set AUTOCOMMIT off__ 

__\echo :AUTOCOMMIT__

>off

## 4 Cделать в T1 новую таблицу и наполнить ее данными  

__create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;__

```
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```

## 5 Посмотреть  в T1 текущий уровень изоляции: show transaction isolation level
• посмотреть текущий уровень изоляции: show transaction isolation level

__show transaction isolation level;__
```
 transaction_isolation 
-----------------------
 read committed
(1 row)

```

## 6 Начать новую транзакцию в _обоих_ сессиях с дефолтным (не меняя) уровнем изоляции


__begin;__
>BEGIN


## 7 В первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');

__insert into persons(first_name, second_name) values('sergey', 'sergeev');__
>INSERT 0 1

## 8 Сделать select * from persons во второй сессии

__select * from persons;__
```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

## 9 Видите ли вы новую запись и если да то почему?

__Нет. Не было _commit_ в первой сессии.__

## 10 Завершить первую транзакцию - commit;

__commit;__
>COMMIT

## 11 Cделать select * from persons во второй сессии

__select * from persons;__

```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

## 12 Видите ли вы новую запись и если да, то почему?
    Да. Был transaction_isolation=read committed и был commit в первой сессии;
	
## 13 Завершите транзакцию во второй сессии
end;
>COMMIT


## 14 Начать новые, но уже repeatable read транзации - set transaction isolation level repeatable read;

__BEGIN ISOLATION LEVEL REPEATABLE READ;__
>BEGIN

__BEGIN ISOLATION LEVEL REPEATABLE READ;__
>BEGIN

## 15 В первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
__insert into persons(first_name, second_name) values('sveta', 'svetova');__
>INSERT 0 1

## 16 Сделать select * from persons во второй сессии

select * from persons;
```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

## 17 Видите ли вы новую запись и если да. то почему?
   Нет. 
   Причины: 
     - Еще не было commit в T1 
     - ISOLATION LEVEL=REPEATABLE READ для T2;  -- было установлено повторяющееся чтение для уровня REPEATABLE READ

## 18 Завершить первую транзакцию - commit;
__commit;__
>COMMIT

## 19 Cделать select * from persons во второй сессии
__select * from persons;__

```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

## 20 Видите ли вы новую запись и если да, то почему?
Нет. commit прошел, но ISOLATION LEVEL=REPEATABLE READ для T2; 
-- Для T2 было установлен уровень REPEATABLE READ (требование повторяющегося чтения). 

## 21 Завершить вторую транзакцию
__end;__
>COMMIT

## 22 Сделать select * from persons во второй сессии
```
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

## 23 Видите ли вы новую запись и если да, то почему? 
Да. Это уже новая транзакция в T2 и данные в T1 сохранены (commited)
