1. Создадим тестовую базу trigges и таблицы в ней

   CREATE DATABASE triggers;

   DROP SCHEMA IF EXISTS pract_functions CASCADE;
   CREATE SCHEMA pract_functions;

   SET search_path = pract_functions, publ

   -- товары:
   CREATE TABLE goods
   (
       goods_id    integer PRIMARY KEY,
       good_name   varchar(63) NOT NULL,
       good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
   );
   INSERT INTO goods (goods_id, good_name, good_price) VALUES 	(1, 'Спички хозайственные', .50), (2, 'Автомобиль Ferrari FXX K', 185000000.01);

   -- Продажи
   CREATE TABLE sales
   (
       sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
       good_id     integer REFERENCES goods (goods_id),
       sales_time  timestamp with time zone DEFAULT now(),
       sales_qty   integer CHECK (sales_qty > 0)
   );

   INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

   CREATE TABLE good_sum_mart (good_name   varchar(63) NOT NULL, sum_sale	numeric(16, 2) NOT NULL);

2. Заполним таблицу good_sum_mart уже имеющимися записями из таблицы sales

   INSERT INTO good_sum_mart SELECT goods.good_name, sum(goods.good_price * sales.sales_qty) FROM goods INNER JOIN sales ON sales.good_id = goods.goods_id GROUP BY goods.good_name;

   

   Проверим данные в таблице good_sum_mart 

   SELECT * FROM good_sum_mart;

   

           good_name         |   sum_sale
   --------------------------+--------------
    Автомобиль Ferrari FXX K | 185000000.01
    Спички хозайственные     |        65.50

3. Создадим представление good_sum_mart_view

   CREATE VIEW good_sum_mart_view AS SELECT goods.good_name, sum(goods.good_price * sales.sales_qty) FROM goods INNER JOIN sales ON sales.good_id = goods.goods_id GROUP BY goods.good_name;

   

   Для начала создадим триггер на добавление записи со стоимостью либо добавление новой записи в таблицу со значением записи

   CREATE OR REPLACE function ft_insert_sales()
   RETURNS trigger
   AS
   $BODY$
   DECLARE
   good_name varchar(63);
   good_price numeric(12,2);
   BEGIN
   SELECT goods.good_name, goods.good_price * NEW.sales_qty INTO good_name, good_price FROM goods 		WHERE  goods.goods_id = NEW.good_id;
   IF EXISTS (SELECT from good_sum_mart WHERE good_sum_mart.good_name = good_name)
   THEN UPDATE good_sum_mart SET sum_sale = sum_sale + good_price WHERE 											good_sum_mart.good_name = good_name;
   ELSE INSERT INTO good_sum_mart (good_name, sum_sale) VALUES (good_name, good_price);
   END IF;
   RETURN NEW;
   END;
   $BODY$
   LANGUAGE plpgsql
   VOLATILE
   SET search_path = pract_functions, public
   COST 50;

   

   При удалении следует вычитание текущей стоимости записи с последующим удалением строк, у которых стоимость меньше либо равна 0

   CREATE OR REPLACE FUNCTION ft_delete_sales()
   RETURNS trigger
   AS
   $BODY$
   DECLARE
   good_name varchar(63);
   good_price numeric(12,2););
   BEGIN
   SELECT goods.good_name, goods.good_price * OLD.sales_qty INTO good_name, good_price FROM goods WHERE goods.goods_id = OLD.good_id;
   IF EXISTS(SELECT from good_sum_mart WHERE from good_sum_mart.good_name = good_name)
   THEN 
   UPDATE good_sum_mart SET sum_sale = sum_sale - good_price WHERE good_sum_mart.good_name = good_name;
   DELETE FROM good_sum_mart WHERE good_sum_mart.good_name = good_name and (sum_sale < 0 or sum_sale = 0);
   END IF;
   RETURN NEW;
   END;
   $BODY$
   LANGUAGE plpgsql
   VOLATILE
   SET search_path = pract_functions, public
   COST 50;

   

   При обновлении следуют обе процедуры описанные выше.

   CREATE OR REPLACE FUNCTION ft_update_sales()
   RETURNS trigger
   AS
   $BODY$
   DECLARE
   good_name_old varchar(63);
   good_price_old numeric(12,2);
   good_name_new varchar(63);
   good_price_new numeric(12,2);
   BEGIN
   SELECT goods.good_name, goods.good_price * OLD.sales_qty INTO good_name_old, good_price_old FROM goods WHERE goods.goods_id = OLD.good_id;
   SELECT goods.good_name, goods.good_price * NEW.sales_qty INTO good_name_new, good_price_new FROM goods G WHERE goods.goods_id = NEW.good_id;
   IF EXISTS(SELECT FROM good_sum_mart WHERE good_sum_mart .good_name = good_name_new)
   THEN UPDATE good_sum_mart SET sum_sale = sum_sale + good_price_new where good_sum_mart .good_name = good_name_new;
   ELSE INSERT INTO good_sum_mart (good_name, sum_sale) VALUES (good_name_new, good_price_new);
   END IF;
   IF EXISTS(SELECT FROM good_sum_mart WHERE good_sum_mart .good_name = good_name_old)
   THEN 
   UPDATE good_sum_mart SET sum_sale = sum_sale - good_price_old WHERE good_sum_mart .good_name = good_name_old;
   DELETE FROM good_sum_mart WHERE good_sum_mart .good_name = good_name_old and (sum_sale < 0 or sum_sale = 0);
   END IF;
   RETURN NEW;
   END;
   $BODY$
   LANGUAGE plpgsql
   VOLATILE
   SET search_path = pract_functions, public
   COST 50;

   

   Добавим триггеры

   CREATE TRIGGER tr_insert_sales
   AFTER INSERT
   ON sales
   FOR EACH ROW
   EXECUTE PROCEDURE ft_insert_sales();

   CREATE TRIGGER tr_delete_sales
   AFTER DELETE
   ON sales
   FOR EACH ROW
   EXECUTE PROCEDURE ft_delete_sales();

   CREATE TRIGGER tr_update_sales
   AFTER UPDATE
   ON sales
   FOR EACH ROW
   EXECUTE PROCEDURE ft_update_sales();

   
