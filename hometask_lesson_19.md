1. Создадим базу в кластере 

   CREATE DATABASE test;

2. Выйдем из psql и скачаем шаблонную базу в домашнюю директорию

   cd ~/

    sudo wget --quiet https://www.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip

3. Установим архиватор для распоковки ZIP архивов в среде Linux и распакуем архив с базой

   sudo apt install unzip

   sudo unzip dvdrental.zip

4. Утилитой pg_restore восстановим базу из скачанного архива

   sudo pg_restore -U postgres -d test dvdrental.tar

5. Подключимся к базе test и посмотрим список таблиц

   psql test postgres

   \dt+

                                            List of relations
    Schema |     Name      | Type  |  Owner   | Persistence | Access method |    Size    | Description
   --------+---------------+-------+----------+-------------+---------------+------------+-------------
    public | actor         | table | postgres | permanent   | heap          | 40 kB      |
    public | address       | table | postgres | permanent   | heap          | 88 kB      |
    public | category      | table | postgres | permanent   | heap          | 8192 bytes |
    public | city          | table | postgres | permanent   | heap          | 64 kB      |
    public | country       | table | postgres | permanent   | heap          | 8192 bytes |
    public | customer      | table | postgres | permanent   | heap          | 96 kB      |
    public | film          | table | postgres | permanent   | heap          | 464 kB     |
    public | film_actor    | table | postgres | permanent   | heap          | 272 kB     |
    public | film_category | table | postgres | permanent   | heap          | 72 kB      |
    public | inventory     | table | postgres | permanent   | heap          | 232 kB     |
    public | language      | table | postgres | permanent   | heap          | 8192 bytes |
    public | payment       | table | postgres | permanent   | heap          | 896 kB     |
    public | rental        | table | postgres | permanent   | heap          | 1232 kB    |
    public | staff         | table | postgres | permanent   | heap          | 16 kB      |
    public | store         | table | postgres | permanent   | heap          | 8192 bytes |

   

6. Посмотрим самую большую по объёму таблицу rental.

   Выберем из таблицы все записи и подсчитаем число строк.

   \x

   SELECT count(*) FROM rental;

   В таблице 16044 строки

   -[ RECORD 1 ]
   count | 16044

7. Посмотрим сколько времени занимает выборка всех данных из таблицы (в боевой среде так делать нельзя)

   \timing

   SELECT * FROM rental;

   Time: 16.843 ms

8. Сделаем выборку для клиента с id=333

    SELECT count(*) FROM rental WHERE customer_id = 333;

   -[ RECORD 1 ]
   count | 27

   Time: 1.036 ms

9. Создадим простой индекс по таблице по колонкам customer_id и rental_id

   CREATE INDEX idx_rental_customer_id ON rental(customer_id);

   CREATE INDEX idx_rental_id ON rental(rental_id);

10. Проведём повторную выборку по записям с customer_id = 333;

    SELECT count(*) FROM rental WHERE customer_id = 333;

    -[ RECORD 1 ]
    count | 27

    Time: 0.581 ms

11. Проверим, что использует планировщик запроса

                                               QUERY PLAN
    -------------------------------------------------------------------------------------------------
     Aggregate  (cost=1.89..1.90 rows=1 width=8)
       ->  Index Only Scan using idx_rental_customer_id on rental  (cost=0.29..1.82 rows=25 width=0)
             Index Cond: (customer_id = 333)
    (3 rows)

    Time: 0.718 ms

    Выполнено сканирование по индексу  idx_rental_customer_id а не последовательное чтение блок за блоком.

    При этом затраты на получение первой строки составили 0.29, затраты на получение всех строк 1.82

12. Удалим индекс и повторим запрос

    DROP INDEX idx_rental_customer_id;

    EXPLAIN SELECT count(*) FROM rental WHERE customer_id = 333;

                                                               QUERY PLAN
    --------------------------------------------------------------------------------------------------------------------------------
     Aggregate  (cost=191.33..191.34 rows=1 width=8)
       ->  Index Only Scan using idx_unq_rental_rental_date_inventory_id_customer_id on rental  (cost=0.29..191.27 rows=25 width=0)
             Index Cond: (customer_id = 333)
    (3 rows)

    Использовался ещё один индекс idx_unq_rental_rental_date_inventory_id_customer_id

    Удалим его тоже

    DROP INDEX idx_unq_rental_rental_date_inventory_id_customer_id;

13. Повторим запрос

                              QUERY PLAN
    ---------------------------------------------------------------
     Aggregate  (cost=350.61..350.62 rows=1 width=8)
       ->  Seq Scan on rental  (cost=0.00..350.55 rows=25 width=0)
             Filter: (customer_id = 333)
    (3 rows)

    "Стоимость" выполнения запроса без наличия индекса значительно выросла.

    350 без индекса против 1.82 с индексом при одинаковом количестве записей в выборке

14. Создадим индекс для полнотекстового поиска

    Для этого будем использовать таблицу address

    Сделаем выборку из таблицы БЕЗ индекса

    \x

    EXPLAIN SELECT * FROM address;

    -[ RECORD 1 ]---------------------------------------------------------
    QUERY PLAN | Seq Scan on address  (cost=0.00..14.03 rows=603 width=61)

    Выполнился перебор таблицы для поиска.

    Наложим дополнительное условие поиска и найдём название адресов, содержащих boulevard;

15. Выполним запрос по таблице с отбором

     EXPLAIN SELECT * FROM address WHERE address LIKE '%Boulevard%';

                                       QUERY PLAN
    ---------------------------------------------------------------------------------
     Aggregate  (cost=317.04..317.05 rows=1 width=8)
       ->  Seq Scan on address  (cost=0.00..317.04 rows=3 width=0)
             Filter: (to_tsvector((address)::text) @@ to_tsquery('Boulevard'::text))
    (3 rows)

16. Создадим индекс и повторим запрос

    CREATE INDEX idx_address_gin ON address USING GIN (to_tsvector('english', address));

    EXPLAIN SELECT count(*) FROM address WHERE to_tsvector('english', address) @@ to_tsquery('english','Boulevard');

                                                     QUERY PLAN
    ------------------------------------------------------------------------------------------------------------
     Aggregate  (cost=6.13..6.14 rows=1 width=8)
       ->  Bitmap Heap Scan on address  (cost=2.22..6.13 rows=3 width=0)
             Recheck Cond: (to_tsvector('english'::regconfig, (address)::text) @@ '''boulevard'''::tsquery)
             ->  Bitmap Index Scan on idx_address_gin  (cost=0.00..2.22 rows=3 width=0)
                   Index Cond: (to_tsvector('english'::regconfig, (address)::text) @@ '''boulevard'''::tsquery)
    (5 rows)SELECT 

17. Создание частичного индекса

    Создадим обычный индекс для таблицы rental по колю rental_id где ID больше 8000

    CREATE INDEX idx_inventory_id ON rental(rental_id) WHERE inventory_id > 3000;

    Удалим существующие индексы

    DROP INDEX idx_fk_inventory_id;

    DROP INDEX  idx_unq_rental_rental_date_inventory_id_customer_id;

    Протестируем работу индекса сделав запрос по таблице наложив условие получения записей с inventory_id в диапазоне от 1 до 2999

    EXPLAIN SELECT count(*) FROM rental WHERE inventory_id BETWEEN 1 AND 2999;

                                QUERY PLAN
    ------------------------------------------------------------------
     Aggregate  (cost=416.98..416.99 rows=1 width=8)
       ->  Seq Scan on rental  (cost=0.00..390.66 rows=10528 width=0)
             Filter: ((inventory_id >= 1) AND (inventory_id <= 2999))
    (3 rows)
    
    Протестируем работу индекса сделав запрос по таблице наложив условие получения записей с inventory_id в диапазоне от 3001;
    
    EXPLAIN SELECT count(*) FROM rental WHERE inventory_id >= 3001;
    
                                            QUERY PLAN
    ------------------------------------------------------------------------------------------
     Aggregate  (cost=182.51..182.52 rows=1 width=8)
       ->  Index Scan using idx_inventory_id on rental  (cost=0.28..168.73 rows=5512 width=0)
             Filter: (inventory_id >= 3001)
    (3 rows)
    
    Выполним запрос по всей таблице
    
    EXPLAIN SELECT count(*) FROM rental WHERE inventory_id >= 1;
    
                                QUERY PLAN
    ------------------------------------------------------------------
     Aggregate  (cost=390.66..390.67 rows=1 width=8)
       ->  Seq Scan on rental  (cost=0.00..350.55 rows=16042 width=0)
             Filter: (inventory_id >= 1)
    (3 rows)
    
    Так как индекса нет, было выполнено сканирование таблицы.
    
    
    
    
    
    
    
    
    
    







