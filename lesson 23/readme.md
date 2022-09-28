# **Введение**

Домашнее задание на тему "Хранимые функции и процедуры".

# **Выполнение**

Создадим таблицы в базе

```
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```
А также универсалный триггер.
```
CREATE OR REPLACE function  ft_for_good_sum_mar()
RETURNS trigger
as
$$
declare 
  gname varchar(63);
  gprice numeric(12, 2);
 data_row record;
begin
	--RAISE NOTICE E'\nG_TABLE_NAME = %\nTG_WHEN = %\nTG_OP = %\nTG_LEVEL = %\n -------------', TG_TABLE_NAME, TG_WHEN, TG_OP, TG_LEVEL;
	IF TG_LEVEL = 'ROW' THEN
        CASE TG_OP
            WHEN 'DELETE'
                THEN
                gprice = (select good_price from goods where goods_id = OLD.good_id);
                gname = (select good_name from goods where goods_id = OLD.good_id);
                update good_sum_mart set sum_sale = sum_sale - OLD.sales_qty * gprice where good_name = gname;
               return old;
            WHEN 'UPDATE'
                THEN 
                gprice = (select good_price from goods where goods_id = OLD.good_id);
                gname = (select good_name from goods where goods_id = OLD.good_id);
                update good_sum_mart set sum_sale = sum_sale - OLD.sales_qty * gprice + new.sales_qty *gprice where good_name = gname;
                return new;
            WHEN 'INSERT'
            then
                gprice = (select good_price from goods where goods_id = new.good_id);
                gname = (select good_name from goods where goods_id = new.good_id);
               update good_sum_mart set sum_sale = sum_sale + new.sales_qty * gprice where good_name = gname;
               if not found then
               insert into good_sum_mart (good_name, sum_sale) values (gname, new.sales_qty * gprice);
               end if; 
               return new;
        END CASE;
    END IF;
END;
$$  LANGUAGE plpgsql
  volatile
 
CREATE or replace TRIGGER tr_for_good_sum_mart
AFTER INSERT OR update or delete 
ON sales
FOR EACH ROW
EXECUTE procedure  ft_for_good_sum_mar(); 

```
Заполним данными.

```
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
select * from sales s ;

```
| sales\_id | good\_id | sales\_time | sales\_qty |
| :--- | :--- | :--- | :--- |
| 37 | 1 | 2022-09-28 13:35:11.712218 +00:00 | 10 |
| 38 | 1 | 2022-09-28 13:35:11.712218 +00:00 | 1 |
| 39 | 1 | 2022-09-28 13:35:11.712218 +00:00 | 120 |
| 40 | 2 | 2022-09-28 13:35:11.712218 +00:00 | 1 |
```
select * from good_sum_mart gsm; 
```
| good\_name | sum\_sale |
| :--- | :--- |
| Спички хозайственные | 65.50 |
| Автомобиль Ferrari FXX K | 185000000.01 |

Данны были заполнены при инсерте.

Удалим продажу
```
delete from sales where sales_id  =49;
```
| good\_name | sum\_sale |
| :--- | :--- |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозайственные | 60.50 |

Данные обновились триггером.

Обновим количество.
```
update sales set sales_qty =20 where sales_id = 50
```
| good\_name | sum\_sale |
| :--- | :--- |
| Автомобиль Ferrari FXX K | 185000000.01 |
| Спички хозайственные | 70.00 |

Данные обновились триггером.

Таблица витрина обновляется автоматически триггером.





