1. Подключимся на сервер, скачаем демо-базу в домашний каталог и разархивируем её

   cd ~

   sudo wget https://edu.postgrespro.ru/demo_big.zip

   sudo unzip demo_big.zip

   ls

   demo_big.sql

2. Подключимся к кластеру и развернём базу

   psql postgres postgres 

   \i ~/demo_big.sql

   \l+ demo

                                               List of databases
    Name |  Owner   | Encoding | Collate |  Ctype  | Access privileges |  Size   | Tablespace | Description
   ------+----------+----------+---------+---------+-------------------+---------+------------+-------------
    demo | postgres | UTF8     | C.UTF-8 | C.UTF-8 |                   | 2639 MB | pg_default |

3. Подключимся к базе demo и проверим состав таблиц

   \с demo

   \dt+

   Did not find any relations.

   Просмотрим список схем

   \dn

      List of schemas
      Name   |  Owner
   ----------+----------
    bookings | postgres
    public   | postgres

   Появилась ещё одна схема bookings.

   Проверим какие схемы участвуют в поиске

   \x

   SHOW search_path;

   -[ RECORD 1 ]----------------
   search_path | "$user", public

   Добавим схему bookings в search_path 

   SET search_path TO "$user", public, bookings;

   Повторим получение списка таблиц

   \x

   \dt+

                                               List of relations
     Schema  |      Name       | Type  |  Owner   | Persistence | Access method |  Size  |    Description
   ----------+-----------------+-------+----------+-------------+---------------+--------+-------------------
    bookings | aircrafts       | table | postgres | permanent   | heap          | 16 kB  | Самолеты
    bookings | airports        | table | postgres | permanent   | heap          | 48 kB  | Аэропорты
    bookings | boarding_passes | table | postgres | permanent   | heap          | 455 MB | Посадочные талоны
    bookings | bookings        | table | postgres | permanent   | heap          | 105 MB | Бронирования
    bookings | flights         | table | postgres | permanent   | heap          | 21 MB  | Рейсы
    bookings | seats           | table | postgres | permanent   | heap          | 96 kB  | Места
    bookings | ticket_flights  | table | postgres | permanent   | heap          | 547 MB | Перелеты
    bookings | tickets         | table | postgres | permanent   | heap          | 386 MB | Билеты

   Согласно полученным данным самая большая таблица ticket_flights. Её и будем секционировать

4. Посмотрим на структуру полей таблицы ticket_flights.

   \d+ ticket_flights

                                                       Table "bookings.ticket_flights"
        Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description
   -----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+---------------------
    ticket_no       | character(13)         |           | not null |         | extended |             |              | Номер билета
    flight_id       | integer               |           | not null |         | plain    |             |              | Идентификатор рейса
    fare_conditions | character varying(10) |           | not null |         | extended |             |              | Класс обслуживания
    amount          | numeric(10,2)         |           | not null |         | main     |             |              | Стоимость перелета

5. Посмотрим на содержимое полей сделав запрос первых 30 строк из таблицы

   SELECT * FROM ticket_flights ORDER BY ticket_no ASC LIMIT 30;

      ticket_no   | flight_id | fare_conditions |  amount
   ---------------+-----------+-----------------+----------
    0005432000284 |    187662 | Economy         |  6200.00
    0005432000285 |    187570 | Business        | 18500.00
    0005432000286 |    187570 | Economy         |  6200.00
    0005432000287 |    187728 | Economy         |  6200.00
    0005432000288 |    187728 | Business        | 18500.00
    0005432000289 |    187754 | Economy         |  6200.00
    0005432000290 |    187754 | Economy         |  6200.00
    0005432000291 |    187506 | Business        | 18500.00
    0005432000292 |    187506 | Economy         |  6200.00
    0005432000293 |    187625 | Economy         |  6200.00
    0005432000294 |    187625 | Economy         |  6800.00
    0005432000295 |    187625 | Economy         |  6200.00
    0005432000296 |    187600 | Economy         |  6200.00
    0005432000297 |    187600 | Business        | 18500.00
    0005432000298 |    187741 | Economy         |  6200.00
    0005432000299 |    187652 | Economy         |  6200.00
    0005432000300 |    187652 | Economy         |  6200.00
    0005432000301 |    187801 | Economy         |  6200.00
    0005432000302 |    187801 | Economy         |  6200.00
    0005432000303 |    187801 | Economy         |  6200.00
    0005432000304 |    187801 | Economy         |  6200.00
    0005432000305 |    187686 | Business        | 18500.00
    0005432000306 |    187602 | Economy         |  6200.00
    0005432000307 |    187602 | Economy         |  6200.00
    0005432000308 |    187613 | Economy         |  6200.00
    0005432000309 |    187613 | Economy         |  6200.00
    0005432000310 |    187590 | Economy         |  6200.00
    0005432000311 |    187590 | Economy         |  6200.00
    0005432000312 |    187493 | Economy         |  6200.00
    0005432000313 |    187493 | Economy         |  6200.00

6. Сделаем подсчёт количества строк в таблице 

   SELECT count(*) FROM ticket_flights;

   -[ RECORD 1 ]--
   count | 8391852

7. Проверим возможные значения в колонке fare_conditions

   -[ RECORD 1 ]---+---------
   fare_conditions | Business
   -[ RECORD 2 ]---+---------
   fare_conditions | Comfort
   -[ RECORD 3 ]---+---------
   fare_conditions | Economy

   По данной колонке можно применить секционирование по списку значений.

8. Выполним секционирование

   CREATE TABLE fare_conditions_business PARTITION OF ticket_flights FOR VALUES IN ('Business');

   Получим ошибку

   "ticket_flights" is not partitioned

   Сначала надо сделать секционированную таблицу, которая по составу полей будет являться копией текущей.

   Далее возможны 2 сценария

   - Мы делаем текущю таблицу секцией новой, но в этом особого смысла нет.

   - Мы сделаем новую таблицу и перельём в неё данные из такущей и далее разобьём данные на секции

     Второй вариант выглядит логичнее

9. Создадим новую таблицу ticket_flights_part по структуре копию таблицы ticket_flights, создадим секции и перенсём в неё данные из таблицы  ticket_flights

   CREATE TABLE bookings.ticket_flights_part (LIKE ticket_flights) PARTITION BY LIST (fare_conditions);

   

   Проверим состав полей в обоих таблицах

   \d+ ticket_flights

                                                       Table "bookings.ticket_flights"
        Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description
   -----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+---------------------
    ticket_no       | character(13)         |           | not null |         | extended |             |              | Номер билета
    flight_id       | integer               |           | not null |         | plain    |             |              | Идентификатор рейса
    fare_conditions | character varying(10) |           | not null |         | extended |             |              | Класс обслуживания
    amount          | numeric(10,2)         |           | not null |         | main     |             |              | Стоимость перелета
   Indexes:
       "ticket_flights_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
   Check constraints:
       "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
       "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
   Foreign-key constraints:
       "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
       "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)

   

   \d+ ticket_flights_part

                                           Partitioned table "bookings.ticket_flights_part"
        Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
   -----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
    ticket_no       | character(13)         |           | not null |         | extended |             |              |
    flight_id       | integer               |           | not null |         | plain    |             |              |
    fare_conditions | character varying(10) |           | not null |         | extended |             |              |
    amount          | numeric(10,2)         |           | not null |         | main     |             |              |
   Partition key: LIST (fare_conditions)
   Number of partitions: 0

   Создадим секции для таблицы ticket_flights_part

   CREATE TABLE bookings.fare_conditions_economy PARTITION OF ticket_flights_part FOR VALUES IN ('Economy);

   CREATE TABLE bookings.fare_conditions_comfort PARTITION OF ticket_flights_part FOR VALUES IN ('Comfort');

   CREATE TABLE bookings.fare_conditions_business PARTITION OF ticket_flights_part FOR VALUES IN ('Business');

   

   Снова получим информацию по таблице

   \d+ ticket_flights_part

                                          Partitioned table "bookings.ticket_flights_part"
        Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
   -----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
    ticket_no       | character(13)         |           | not null |         | extended |             |              |
    flight_id       | integer               |           | not null |         | plain    |             |              |
    fare_conditions | character varying(10) |           | not null |         | extended |             |              |
    amount          | numeric(10,2)         |           | not null |         | main     |             |              |
   Partition key: LIST (fare_conditions)
   Partitions: fare_conditions_business FOR VALUES IN ('Business'),
               fare_conditions_comfort FOR VALUES IN ('Comfort'),
               fare_conditions_economy FOR VALUES IN ('Economy')

   

   Получим спосок таблиц в базе

   \dt+

     

                                                         List of relations
     Schema  |           Name           |       Type        |  Owner   | Persistence | Access method |  Size   |    Description
   ----------+--------------------------+-------------------+----------+-------------+---------------+---------+-------------------
    bookings | aircrafts                | table             | postgres | permanent   | heap          | 16 kB   | Самолеты
    bookings | airports                 | table             | postgres | permanent   | heap          | 48 kB   | Аэропорты
    bookings | boarding_passes          | table             | postgres | permanent   | heap          | 455 MB  | Посадочные талоны
    bookings | bookings                 | table             | postgres | permanent   | heap          | 105 MB  | Бронирования
    bookings | fare_conditions_business | table             | postgres | permanent   | heap          | 0 bytes |
    bookings | fare_conditions_comfort  | table             | postgres | permanent   | heap          | 0 bytes |
    bookings | fare_conditions_economy  | table             | postgres | permanent   | heap          | 0 bytes |
    bookings | flights                  | table             | postgres | permanent   | heap          | 21 MB   | Рейсы
    bookings | seats                    | table             | postgres | permanent   | heap          | 96 kB   | Места
    bookings | ticket_flights           | table             | postgres | permanent   | heap          | 547 MB  | Перелеты
    bookings | ticket_flights_part      | partitioned table | postgres | permanent   |               | 0 bytes |
    bookings | tickets                  | table             | postgres | permanent   | heap          | 386 MB  | Билеты
   (12 rows)

   

   Пересём в таблицу ticket_flights_part данные из таблицы ticket_flights

   INSERT INTO ticket_flights_part SELECT * FROM ticket_flights;

   INSERT 0 8391852

   Вставлено ровно 8391852 строк, ровно как показывал запрос по таблице ticket_flights в пункте 6.

   

   Проверим список таблиц

   \dt+

                                                          List of relations
     Schema  |           Name           |       Type        |  Owner   | Persistence | Access method |  Size   |    Description
   ----------+--------------------------+-------------------+----------+-------------+---------------+---------+-------------------
    bookings | aircrafts                | table             | postgres | permanent   | heap          | 16 kB   | Самолеты
    bookings | airports                 | table             | postgres | permanent   | heap          | 48 kB   | Аэропорты
    bookings | boarding_passes          | table             | postgres | permanent   | heap          | 455 MB  | Посадочные талоны
    bookings | bookings                 | table             | postgres | permanent   | heap          | 105 MB  | Бронирования
    bookings | fare_conditions_business | table             | postgres | permanent   | heap          | 56 MB   |
    bookings | fare_conditions_comfort  | table             | postgres | permanent   | heap          | 9368 kB |
    bookings | fare_conditions_economy  | table             | postgres | permanent   | heap          | 481 MB  |
    bookings | flights                  | table             | postgres | permanent   | heap          | 21 MB   | Рейсы
    bookings | seats                    | table             | postgres | permanent   | heap          | 96 kB   | Места
    bookings | ticket_flights           | table             | postgres | permanent   | heap          | 547 MB  | Перелеты
    bookings | ticket_flights_part      | partitioned table | postgres | permanent   |               | 0 bytes |
    bookings | tickets                  | table             | postgres | permanent   | heap          | 386 MB  | Билеты
   (12 rows)

   Видно, что таблицы fare_conditions_business, fare_conditions_comfort и fare_conditions_comfort заполнились данными и суммарный объем трёх таблиц равен объёму таблицы ticket_flights 

   

   Получим описание таблицы ticket_flights_part

   \d ticket_flights_part

                                     Partitioned table "bookings.ticket_flights_part"
        Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
   -----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
    ticket_no       | character(13)         |           | not null |         | extended |             |              |
    flight_id       | integer               |           | not null |         | plain    |             |              |
    fare_conditions | character varying(10) |           | not null |         | extended |             |              |
    amount          | numeric(10,2)         |           | not null |         | main     |             |              |
   Partition key: LIST (fare_conditions)
   Partitions: fare_conditions_business FOR VALUES IN ('Business'),
               fare_conditions_comfort FOR VALUES IN ('Comfort'),
               fare_conditions_economy FOR VALUES IN ('Economy')

   

   В таблице всё ещё нет индекса

   Добавим его.

   

10. CREATE INDEX idx_ticket_flights_part ON ticket_flights_part(ticket_no, flight_id);

    Проверим как создался индекс

    \di+ idx_ticket_flights_part

                                                                   List of relations
      Schema  |          Name           |       Type        |  Owner   |        Table        | Persistence | Access method |  Size   | Description
    ----------+-------------------------+-------------------+----------+---------------------+-------------+---------------+---------+-------------
     bookings | idx_ticket_flights_part | partitioned index | postgres | ticket_flights_part | permanent   | btree         | 0 bytes |

    

    \d+ ticket_flights_part

                                            Partitioned table "bookings.ticket_flights_part"
         Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
    -----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
     ticket_no       | character(13)         |           | not null |         | extended |             |              |
     flight_id       | integer               |           | not null |         | plain    |             |              |
     fare_conditions | character varying(10) |           | not null |         | extended |             |              |
     amount          | numeric(10,2)         |           | not null |         | main     |             |              |
    Partition key: LIST (fare_conditions)
    Indexes:
        "idx_ticket_flights_part" btree (ticket_no, flight_id)
    Partitions: fare_conditions_business FOR VALUES IN ('Business'),
                fare_conditions_comfort FOR VALUES IN ('Comfort'),
                fare_conditions_economy FOR VALUES IN ('Economy')

    

    И cоздадим ещё один индекс btree по колонкам flight_id и fare_conditions ;

    CREATE INDEX idx_ticket_flights_part_fare_conditions_flight_id ON ticket_flights_part USING btree (fare_conditions, flight_id);

11. Проверим, как влияет партиционирование на скорость выполнения запросов

    EXPLAIN SELECT count(*) FROM ticket_flights_part WHERE fare_conditions = 'Economy' AND flight_id > 200;

    

                                                                                                 QUERY PLAN
    ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     Finalize Aggregate  (cost=136542.01..136542.02 rows=1 width=8)
       ->  Gather  (cost=136541.90..136542.01 rows=1 width=8)
             Workers Planned: 1
             ->  Partial Aggregate  (cost=135541.90..135541.91 rows=1 width=8)
                   ->  Parallel Index Only Scan using fare_conditions_economy_fare_conditions_flight_id_idx on fare_conditions_economy ticket_flights_part  (cost=0.43..124684.29 rows=4343043 width=0)
                         Index Cond: ((fare_conditions = 'Economy'::text) AND (flight_id > 200))
     JIT:
       Functions: 6
       Options: Inlining false, Optimization false, Expressions true, Deforming true
    (9 rows)
    
    Все данные получены из индекса (cost=0.43..124684.29 rows=4343043 width=0)
    
    
    
    12. Аналогичый тест на несекционированной таблице ticket_flights и без индекса
    
        EXPLAIN SELECT count(*) FROM ticket_flights WHERE fare_conditions = 'Economy' AND flight_id > 200;
    
                                                     QUERY PLAN
        ----------------------------------------------------------------------------------------------------
         Finalize Aggregate  (cost=155835.19..155835.20 rows=1 width=8)
           ->  Gather  (cost=155835.08..155835.19 rows=1 width=8)
                 Workers Planned: 1
                 ->  Partial Aggregate  (cost=154835.08..154835.09 rows=1 width=8)
                       ->  Parallel Seq Scan on ticket_flights  (cost=0.00..143978.75 rows=4342531 width=0)
                             Filter: ((flight_id > 200) AND ((fare_conditions)::text = 'Economy'::text))
         JIT:
           Functions: 6
           Options: Inlining false, Optimization false, Expressions true, Deforming true
        (9 rows)
    
        
    
    13. Проведём повторный тест удалив индекс из таблицы ticket_flights_part 
    
        DROP INDEX idx_ticket_flights_part_fare_conditions_flight_id
    
        
    
        EXPLAIN SELECT count(*) FROM ticket_flights_part WHERE fare_conditions = 'Economy' AND flight_id > 200;
    
                                                                   QUERY PLAN
        ---------------------------------------------------------------------------------------------------------------------------------
         Finalize Aggregate  (cost=138685.29..138685.30 rows=1 width=8)
           ->  Gather  (cost=138685.18..138685.29 rows=1 width=8)
                 Workers Planned: 1
                 ->  Partial Aggregate  (cost=137685.18..137685.19 rows=1 width=8)
                       ->  Parallel Seq Scan on fare_conditions_economy ticket_flights_part  (cost=0.00..126827.57 rows=4343043 width=0)
                             Filter: ((flight_id > 200) AND ((fare_conditions)::text = 'Economy'::text))
         JIT:
           Functions: 7
           Options: Inlining false, Optimization false, Expressions true, Deforming true
        (9 rows)
    
        
    
        EXPLAIN SELECT count(*) FROM ticket_flights WHERE fare_conditions = 'Economy' AND flight_id > 200;
    
                                                     QUERY PLAN
        ----------------------------------------------------------------------------------------------------
         Finalize Aggregate  (cost=155835.19..155835.20 rows=1 width=8)
           ->  Gather  (cost=155835.08..155835.19 rows=1 width=8)
                 Workers Planned: 1
                 ->  Partial Aggregate  (cost=154835.08..154835.09 rows=1 width=8)
                       ->  Parallel Seq Scan on ticket_flights  (cost=0.00..143978.75 rows=4342531 width=0)
                             Filter: ((flight_id > 200) AND ((fare_conditions)::text = 'Economy'::text))
         JIT:
           Functions: 6
           Options: Inlining false, Optimization false, Expressions true, Deforming true
        (9 rows)
    
    Как видно, процедура при выполнении запроса на секционированной таблице менее дорогая, чем на Несекционированной. Даже без индекса.
    
    