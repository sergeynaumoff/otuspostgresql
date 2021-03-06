Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

1. Включим вывод сообщений в журнал сервера

   postgresql.conf >> log_lock_waits = on

2. Настроить в postgresql.conf максимальное время ожидания блокировки 

   postgresq.conf >> deadlock_timeout = 200ms

3. Перезагрузим конфигурацию сервера

   sudo pg_ctlcluster 14 main reload

4. Подключимся к серверу и проверим как применились настройки конфигурации

   SHOW log_lock_waits;

    log_lock_waits
   ----------------
    on
   (1 row)

   

   SHOW deadlock_timeout;

    deadlock_timeout
   ------------------
    200ms
   (1 row)

5. Создадим новую базу данных для теста и подключимся к ней

   CREATE DATABASE locks;

   \с locks;

6. Запустим параллельную ssh сессию и начнём мониторить файл журнала командой tail

   sudo tail -f -n 15 /var/log/postgresql/postgresql-14-main.log

7. Создадим в базе locks табличку locks и наполним её данными

   CREATE TABLE locks (no int PRIMARY KEY, first_name text, second_name text);

   INSERT INTO locks (no, first_name, second_name) VALUES (1, 'sergey', 'naumov'), (2, 'petr', 'petrov');

8. Запустим третью сессию ssh и подключимся к Postgres

   Итого в работе 3 сессии:

    	1. Первое подключение к PostgreSQL
    	2. Второе подключение к PostgreSQL
    	3. Монитор логов

9. Начнём транзакцию в первой сессии и изменим строку в таблице

   BEGIN;

   UPDATE locks SET first_name = 'ivan', second_name = 'ivanov' WHERE no = 1;

10. Начнём транзакцию во второй сессии и изменим строку в таблице

    BEGIN;

    UPDATE locks SET first_name = 'makar', second_name = 'makarov WHERE no = 1;

11.События в логах

2022-04-07 21:11:42.886 UTC [6451] postgres@locks WARNING:  there is already a transaction in progress
2022-04-07 21:12:32.647 UTC [6451] postgres@locks LOG:  process 6451 still waiting for ShareLock on transaction 756 after 200.138 ms
2022-04-07 21:12:32.647 UTC [6451] postgres@locks DETAIL:  Process holding the lock: 6449. Wait queue: 6451.
2022-04-07 21:12:32.647 UTC [6451] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "locks"
2022-04-07 21:12:32.647 UTC [6451] postgres@locks STATEMENT:  UPDATE locks SET first_name = 'makar', second_name = 'makarov' WHERE no = 1;

------

Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

1. Создадим новую базу данных для теста и подключимся к ней

   CREATE DATABASE locks;

   \с locks;

2. Создадим в базе locks табличку locks и наполним её данными

   CREATE TABLE locks (no int PRIMARY KEY, first_name text, second_name text);

   INSERT INTO locks (no, first_name, second_name) VALUES (1, 'sergey', 'naumov'), (2, 'petr', 'petrov');

3. Получим oid базы

   \x

   SELECT * FROM pg_database WHERE datname = 'locks';

   -[ RECORD 1 ]-+--------
   oid           | 16469
   datname       | locks
   datdba        | 10
   encoding      | 6
   datcollate    | C.UTF-8
   datctype      | C.UTF-8
   datistemplate | f
   datallowconn  | t
   datconnlimit  | -1
   datlastsysoid | 13704
   datfrozenxid  | 726
   datminmxid    | 1
   dattablespace | 1663
   datacl        |

4. Поучим OID таблицы

   SELECT pg_relation_filepath('locks');

   -[ RECORD 1 ]--------+-----------------
   pg_relation_filepath | base/16469/16470

5. Начнём транзакцию и изменим строку в базе

   BEGIN;

6. Получим ID процесса

   SELECT pg_backend_pid();

   -[ RECORD 1 ]--+-----
   pg_backend_pid | 9166

7. Посмотрим блокировки по PID

   SELECT * FROM pg_locks WHERE pid = 9166;

   -[ RECORD 1 ]------+----------------
   locktype           | relation
   database           | 16469
   relation           | 12290
   page               |
   tuple              |
   virtualxid         |
   transactionid      |
   classid            |
   objid              |
   objsubid           |
   virtualtransaction | 5/52
   pid                | 9166
   mode               | AccessShareLock
   granted            | t
   fastpath           | t
   waitstart          |
   -[ RECORD 2 ]------+----------------
   locktype           | virtualxid
   database           |
   relation           |
   page               |
   tuple              |
   virtualxid         | 5/52
   transactionid      |
   classid            |
   objid              |
   objsubid           |
   virtualtransaction | 5/52
   pid                | 9166
   mode               | ExclusiveLock
   granted            | t
   fastpath           | t
   waitstart          |

   Данный процесс получил блокировку ExclusiveLock на виртуальный ID транзакции. Блокировка предоставлена потому как granted  = t

   Блокировка AccessShareLock на relation 12290 - это блокировка в pg_locks

8. Выполним обновление строки в таблице

   UPDATE locks SET first_name = 'ivan', second_name = 'ivanov' WHERE no = 1;

9. Получим ID Текущей транзакции

   \x

   SELECT txid_current(); 

   -[ RECORD 1 ]+----
   txid_current | 795

10. Получим список блокировок но только с другим запросом к pg_locks с отбором по pin 9166

    SELECT locktype, relation::REGCLASS, virtualxid, transactionid, mode, granted FROM pg_locks WHERE pid = 9166;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_Klg8SWuloh_1.png?raw=true)

​	На скриншоте видно, что наш процесс держит 5 блокировок. Нас интересуют только 4, так как одни из блокировок идёт на pg_locks

​	Соответственно видно что получена блокировка на номер транзакции и виртуальный номер транзакции а так же таблицу и первичный ключ в режиме Row Exclusive. блокировку в этом режиме получает любая команда, которая ***изменяет данные\*** в таблице.

11. Запустим второй сеанс, подключимся к базе locks, начнём транзакцию и получим pid и id транзакции

    \c locks

    \x

    SELECT pg_backend_pid();
    -[ RECORD 1 ]--+------
    pg_backend_pid | 13040

    BEGIN;

    SELECT txid_current();

    -[ RECORD 1 ]+----
    txid_current | 801

12. Получим список блокировок с отбором по pid 13040 

    SELECT locktype, relation::REGCLASS, virtualxid, transactionid, mode, granted FROM pg_locks WHERE pid = 13040;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_2uuEqLhWlr_2.png?raw=true)

    Получена блокировка на ID транзакции и виртуальный ID транзакции

13. Начнём менять данные в таблице во втором сеансе

    UPDATE locks SET first_name = 'petr', second_name = 'petrov' WHERE no = 1;

    Транзакция зависла

14. Получим список блокировок с отбором по pid 13040 

    SELECT locktype, relation::REGCLASS, virtualxid, transactionid, mode, granted FROM pg_locks WHERE pid = 13040;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_anjVfhN2vM_3.png?raw=true)

Здесь:

	- Share блокировка (не предоставлена) на транзакцию с ID 795 - это наша первая транзакция с изменением которая пока   не завершена
	- Эксклюзивная блокировка на номер транзакции 801 - текущая транзакция 
	- Блокировка на виртуальный ID транзакции (именно текущей транзакции 801)
	- Блокировка на таблицу и первичный ключ
	- Блокировка на кортеж

15. Запустим третий сеанс, начнём транзакцию и получим pid и id транзакции

    \с locks

    \x

    SELECT pg_backend_pid();

    -[ RECORD 1 ]--+------
    pg_backend_pid | 15396

    BEGIN;

    SELECT txid_current();

    -[ RECORD 1 ]+----
    txid_current | 802

16. Получим список блокировок с отбором по pid 15396

    SELECT locktype, relation::REGCLASS, virtualxid, transactionid, mode, granted FROM pg_locks WHERE pid = 15396;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_wiO6GFp9Q9_4.png?raw=true)

​	   Получена блокировка на ID транзакции и виртуальный ID транзакции

17. Начнём менять данные в таблице в третьем сеансе

    UPDATE locks SET first_name = 'andey', second_name = 'andreev' WHERE no = 1;

    Транзакция зависла

18. Получим список блокировок с отбором по pid 15396

    SELECT locktype, relation::REGCLASS, virtualxid, transactionid, mode, granted FROM pg_locks WHERE pid = 15396;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_QZh5X8dFoH_5.png?raw=true)

    Здесь:

     - Эксклюзивная блокировка на номер транзакции 802 - текущая транзакция 
     - Блокировка на виртуальный ID транзакции (именно текущей транзакции 802)
     - Блокировка на таблицу и первичный ключ
     - Блокировка на кортеж (не предоставлена)

19. Выполним COMMIT транзакции в первом сеансе и посмотрим как отработают транзакции во втором и третьем сеансе

    COMMIT;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_2eGZkVPiuF_6.png?raw=true)

​	Транзакция в первом сеансе завершилась, во втором сеанса прошёл UPDATE. В третьем сеансе ничего не поменялось.

20. Выполним SELECT из таблицы

    locks=# SELECT * FROM locks;
     no | first_name | second_name
    ----+------------+-------------
      2 | petr       | petrov
      1 | ivan       | ivanov

    

    Так как текущий уровень изоляции READ_COMMITED

    -[ RECORD 1 ]---------+---------------
    transaction_isolation | read committed

    Будут отображаться только завершённые транзакции

20. Выполним COMMIT транзакции во втором сеансе

    COMMIT;

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_1XC66gEqyJ_7.png?raw=true)

​	Транзакция во втором сеансе завершилась, в третьем сеансе прошёл UPDATE

21. Выполним SELECT из таблицы

    SELECT * FROM locks;

     no | first_name | second_name
    ----+------------+-------------
      2 | petr       | petrov
      1 | petr       | petrov

22. Выполним COMMIT транзакции в третьем сеансе

    COMMIT

    ![](https://github.com/sergeynaumoff/otuspostgresql/blob/main/WindowsTerminal_nFtxjdG1yQ_8.png?raw=true)

23. Выполним SELECT из таблицы

    SELECT * FROM locks;

     no | first_name | second_name
    ----+------------+-------------
      2 | petr       | petrov
      1 | andey      | andreev

Вывод - транзакции выстроились в очередь друг за другом.

------

Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

1. Создадим в базе locks табличку accounts и наполним её данными

   \с locks

   CREATE TABLE accounts( acc_no integer PRIMARY KEY, amount numeric);

   INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);

2. Проверим конфигурацию кластера на пример таймаута взаимоблокировок и логирования блокировок

   SHOW deadlock_timeout;

   -[ RECORD 1 ]----+------
   deadlock_timeout | 200ms

   Увеличим до 1000

   

   SHOW log_lock_waits;

   -[ RECORD 1 ]--+---
   log_lock_waits | on

3. Начнём мониторить лог

   sudo tail -n 10 -f /var/log/postgresql/postgresql-14-main.log

4. Начнём транзакцию в первой сессии на счёте №1 уменьшим сумму на 100 для переноса её на счёт 2

   BEGIN;

   \x

   SELECT pg_backend_pid();

   -[ RECORD 1 ]--+-----
   pg_backend_pid | 1491

   UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;

5. Стартуем вторую сессию и начнём менять в ней данные

   на счёте 2 уменьшим сумму на 10 для переноса её на счёт 1

   BEGIN;

   \x

   SELECT pg_backend_pid();

   -[ RECORD 1 ]--+-----
   pg_backend_pid | 1540

   UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;

6. В первой сессии выполним транзакцию 

   Полним счёт 2 на 100, которые ранее были сняты со счёта 1

   UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

7. Во второй сессии выполним транзакцию

   Пополним счёт 1 на 10, которые ранее были сняты со счёта 2

   UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;

Итого pid первой сессии 1491, pid второй сессии 1540

После выполнения первых 6 операций в логах получим

2022-04-11 10:02:19.612 UTC [1491] postgres@locks LOG:  process 1491 still waiting for ShareLock on transaction 824 after 1000.144 ms
2022-04-11 10:02:19.612 UTC [1491] postgres@locks DETAIL:  Process holding the lock: 1540. Wait queue: 1491.
2022-04-11 10:02:19.612 UTC [1491] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2022-04-11 10:02:19.612 UTC [1491] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

После выполнения действия 6 в логах появляются записи. До момента выполнения операции 6 (UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;) блокировки не возникало потому что не возникало операций изменяющих одну и ту же строку в одной и той же таблице.

То есть на данный момент мы имеем 2 операции которые меняют с троку с acc_no = 2 в таблице accounts:

- Процесс с ID 1540 выполнил  UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2 и стартовал изменения в таблице по acc_no = 2 первее 
- Процесс с ID 1491 начал изменения в этой же строке операцией UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2; и встал в очередь на ожидание.

После выполнения операции 7 в логах получим

2022-04-11 10:11:40.036 UTC [1540] postgres@locks LOG:  process 1540 detected deadlock while waiting for ShareLock on transaction 823 after 1000.161 ms
2022-04-11 10:11:40.036 UTC [1540] postgres@locks DETAIL:  Process holding the lock: 1491. Wait queue: .
2022-04-11 10:11:40.036 UTC [1540] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2022-04-11 10:11:40.036 UTC [1540] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
2022-04-11 10:11:40.037 UTC [1540] postgres@locks ERROR:  deadlock detected
2022-04-11 10:11:40.037 UTC [1540] postgres@locks DETAIL:  Process 1540 waits for ShareLock on transaction 823; blocked by process 1491.
        Process 1491 waits for ShareLock on transaction 824; blocked by process 1540.
        Process 1540: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
        Process 1491: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
2022-04-11 10:11:40.037 UTC [1540] postgres@locks HINT:  See server log for query details.
2022-04-11 10:11:40.037 UTC [1540] postgres@locks CONTEXT:  while updating tuple (0,1) in relation "accounts"
2022-04-11 10:11:40.037 UTC [1540] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
2022-04-11 10:11:40.037 UTC [1491] postgres@locks LOG:  process 1491 acquired ShareLock on transaction 824 after 561425.103 ms
2022-04-11 10:11:40.037 UTC [1491] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
2022-04-11 10:11:40.037 UTC [1491] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

То есть получена взаимоблокировка в которой процессы 1540 и 1491 ждут завершения друг друга

------

Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Ответ: *взаимоблокировка 2-х транзакций, выполняющих UPDATE одной и той же таблицы (без where) возможна, если одна команда будет обновлять строки таблицы в прямом порядке, а другая - в обратном* 









