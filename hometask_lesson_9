1. Создаём базу данных testdb

2. \c testdb

3. CREATE EXTENSION pg_buffercache;

4. **SELECT** count(*) **FROM** pg_buffercache **WHERE** isdirty;

   count
   -------
        50

   (1 row)

5. Проверим статистику выполненных CHECKPOINT

   SELECT * FROM pg_stat_bgwriter \gx

   checkpoints_timed     | 2
   checkpoints_req       | 2
   checkpoint_write_time | 4931
   checkpoint_sync_time  | 11
   buffers_checkpoint    | 53
   buffers_clean         | 0
   maxwritten_clean      | 0
   buffers_backend       | 1
   buffers_backend_fsync | 0
   buffers_alloc         | 292
   stats_reset           | 2022-04-03 19:10:46.518735+00

6. Сборисим грязные буфферы на диск командой CHECKPOINT;

   **SELECT** count(*) **FROM** pg_buffercache **WHERE** isdirty;

   count

        0

   (1 row)

7. Проверим статистику выполненных CHECKPOINT

   SELECT * FROM pg_stat_bgwriter \gx

   checkpoints_timed     | 3
   checkpoints_req       | 3
   checkpoint_write_time | 4932
   checkpoint_sync_time  | 13
   buffers_checkpoint    | 54
   buffers_clean         | 0
   maxwritten_clean      | 0
   buffers_backend       | 1
   buffers_backend_fsync | 0
   buffers_alloc         | 299
   stats_reset           | 2022-04-03 19:10:46.518735+00

8. Посмотрим на список файлов WAL

              name           |   size   |      modification
   --------------------------+----------+------------------------
    000000010000000000000001 | 16777216 | 2022-04-03 19:12:57+00
   (1 row)

9. Смотрим текущий LSN

   SELECT pg_current_wal_insert_lsn();

    pg_current_wal_insert_lsn
   ---------------------------
    0/171CDB8
   (1 row)

10. Смотрим в каком WAL "лежит" LSN

    SELECT pg_walfile_name( 0/171CDB8');

         pg_walfile_name
    --------------------------
     000000010000000000000001
    (1 row)

11. Меняем выполнение контрольной точки на раз в 30 секунд в файле postgresql.conf

​	checkpoint_timeout = 30s

​	sudo pg_ctlcluster 14 main reloaв

12. Проверим статистику выполненных CHECKPOINT

    SELECT * FROM pg_stat_bgwriter \gx

    -[ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 9
    checkpoints_req       | 3
    checkpoint_write_time | 4932
    checkpoint_sync_time  | 13
    buffers_checkpoint    | 54
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 1
    buffers_backend_fsync | 0
    buffers_alloc         | 304
    stats_reset           | 2022-04-03 19:10:46.518735+00

10. Запускаем pgbench

pgbench -i -U postgres -d testdb

pgbench -c 1 -P 10 -T 600 -U postgres testdb;

pgbench (14.2 (Debian 14.2-1.pgdg100+1))
starting vacuum...end.
progress: 10.0 s, 575.2 tps, lat 1.737 ms stddev 0.453
progress: 20.0 s, 581.3 tps, lat 1.719 ms stddev 0.235
progress: 30.0 s, 602.3 tps, lat 1.659 ms stddev 0.254
progress: 40.0 s, 567.5 tps, lat 1.761 ms stddev 0.269
progress: 50.0 s, 599.2 tps, lat 1.668 ms stddev 0.281
progress: 60.0 s, 573.7 tps, lat 1.742 ms stddev 0.332
progress: 70.0 s, 672.1 tps, lat 1.487 ms stddev 0.330
progress: 80.0 s, 687.0 tps, lat 1.455 ms stddev 0.268
progress: 90.0 s, 579.5 tps, lat 1.725 ms stddev 0.456
progress: 100.0 s, 575.5 tps, lat 1.737 ms stddev 0.305
progress: 110.0 s, 653.1 tps, lat 1.531 ms stddev 0.294
progress: 120.0 s, 630.3 tps, lat 1.586 ms stddev 0.262
progress: 130.0 s, 595.5 tps, lat 1.679 ms stddev 0.234
progress: 140.0 s, 604.5 tps, lat 1.653 ms stddev 0.251
progress: 150.0 s, 675.1 tps, lat 1.481 ms stddev 0.299
progress: 160.0 s, 590.4 tps, lat 1.693 ms stddev 0.236
progress: 170.0 s, 666.5 tps, lat 1.500 ms stddev 0.276
progress: 180.0 s, 656.4 tps, lat 1.523 ms stddev 0.297
progress: 190.0 s, 612.2 tps, lat 1.633 ms stddev 0.214
progress: 200.0 s, 714.9 tps, lat 1.398 ms stddev 0.267
progress: 210.0 s, 633.3 tps, lat 1.578 ms stddev 0.241
progress: 220.0 s, 512.0 tps, lat 1.952 ms stddev 0.414
progress: 230.0 s, 536.8 tps, lat 1.862 ms stddev 0.265
progress: 240.0 s, 588.4 tps, lat 1.699 ms stddev 0.324
progress: 250.0 s, 590.3 tps, lat 1.693 ms stddev 0.211
progress: 260.0 s, 588.2 tps, lat 1.699 ms stddev 0.198
progress: 270.0 s, 649.6 tps, lat 1.539 ms stddev 0.279
progress: 280.0 s, 626.3 tps, lat 1.596 ms stddev 0.276
progress: 290.0 s, 701.8 tps, lat 1.424 ms stddev 0.267
progress: 300.0 s, 613.3 tps, lat 1.630 ms stddev 0.270
progress: 310.0 s, 589.9 tps, lat 1.695 ms stddev 0.220
progress: 320.0 s, 654.0 tps, lat 1.528 ms stddev 0.275
progress: 330.0 s, 600.0 tps, lat 1.666 ms stddev 0.221
progress: 340.0 s, 554.0 tps, lat 1.804 ms stddev 0.290
progress: 350.0 s, 591.5 tps, lat 1.690 ms stddev 0.208
progress: 360.0 s, 625.3 tps, lat 1.598 ms stddev 0.271
progress: 370.0 s, 572.9 tps, lat 1.745 ms stddev 0.255
progress: 380.0 s, 542.7 tps, lat 1.842 ms stddev 0.234
progress: 390.0 s, 600.1 tps, lat 1.666 ms stddev 0.288
progress: 400.0 s, 587.9 tps, lat 1.700 ms stddev 0.306
progress: 410.0 s, 603.4 tps, lat 1.656 ms stddev 0.255
progress: 420.0 s, 650.7 tps, lat 1.536 ms stddev 0.300
progress: 430.0 s, 667.0 tps, lat 1.499 ms stddev 0.285
progress: 440.0 s, 676.0 tps, lat 1.479 ms stddev 0.293
progress: 450.0 s, 623.8 tps, lat 1.602 ms stddev 0.295
progress: 460.0 s, 568.3 tps, lat 1.759 ms stddev 0.320
progress: 470.0 s, 681.6 tps, lat 1.466 ms stddev 0.284
progress: 480.0 s, 605.8 tps, lat 1.650 ms stddev 0.259
progress: 490.0 s, 591.2 tps, lat 1.691 ms stddev 0.218
progress: 500.0 s, 604.9 tps, lat 1.652 ms stddev 0.276
progress: 510.0 s, 674.4 tps, lat 1.482 ms stddev 0.308
progress: 520.0 s, 564.8 tps, lat 1.770 ms stddev 0.470
progress: 530.0 s, 553.8 tps, lat 1.805 ms stddev 0.304
progress: 540.0 s, 700.9 tps, lat 1.426 ms stddev 0.275
progress: 550.0 s, 598.4 tps, lat 1.670 ms stddev 0.247
progress: 560.0 s, 583.8 tps, lat 1.712 ms stddev 4.727
progress: 570.0 s, 603.5 tps, lat 1.656 ms stddev 0.199
progress: 580.0 s, 572.9 tps, lat 1.745 ms stddev 0.250
progress: 590.0 s, 614.4 tps, lat 1.627 ms stddev 0.246
progress: 600.0 s, 582.0 tps, lat 1.718 ms stddev 0.263
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 366922
latency average = 1.634 ms
latency stddev = 0.670 ms
initial connection time = 3.303 ms
tps = 611.538116 (without initial connection time)

11. Подключаемся к базе и проверим текущий LSN

     pg_current_wal_insert_lsn
    ---------------------------
     0/1BE91410
    (1 row)

12. Проверим количество файлов WAL  в директории 

    SELECT * FROM pg_ls_waldir();

               name           |   size   |      modification
    --------------------------+----------+------------------------
     00000001000000000000001E | 16777216 | 2022-04-03 19:26:19+00
     00000001000000000000001C | 16777216 | 2022-04-03 19:25:25+00
     00000001000000000000001B | 16777216 | 2022-04-03 19:27:49+00
     00000001000000000000001D | 16777216 | 2022-04-03 19:25:51+00
    (4 rows)

13. Узнать объём журнальных записей за время проведённого теста

    SELECT '0/1BE91410'::pg_lsn - '0/171CDB8'::pg_lsn;

     ?column?
    -----------
     444024408
    (1 row)

14. Проверим количество выполненных CHECKPOINT

    -[ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 43
    checkpoints_req       | 3
    checkpoint_write_time | 571351
    checkpoint_sync_time  | 138
    buffers_checkpoint    | 41733
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 4079
    buffers_backend_fsync | 0
    buffers_alloc         | 4871
    stats_reset           | 2022-04-03 19:10:46.518735+00

15. О количестве выполненых контрольных точек свидетельствуют показатели

    ​	checkpoints_timed  

    ​	checkpoints_req   

    Всего на момент начала теста было 

    ​	checkpoints_timed     | 9
    ​	checkpoints_req       | 3

    После окончания теста 

    ​	checkpoints_timed     | 43
    ​	checkpoints_req       | 3

    При этом на момент окончания теста

    ​	buffers_checkpoint    | 41733
    ​	buffers_clean         | 0
    ​	buffers_backend       | 4079

    buffers_checkpoint — число записанных страниц процессом контрольной точки,

    buffers_backend — число записанных страниц обслуживающими процессами,

    buffers_clean — число записанных страниц процессом фоновой записи.

    В хорошо настроенной системе значение buffers_backend  существенно меньше, чем сумма buffers_checkpoint и buffers_clean

16. Сбросим статистику и повторим тест

    SELECT pg_stat_reset_shared('bgwriter');

    SELECT * FROM pg_stat_bgwriter \gx

    -[ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 0
    checkpoints_req       | 0
    checkpoint_write_time | 0
    checkpoint_sync_time  | 0
    buffers_checkpoint    | 0
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 0
    buffers_backend_fsync | 0
    buffers_alloc         | 0
    stats_reset           | 2022-04-03 19:48:08.021074+00

    pgbench -i -U postgres -d testdb

    pgbench -c 1 -P 10 -T 600 -U postgres testdb;

    

    pgbench (14.2 (Debian 14.2-1.pgdg100+1))
    starting vacuum...end.
    progress: 10.0 s, 620.6 tps, lat 1.610 ms stddev 0.275
    progress: 20.0 s, 676.1 tps, lat 1.478 ms stddev 0.280
    progress: 30.0 s, 615.0 tps, lat 1.625 ms stddev 0.239
    progress: 40.0 s, 650.0 tps, lat 1.538 ms stddev 0.249
    progress: 50.0 s, 577.6 tps, lat 1.730 ms stddev 0.291
    progress: 60.0 s, 566.2 tps, lat 1.766 ms stddev 0.233
    progress: 70.0 s, 580.9 tps, lat 1.721 ms stddev 0.270
    progress: 80.0 s, 633.4 tps, lat 1.578 ms stddev 0.415
    progress: 90.0 s, 528.3 tps, lat 1.892 ms stddev 0.321
    progress: 100.0 s, 628.7 tps, lat 1.590 ms stddev 0.315
    progress: 110.0 s, 607.7 tps, lat 1.645 ms stddev 0.283
    progress: 120.0 s, 748.0 tps, lat 1.336 ms stddev 0.242
    progress: 130.0 s, 622.2 tps, lat 1.606 ms stddev 0.265
    progress: 140.0 s, 647.5 tps, lat 1.544 ms stddev 0.254
    progress: 150.0 s, 612.1 tps, lat 1.633 ms stddev 0.291
    progress: 160.0 s, 717.2 tps, lat 1.394 ms stddev 0.258
    progress: 170.0 s, 633.8 tps, lat 1.577 ms stddev 0.271
    progress: 180.0 s, 758.9 tps, lat 1.317 ms stddev 0.253
    progress: 190.0 s, 640.5 tps, lat 1.561 ms stddev 0.264
    progress: 200.0 s, 653.7 tps, lat 1.529 ms stddev 0.251
    progress: 210.0 s, 724.9 tps, lat 1.379 ms stddev 0.272
    progress: 220.0 s, 655.2 tps, lat 1.526 ms stddev 0.254
    progress: 230.0 s, 610.0 tps, lat 1.639 ms stddev 0.210
    progress: 240.0 s, 636.6 tps, lat 1.570 ms stddev 0.244
    progress: 250.0 s, 597.4 tps, lat 1.673 ms stddev 0.225
    progress: 260.0 s, 623.5 tps, lat 1.603 ms stddev 0.296
    progress: 270.0 s, 621.3 tps, lat 1.609 ms stddev 0.263
    progress: 280.0 s, 607.6 tps, lat 1.645 ms stddev 0.180
    progress: 290.0 s, 654.9 tps, lat 1.526 ms stddev 0.250
    progress: 300.0 s, 654.9 tps, lat 1.526 ms stddev 0.279
    progress: 310.0 s, 624.6 tps, lat 1.600 ms stddev 0.328
    progress: 320.0 s, 672.1 tps, lat 1.487 ms stddev 0.264
    progress: 330.0 s, 599.0 tps, lat 1.669 ms stddev 0.219
    progress: 340.0 s, 654.3 tps, lat 1.527 ms stddev 0.269
    progress: 350.0 s, 695.0 tps, lat 1.438 ms stddev 0.262
    progress: 360.0 s, 624.1 tps, lat 1.602 ms stddev 0.253
    progress: 370.0 s, 604.2 tps, lat 1.654 ms stddev 0.270
    progress: 380.0 s, 597.1 tps, lat 1.674 ms stddev 0.301
    progress: 390.0 s, 569.2 tps, lat 1.756 ms stddev 0.717
    progress: 400.0 s, 621.9 tps, lat 1.607 ms stddev 0.282
    progress: 410.0 s, 619.5 tps, lat 1.613 ms stddev 0.239
    progress: 420.0 s, 610.3 tps, lat 1.638 ms stddev 4.754
    progress: 430.0 s, 617.3 tps, lat 1.619 ms stddev 0.255
    progress: 440.0 s, 618.6 tps, lat 1.616 ms stddev 0.297
    progress: 450.0 s, 619.5 tps, lat 1.613 ms stddev 0.248
    progress: 460.0 s, 670.0 tps, lat 1.492 ms stddev 0.251
    progress: 470.0 s, 602.4 tps, lat 1.659 ms stddev 0.273
    progress: 480.0 s, 611.6 tps, lat 1.634 ms stddev 0.323
    progress: 490.0 s, 630.1 tps, lat 1.586 ms stddev 0.330
    progress: 500.0 s, 603.9 tps, lat 1.655 ms stddev 0.255
    progress: 510.0 s, 602.5 tps, lat 1.659 ms stddev 0.264
    progress: 520.0 s, 674.8 tps, lat 1.481 ms stddev 0.289
    progress: 530.0 s, 595.8 tps, lat 1.678 ms stddev 0.234
    progress: 540.0 s, 610.0 tps, lat 1.639 ms stddev 0.255
    progress: 550.0 s, 607.7 tps, lat 1.645 ms stddev 0.304
    progress: 560.0 s, 582.3 tps, lat 1.716 ms stddev 0.276
    progress: 570.0 s, 675.4 tps, lat 1.480 ms stddev 0.305
    progress: 580.0 s, 596.7 tps, lat 1.675 ms stddev 0.182
    progress: 590.0 s, 630.6 tps, lat 1.585 ms stddev 0.245
    progress: 600.0 s, 675.7 tps, lat 1.479 ms stddev 0.270
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 378210
    latency average = 1.586 ms
    latency stddev = 0.674 ms
    initial connection time = 3.278 ms
    tps = 630.352983 (without initial connection time)

17. Снова получим данные статистики

    SELECT * FROM pg_stat_bgwriter \gx

    [ RECORD 1 ]---------+------------------------------
    checkpoints_timed     | 24
    checkpoints_req       | 0
    checkpoint_write_time | 538579
    checkpoint_sync_time  | 98
    buffers_checkpoint    | 37141
    buffers_clean         | 0
    maxwritten_clean      | 0
    buffers_backend       | 2419
    buffers_backend_fsync | 0
    buffers_alloc         | 2415
    stats_reset           | 2022-04-03 19:48:08.021074+00

    Тест был выполнен на очищенной статистике а значит данные в ней можно считать коррекными и отображающими статистику именно за период теста. Согласно настройкам checkpoint должен был выполняться каждый 30 секунд. При учёте проведения тестов в течение 10 минут всего должно было выполниться 20 checkpoint. В статистике выполнено 24 и ни одного по требованию (checkpoints_req) из чего можно сделать вывод что все контрольные точки были выполнены по расписанию.

18. Проведём нагрузочный тест в асинхронном режиме 

    Включим асинхронный режим в настройках кластера: synchronous_commit = off

    pg_ctlcluster 14 main reload

19. Запустим тест

    pgbench -c 1 -P 10 -T 600 -U postgres testdb;

    pgbench (14.2 (Debian 14.2-1.pgdg100+1))
    starting vacuum...end.
    progress: 10.0 s, 1300.5 tps, lat 0.768 ms stddev 0.092
    progress: 20.0 s, 1429.9 tps, lat 0.699 ms stddev 0.168
    progress: 30.0 s, 1175.4 tps, lat 0.850 ms stddev 0.168
    progress: 40.0 s, 1229.8 tps, lat 0.812 ms stddev 0.166
    progress: 50.0 s, 1202.3 tps, lat 0.831 ms stddev 0.202
    progress: 60.0 s, 1184.3 tps, lat 0.844 ms stddev 0.183
    progress: 70.0 s, 1256.6 tps, lat 0.795 ms stddev 0.114
    progress: 80.0 s, 1269.1 tps, lat 0.787 ms stddev 0.116
    progress: 90.0 s, 1258.5 tps, lat 0.794 ms stddev 0.124
    progress: 100.0 s, 1278.5 tps, lat 0.782 ms stddev 0.142
    progress: 110.0 s, 1291.6 tps, lat 0.774 ms stddev 0.106
    progress: 120.0 s, 1233.5 tps, lat 0.810 ms stddev 0.131
    progress: 130.0 s, 669.1 tps, lat 1.494 ms stddev 9.174
    progress: 140.0 s, 634.0 tps, lat 1.557 ms stddev 9.806
    progress: 150.0 s, 620.2 tps, lat 1.612 ms stddev 10.031
    progress: 160.0 s, 645.9 tps, lat 1.548 ms stddev 9.845
    progress: 170.0 s, 648.4 tps, lat 1.542 ms stddev 9.823
    progress: 180.0 s, 646.7 tps, lat 1.546 ms stddev 9.836
    progress: 190.0 s, 644.8 tps, lat 1.550 ms stddev 9.846
    progress: 200.0 s, 636.3 tps, lat 1.571 ms stddev 9.882
    progress: 210.0 s, 629.5 tps, lat 1.588 ms stddev 9.956
    progress: 220.0 s, 656.7 tps, lat 1.522 ms stddev 9.756
    progress: 230.0 s, 649.2 tps, lat 1.540 ms stddev 9.820
    progress: 240.0 s, 649.6 tps, lat 1.539 ms stddev 9.814
    progress: 250.0 s, 660.5 tps, lat 1.514 ms stddev 9.729
    progress: 260.0 s, 634.2 tps, lat 1.576 ms stddev 9.924
    progress: 270.0 s, 639.2 tps, lat 1.564 ms stddev 9.771
    progress: 280.0 s, 660.0 tps, lat 1.515 ms stddev 9.736
    progress: 290.0 s, 651.9 tps, lat 1.514 ms stddev 9.675
    progress: 300.0 s, 645.4 tps, lat 1.529 ms stddev 9.718
    progress: 310.0 s, 652.8 tps, lat 1.531 ms stddev 9.790
    progress: 320.0 s, 640.9 tps, lat 1.560 ms stddev 9.876
    progress: 330.0 s, 739.5 tps, lat 1.352 ms stddev 9.229
    progress: 340.0 s, 749.0 tps, lat 1.335 ms stddev 9.146
    progress: 350.0 s, 644.6 tps, lat 1.551 ms stddev 9.852
    progress: 360.0 s, 649.4 tps, lat 1.539 ms stddev 9.815
    progress: 370.0 s, 646.1 tps, lat 1.528 ms stddev 9.727
    progress: 380.0 s, 658.3 tps, lat 1.519 ms stddev 9.749
    progress: 390.0 s, 646.7 tps, lat 1.546 ms stddev 9.830
    progress: 400.0 s, 653.7 tps, lat 1.529 ms stddev 9.764
    progress: 410.0 s, 642.3 tps, lat 1.556 ms stddev 9.867
    progress: 420.0 s, 641.1 tps, lat 1.559 ms stddev 9.871
    progress: 430.0 s, 658.2 tps, lat 1.519 ms stddev 9.745
    progress: 440.0 s, 615.3 tps, lat 1.625 ms stddev 10.075
    progress: 450.0 s, 611.1 tps, lat 1.636 ms stddev 10.103
    progress: 460.0 s, 679.7 tps, lat 1.471 ms stddev 9.603
    progress: 470.0 s, 633.4 tps, lat 1.578 ms stddev 9.939
    progress: 480.0 s, 643.8 tps, lat 1.553 ms stddev 9.863
    progress: 490.0 s, 640.0 tps, lat 1.562 ms stddev 9.850
    progress: 500.0 s, 679.3 tps, lat 1.472 ms stddev 9.602
    progress: 510.0 s, 638.6 tps, lat 1.565 ms stddev 9.892
    progress: 520.0 s, 658.8 tps, lat 1.517 ms stddev 9.749
    progress: 530.0 s, 652.3 tps, lat 1.532 ms stddev 9.794
    progress: 540.0 s, 642.1 tps, lat 1.557 ms stddev 9.863
    progress: 550.0 s, 635.2 tps, lat 1.574 ms stddev 9.918
    progress: 560.0 s, 666.2 tps, lat 1.500 ms stddev 9.674
    progress: 570.0 s, 645.7 tps, lat 1.548 ms stddev 9.863
    progress: 580.0 s, 638.5 tps, lat 1.566 ms stddev 9.892
    progress: 590.0 s, 640.2 tps, lat 1.561 ms stddev 9.884
     progress: 600.0 s, 637.3 tps, lat 1.569 ms stddev 9.859
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 1
    number of threads: 1
    duration: 600 s
    number of transactions actually processed: 463118
    latency average = 1.294 ms
    latency stddev = 8.039 ms
    initial connection time = 5.036 ms
    tps = 771.868871 (without initial connection time)

    Средний TPS вырос, потому что в режиме synchronous_commit = off не дожидаемся от сервера фиксации транзакции в WAL

    

20. Создадим новый кластер с включенными контрольными суммами

    sudo pg_createcluster 14 main -- --data-checksums

21. Создадим новую базу с таблицей и запишем в неё пару строк

    CREATE DATABASE testdb;

    testdb=# SHOW data_checksums;
     data_checksums
    ----------------
     on
    (1 row)

    Подключимся к базе

    \c testdb

    CREATE TABLE testtable (c1 int, c2 text);

    INSERT INTO testtable (c1,c2) VALUES (1, 'ivan'), (2, 'pavel');

    SELECT * FROM testtable;

     c1 |  c2
    ----+-------
      1 | ivan
      2 | pavel

22. Посмотрим расположение таблицы SELECT pg_relation_filepath('testtable');

     pg_relation_filepath
    ----------------------
     base/16384/16395
    (1 row)

23. Выключаем кластер

    sudo pg_ctlcluster 14 main stop

24. Поменяем несколько байтов в нулевой странице, например сотрем из заголовка LSN последней журнальной записи

    sudo su

    dd if=/dev/zero of=/var/lib/postgresql/14/main/base/16384/16395 oflag=dsync conv=notrunc bs=1 count=8

25. Включаем кластер

    sudo pg_ctlcluster 14 main start

    \c testdb

    SELECT * FROM testtable;

    Получаем

    WARNING:  page verification failed, calculated checksum 1414 but expected 18432
    ERROR:  invalid page in block 0 of relation base/16384/16395

    Включем игнорирование ошибок контрольных сумм

     SET ignore_checksum_failure = on;

    SELECT * FROM testtable;

    WARNING:  page verification failed, calculated checksum 1414 but expected 18432
     c1 |  c2
    ----+-------
      1 | ivan
      2 | pavel
    (2 rows)





