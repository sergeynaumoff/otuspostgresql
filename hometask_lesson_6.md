1. Создан инстанс postgresql

2. Установлен PostgreSQL 14 и создана база otus и таблица test с 1 строкой

   psql

   \l+

      Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
   -----------+----------+----------+---------+---------+-----------------------
    otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
    postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
    template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres
    template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
              |          |          |         |         | postgres=CTc/postgres

   \c otus

   SELECT * FROM test;

   

    c1
   ----
    1
   (1 row)

   

3. pg_lsclusters показывает что

   14  main    5432 online postgres  /var/lib/postgresql/14/main var/log/postgresql/postgresql-14-main.log

4. Создаём в GCP новый диск далее в виртуальной машине

   lsbls выдаёт наличие нового виртуального диска sdb

   sudo mkfs.ext4 /dev/sdb

   sudo mkdir /mnt/data && sudo chmod 700 -R /mnt/data && sudo chown postgres:postgres /mnt/data

   sudo mount /dev/sdb /mnt/data

   sudo blkid

   ​	получаем /dev/sdb: UUID="850ff658-f177-48e8-b20a-5d01575ab27b" TYPE="ext4"

   sudo nano /etc/fstab

   ​	задаём UUID=850ff658-f177-48e8-b20a-5d01575ab27b /mnt/data ext4

   sudo touch /mnt/data/testfile

   sudo reboot и переподключаемся к машине

   проверяем что диск подмонтировался и файл на месте sudo ls -l /mnt/data

   ​	файл testfile на месте, значит монтирование диска после перезагрузки машины происходит корректно

5. Останавливаем кластер PostgreSQL 14

   sudo pg_ctlcluster 14 main stop

6. Перемещаем каталок гластера на другой диск

   sudo mv /var/lib/postgresql/14/main /mnt/data

7. Стартуем кластер

   sudo pg_ctlcluster 14 main start и получаем ошибку

   Error: /var/lib/postgresql/14/main is not accessible or does not exist

8. Меняем в конфигурационном файле /etc/postgres/14/main/postgresql.conf дииректорию хранения данных кластера

   data_directory = '/var/lib/postgresql/14/main

9. Стартуем кластер sudo pg_ctlcluster 14 main start

10. Проверяем работу кластера pg_lsclusters

    Ver Cluster Port Status Owner     Data directory Log file
    14  main    5432 online <unknown> /mnt/data/main /var/log/postgresql/postgresql-14-main.log

11. Подключимся к кластеру через psql и проверим что данные на месте

    psql 

    \l

    otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
     postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
     template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres
     template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
               |          |          |         |         | postgres=CTc/postgres

    \c otus

    \dt

            List of relations
     Schema | Name | Type  |  Owner
    --------+------+-------+----------
     public | test | table | postgres

    SELECT * FROM test;

     c1
    ----
     1
    (1 row)

    









