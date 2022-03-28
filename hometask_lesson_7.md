1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL) 

​	Создана VM с именем postgres

2 зайдите в созданный кластер под пользователем postgres 

​	psql postgres postgres

​	OTUS

3 создайте новую базу данных testdb 

​	CREATE DATABASE testdb;

4 зайдите в созданную базу данных под пользователем postgres 

​	\c testdb

	You are now connected to database "testdb" as user "postgres".

5 создайте новую схему testnm 

​	CREATE SCHEMA testnm;

6 создайте новую таблицу t1 с одной колонкой c1 типа integer 

​	CREATE TABLE t1 (c1 int);

7 вставьте строку со значением c1=1 

​	INSERT INTO t1 VALUES (1);

8 создайте новую роль readonly 

​	CREATE ROLE readonly;

9 дайте новой роли право на подключение к базе данных testdb 

​	GRANT CONNECT ON DATABASE testdb TO readonly;

10 дайте новой роли право на использование схемы testnm 

​	GRANT USAGE ON SCHEMA testnm TO readonly;

11 дайте новой роли право на select для всех таблиц схемы testnm 

​	GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;

12 создайте пользователя testread с паролем test123 

​	CREATE USER testread WITH PASSWORD 'test123';

13 дайте роль readonly пользователю testread 

​	GRANT readonly TO testread;

14 зайдите под пользователем testread в базу данных testdb 

​	редактируем ph_hba

	# "local" is for Unix domain socket connections only

​	local   all             all                                     md5

​	sudo pg_ctlcluster 14 main realod

​	psql postgres postgres

​	\c testbd

​	\c testdb testread -> test123

​	You are now connected to database "testdb" as user "testread".

15 сделайте select * from t1; 

16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже) 

​	Не получилось

17 напишите что именно произошло в тексте домашнего задания 

​		ERROR:  permission denied for table t1

18 у вас есть идеи почему? ведь права то дали? 

​	Таблица t1 создана в схеме PUBLIC

​	Для пользователя testread не давали прав на схему PUBLIC.

                                     List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+------+-------+----------+-------------+---------------+------------+-------------
 public | t1   | table | postgres | permanent   | heap          | 8192 bytes |
(1 row)

19 посмотрите на список таблиц 

​	\dt+

                                     List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+------+-------+----------+-------------+---------------+------------+-------------
 public | t1   | table | postgres | permanent   | heap          | 8192 bytes |
(1 row)

20 подсказка в шпаргалке под пунктом 20

21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально) 

​	SHOW search_path;

	   search_path
-----------------
 "$user", public

Данная настройка говорит о том, что по-умолчанию при создании таблицы ищется схема с именем пользователя под которым выполняется операция а потом схема PUBLIC

Поэтому таблица создалась в схеме PUBLIC

22 вернитесь в базу данных testdb под пользователем postgres 

​	\с testdb postgres 

​	You are now connected to database "testdb" as user "postgres".

23 удалите таблицу t1 

​	DROP TABLE t1;

24 создайте ее заново но уже с явным указанием имени схемы testnm 

​	CREATE TABLE testnm.t1 (c1 int);

25 вставьте строку со значением c1=1 

​	INSERT INTO testnm.t1 (c1) VALUES (1);

26 зайдите под пользователем testread в базу данных testdb 

​	\c testdb testread

27 сделайте select * from testnm.t1; 

​	ERROR:  permission denied for table t1

28 получилось? 

​	Нет

29 есть идеи почему? если нет - смотрите шпаргалку 

​	Таблица t1 пересоздавалась, права на SELECT в схеме testnm таблице t1 надо переназначить

30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку 

​	ALTER DEFAULT PRIVILEGES in SCHEMA testnm GRANT SELECT on TABLES to readonly;

31 сделайте select * from testnm.t1; 

​	permission denied for table t1

32 получилось? 

​	Нет

33 есть идеи почему? если нет - смотрите шпаргалку 

​	Действует только для новых таблиц. Надо либо пересоздать таблицу либо назначить на таблицу права.

​	Назначим на таблицу права так как в рабочей среде пересоздавать таблицу с данными как-то не очень

​	\c testdb postgres

​	GRANT SELECT ON ALL TABLES in SCHEMA testnm TO readonly;

 	\c testdb testread

31 сделайте select * from testnm.t1; 

32 получилось? 

​	Получилось

	 c1
----
  1
(1 row)

33 ура! 

Ура

34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2); 

	CREATE TABLE

INSERT 0 1

35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly? 

​	Потому что по-молчанию таблица создалась в схеме PUBLIC к которой есть lcjneg у всех пользователей

36 есть идеи как убрать эти права? если нет - смотрите шпаргалку я

​	\с testdb postgres

​	REVOKE CREATE on SCHEMA public FROM public;

​	REVOKE all on DATABASE testdb FROM public;

​	\c testdb testread

37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды 

38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2); 

39 расскажите что получилось и почему 

​	ERROR:  permission denied for schema public

​	Отобрали права на создание в схеме PUBLIC