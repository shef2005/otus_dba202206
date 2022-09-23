Домашнее задание 23
Триггеры, поддержка заполнения витрин

# 1 Постановка задачи

По тексту постановки задачи даются мои уточнения - укрупненное решение задачи. 

```
уточнения
```

Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
В БД создана структура, описывающая товары (таблица **goods**) и продажи (таблица **sales**).
Есть запрос для генерации отчета – сумма продаж по каждому товару. 

БД была денормализована, создана таблица-витрина ( **good_sum_mart**), структура которой повторяет структуру отчета.

**Задача** 

Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

```
Решение создать триггер, запускаемый при совершении продаж (sales): 

trigger tr_sales_
after insert or update or delete on sales 
for each row execute function tr_sales()

Реализация триггера выполняется функцией tr_sales при изменении записей таблицы sales
```

Подсказка1: не забыть, что кроме INSERT есть еще UPDATE и DELETE

```
У нас триггер типа after insert or update or delete.
Встроенная переменная TG_OP содержит операцию на таблицей.
Это дает возможность написать универсальную функцию для любого оператора INSERT/UPDATE/DELETE
```

Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?

```
Актуальность данных на момент внесения изменений.
Недостаток такого решения: 
- необходимые ресурсы для хранения данных.  
- повышенная трудоемкость кода триггера по сравнению с кодом запроса
```

Подсказка2: В реальной жизни возможны изменения цен.

```
Варианты решения
1 В предложенной схеме время продажи есть в таблице  sales_time.
  А таблица goods не содержит времени изменения цен на товар. 
  Надо менять схему таблиц. Это качественно другая задача.
2 Полный пересчет измененной позиции.
3 Создание доп триггера для таблицы goods. Похожего на реализуемый.

В Коде предусмотрено решение по варианту 2 
```



# 2 Решение Задачи

```
Ниже в разделе 2 приводится детальное решение.
Оно содержит код функций: 
tr_sales - триггера на таблице продаж (sales)
update_good_sum_mart_incr - функция инкременального расчета для поля sum_sale таблицы good_sum_mart в разрезе good_name
update_good_sum_mart_full - функция полного расчета для поля sum_sale  таблицы good_sum_mart  в разрезе good_name

Делается проверка/отладка решения.
```

## 2.1 Подготовка данных согласно hw_triggers.sql

### 2.1.1  Товары

CREATE TABLE if not exists goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price) 

VALUES 	(1, 'Спички хозяйственные', .50), 

(2, 'Автомобиль Ferrari FXX K', 185000000.01);

### 2.1.2 Продажи

CREATE TABLE if not exists sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

### 2.1.3 Витрина good_sum_mart

CREATE table if not exists good_sum_mart (good_name   varchar(63) NOT NULL, sum_sale	numeric(16, 2) NOT NULL);

INSERT INTO good_sum_mart
SELECT 
   g.good_name, sum(g.good_price * s.sales_qty) 
  FROM goods g
    join sales s ON s.good_id=g.goods_id
GROUP BY g.good_name
;

## 2.2 Решение

-- Создать триггер  на таблице sales.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

### 2.2.1 Создание функции **tr_sales** 

```
CREATE OR REPLACE FUNCTION tr_sales() returns trigger
LANGUAGE plpgsql
AS $function$
-- 22 сент. 2022 г.   18:37:09
declare
   v_good_id           integer;   -- ссылка на v_good_id
   v_dif_sales_qty     numeric;   -- изменение количества продаж
   v_good_name         varchar;   -- наименование
   v_good_price        numeric;   -- Цена 
   v_rowcount          bigint;    -- число обновленных 
begin
   -- Определим v_good_id и изменение количества продаж sales_qty
   --raise notice e'\n'; 
   --raise notice 'TG_OP=%' ,TG_OP;
   CASE
   -- Изменение good_id	в sales будем считать ошибкой
   when TG_OP='UPDATE' and OLD.good_id<>NEW.good_id then
      raise exception $x$Изменение good_id. OLD.good_id=% <> NEW.good_id=% $x$, OLD.good_id ,NEW.good_id;
   WHEN TG_OP='DELETE' THEN 
      v_good_id=OLD.good_id;  
      v_dif_sales_qty=OLD.sales_qty*(-1);
   WHEN TG_OP='INSERT' THEN 
      v_good_id=NEW.good_id;
      v_dif_sales_qty=NEW.sales_qty*(+1);
   WHEN TG_OP='UPDATE' THEN  
     v_good_id=NEW.good_id;
     v_dif_sales_qty=NEW.sales_qty-OLD.sales_qty; -- coalesce не пишем так как есть constraint sales_qty>0 
   ELSE  
      RAISE EXCEPTION 'invalid good_id';   -- 
   END CASE
   ;    
   --raise notice 'v_good_id=% v_dif_sales_qty=%' ,v_good_id, v_dif_sales_qty;
   --
   -- Определим v_good_name, v_good_price
   select g.good_name, g.good_price 
     into v_good_name, v_good_price -- цена могла измениться ??? 
     from goods g
    where 1=1
      and goods_id=v_good_id 
   ;
   --raise notice 'v_good_name=%, v_good_price=%' ,v_good_name, v_good_price;
   --
   -- Вариант 1 - Инкремент
   --perform (TG_OP ,v_good_name ,v_good_price, v_dif_sales_qty);
   -- Вариант 2 - Полный пересчет позиции
   perform update_good_sum_mart_full (v_good_name);
   return new;
end
$function$
;
```



### 2.2.2 Создание функции **update_good_sum_mart_incr** 

-- Расчет по изменениям

drop FUNCTION if exists update_good_sum_mart_incr;

CREATE OR REPLACE FUNCTION update_good_sum_mart_incr 
(TG_OP varchar ,v_good_name varchar ,v_good_price numeric, v_dif_sales_qty integer) 
RETURNS void
AS $function$
-- 23 сент. 2022 г.   12:53:41
declare
   v_rowcount bigint;
begin
   v_rowcount=0;
   if TG_OP='UPDATE' or TG_OP='DELETE' or TG_OP='INSERT' then 
       update good_sum_mart v
          set sum_sale=sum_sale+v_good_price * v_dif_sales_qty
        where good_name=v_good_name
       ;   

​       GET DIAGNOSTICS v_rowcount = ROW_COUNT;
   end if;
   -- raise notice 'v_rowcount=%', v_rowcount;
   if v_rowcount=0 and TG_OP='INSERT' then 
​      insert into good_sum_mart(good_name, sum_sale)
​      values (v_good_name, v_good_price * v_dif_sales_qty);
   end if;
END
$function$
LANGUAGE plpgsql
;

### 2.2.3 Создание функции **update_good_sum_mart_full**

-- Полный пересчет одного товара

```
drop FUNCTION if exists update_good_sum_mart_full;

CREATE OR REPLACE FUNCTION update_good_sum_mart_full (v_good_name varchar) 
RETURNS void
AS $function$
-- 23 сент. 2022 г.   12:53:41
declare
   v_rowcount bigint;
begin
   raise notice 'v_good_name=%' ,v_good_name;
   delete from good_sum_mart where good_name=v_good_name;
   --
   raise notice 'v_good_name=%' ,v_good_name;
   insert into good_sum_mart(good_name, sum_sale)
   SELECT 
       g.good_name, sum(g.good_price * s.sales_qty) 
      FROM goods g
        join sales s ON s.good_id=g.goods_id
     where g.good_name=v_good_name  
    GROUP BY g.good_name
    having sum(g.good_price * s.sales_qty)>0
    ;
END
$function$
LANGUAGE plpgsql
;
```

2.2.4 Привязка триггера к таблице tr_sales

```
create trigger tr_sales_
after insert or update or delete on sales 
for each row execute function tr_sales()
; 
```

## 2.3 Проверка решения

### 2.3.1 Перезагрузка данных

```
ALTER TABLE sales DISABLE TRIGGER tr_sales_;

truncate table goods cascade;
truncate table sales cascade;
truncate table  good_sum_mart  cascade;

INSERT INTO goods (goods_id, good_name, good_price) 
VALUES  (1, 'Спички хозяйственные', .50), 
(2, 'Автомобиль Ferrari FXX K', 185000000.01);

insert INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

INSERT INTO good_sum_mart
SELECT 
   g.good_name, sum(g.good_price * s.sales_qty) 
  FROM goods g
    join sales s ON s.good_id=g.goods_id
GROUP BY g.good_name
;

select * from goods;
select * from sales;
select * from good_sum_mart;
------------
ALTER TABLE sales enable TRIGGER tr_sales_;
```

### 2.3.2 Продажа товара

*До*

**goods**

| goods_id | good_name                | good_price   |
| -------- | ------------------------ | ------------ |
| 1        | Спички хозяйственные     | 0.50         |
| 2        | Автомобиль Ferrari FXX K | 185000000.01 |

**sales**

| sales_id | good_id | sales_time                    | sales_qty |
| -------- | ------- | ----------------------------- | --------- |
| 31       | 1       | 2022-09-23 15:00:18.021 +0300 | 10        |
| 32       | 1       | 2022-09-23 15:00:18.021 +0300 | 1         |
| 33       | 1       | 2022-09-23 15:00:18.021 +0300 | 120       |
| 34       | 2       | 2022-09-23 15:00:18.021 +0300 | 1         |

**good_sum_mart**

| good_name                | sum_sale     |
| ------------------------ | ------------ |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозяйственные     | 65.50        |

*После команды*

**INSERT** INTO sales (good_id, sales_qty) VALUES (1, 5);

select  *
 from bookings.good_sum_mart
; 

| good_name                | sum_sale     |
| ------------------------ | ------------ |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозяйственные     | 68           |

68=65.5+5*0.5

*После команды*

update sales set sales_qty=3
where good_id=1  and sales_qty=5
;

| good_name                | sum_sale     |
| ------------------------ | ------------ |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозяйственные     | 67.00        |

*После команды*

delete from sales 
where good_id=1
  and sales_qty=3
;

| good_name                | sum_sale     |
| ------------------------ | ------------ |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозяйственные     | 65.50        |

**2.3.3 Примечание**

При исправлении триггера использовал команды

```
ALTER TABLE sales enable TRIGGER tr_sales_;

ALTER TABLE sales DISABLE TRIGGER tr_sales_;
```



