1. С типовыми настройками

 pgbench  -c 10 -P 60 -T 900 -U postgres postgres

progress: 60.0 s, 801.4 tps, lat 12.441 ms stddev 7.457
progress: 120.0 s, 785.1 tps, lat 12.717 ms stddev 9.884
progress: 180.0 s, 771.2 tps, lat 12.951 ms stddev 7.482
progress: 240.0 s, 779.9 tps, lat 12.806 ms stddev 7.389
progress: 300.0 s, 711.4 tps, lat 14.036 ms stddev 13.894
progress: 360.0 s, 482.9 tps, lat 20.671 ms stddev 28.955
progress: 420.0 s, 471.5 tps, lat 21.161 ms stddev 29.090
progress: 480.0 s, 479.0 tps, lat 20.843 ms stddev 28.922
progress: 540.0 s, 481.7 tps, lat 20.731 ms stddev 28.763
progress: 600.0 s, 486.8 tps, lat 20.505 ms stddev 28.915
progress: 660.0 s, 485.8 tps, lat 20.546 ms stddev 28.855
progress: 720.0 s, 460.4 tps, lat 21.658 ms stddev 32.124
progress: 780.0 s, 491.4 tps, lat 20.313 ms stddev 28.680
progress: 840.0 s, 503.4 tps, lat 19.827 ms stddev 28.587
progress: 900.0 s, 498.7 tps, lat 20.005 ms stddev 29.132
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 900 s
number of transactions actually processed: 521447
latency average = 17.228 ms
latency stddev = 23.008 ms
initial connection time = 87.186 ms
tps = 579.425350 (without initial connection time)

2. Получим при помощи PGTune "оптимальные параметры"

```ini
max_connections = 20
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 52428kB
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
```

Подключим их через параметр include_dir в файле postgresql.conf, изменим права и владельца и перезагрузим кластер

sudo chmod 644 /etc/postgresql/14/main/conf.d/postgresqladdconf1.conf && sudo chown postgres:postgres /etc/postgresql/14/main/conf.d/postgresqladdconf1.conf && sudo pg_ctlcluster 14 main restart

Проверим что кластер перестартовал и работает

pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Проверим что конфигурация применилась. Подключимся к кластеру и получим несколько параметров:

psql postgres postgres

\x

SHOW max_connections;

-[ RECORD 1 ]---+---
max_connections | 20

SHOW shared_buffers;

-[ RECORD 1 ]--+----
shared_buffers | 1GB

Параметры применились. 

3. Повторим тест pgbench -c 10 -P 60 -T 900 -U postgres postgres

starting vacuum...end.
progress: 60.0 s, 800.0 tps, lat 12.463 ms stddev 7.456
progress: 120.0 s, 816.0 tps, lat 12.240 ms stddev 7.297
progress: 180.0 s, 820.8 tps, lat 12.168 ms stddev 7.166
progress: 240.0 s, 802.0 tps, lat 12.453 ms stddev 7.155
progress: 300.0 s, 691.1 tps, lat 14.423 ms stddev 16.008
progress: 360.0 s, 504.1 tps, lat 19.811 ms stddev 28.396
progress: 420.0 s, 516.5 tps, lat 19.320 ms stddev 27.795
progress: 480.0 s, 506.6 tps, lat 19.702 ms stddev 28.525
progress: 540.0 s, 500.5 tps, lat 19.930 ms stddev 30.594
progress: 600.0 s, 521.2 tps, lat 19.147 ms stddev 27.783
progress: 660.0 s, 502.2 tps, lat 19.870 ms stddev 28.223
progress: 720.0 s, 520.2 tps, lat 19.190 ms stddev 27.917
progress: 780.0 s, 518.0 tps, lat 19.259 ms stddev 28.330
progress: 840.0 s, 523.2 tps, lat 19.065 ms stddev 27.860
progress: 900.0 s, 527.0 tps, lat 18.930 ms stddev 28.095
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 900 s
number of transactions actually processed: 544178
latency average = 16.507 ms
latency stddev = 22.485 ms
initial connection time = 89.865 ms
tps = 604.680578 (without initial connection time)

Среднее значение TPS немного увеличилось

4. Изменим значение shared_buffers до 2GB, перезагрузим кластер  и повторим тест

   pgbench -c 10 -P 60 -T 900 -U postgres postgres


pgbench (14.2 (Debian 14.2-1.pgdg100+1))
starting vacuum...end.
progress: 60.0 s, 812.9 tps, lat 12.272 ms stddev 9.228
progress: 120.0 s, 818.6 tps, lat 12.206 ms stddev 6.465
progress: 180.0 s, 807.2 tps, lat 12.379 ms stddev 6.607
progress: 240.0 s, 839.3 tps, lat 11.906 ms stddev 6.283
progress: 300.0 s, 814.9 tps, lat 12.261 ms stddev 6.528
progress: 360.0 s, 621.4 tps, lat 16.052 ms stddev 19.769
progress: 420.0 s, 507.8 tps, lat 19.658 ms stddev 27.129
progress: 480.0 s, 492.4 tps, lat 20.286 ms stddev 27.102
progress: 540.0 s, 494.0 tps, lat 20.222 ms stddev 27.796
progress: 600.0 s, 488.9 tps, lat 20.418 ms stddev 29.110
progress: 660.0 s, 508.9 tps, lat 19.621 ms stddev 28.069
progress: 720.0 s, 508.2 tps, lat 19.651 ms stddev 27.105
progress: 780.0 s, 503.9 tps, lat 19.802 ms stddev 27.204
progress: 840.0 s, 503.7 tps, lat 19.828 ms stddev 27.282
progress: 900.0 s, 512.2 tps, lat 19.488 ms stddev 27.144
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 900 s
number of transactions actually processed: 554095
latency average = 16.222 ms
latency stddev = 20.836 ms
initial connection time = 87.046 ms
tps = 615.686089 (without initial connection time)



5. Увеличим shared_buffers до 3.5GB (3584MB) и повторим тест

pgbench -c 10 -P 60 -T 900 -U postgres postgres

pgbench (14.2 (Debian 14.2-1.pgdg100+1))
starting vacuum...end.
progress: 60.0 s, 774.8 tps, lat 12.873 ms stddev 7.402
progress: 120.0 s, 797.6 tps, lat 12.526 ms stddev 6.635
progress: 180.0 s, 809.3 tps, lat 12.344 ms stddev 9.125
progress: 240.0 s, 797.3 tps, lat 12.531 ms stddev 6.634
progress: 300.0 s, 824.0 tps, lat 12.128 ms stddev 6.335
progress: 360.0 s, 538.0 tps, lat 18.551 ms stddev 24.556
progress: 420.0 s, 521.1 tps, lat 19.159 ms stddev 26.634
progress: 480.0 s, 522.5 tps, lat 19.116 ms stddev 26.948
progress: 540.0 s, 521.0 tps, lat 19.156 ms stddev 27.043
progress: 600.0 s, 523.4 tps, lat 19.088 ms stddev 26.863
progress: 660.0 s, 515.4 tps, lat 19.378 ms stddev 27.521
progress: 720.0 s, 498.2 tps, lat 20.035 ms stddev 27.560
progress: 780.0 s, 510.3 tps, lat 19.568 ms stddev 30.184
progress: 840.0 s, 519.7 tps, lat 19.221 ms stddev 27.086
progress: 900.0 s, 512.7 tps, lat 19.467 ms stddev 27.175
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
duration: 900 s
number of transactions actually processed: 551120
latency average = 16.308 ms
latency stddev = 21.242 ms
initial connection time = 90.797 ms
tps = 612.398108 (without initial connection time)

6. Изменим значение max_worker_processes на 1

   pgbench -c 10 -P 60 -T 900 -U postgres postgres
   
   pgbench (14.2 (Debian 14.2-1.pgdg100+1))
   starting vacuum...end.
   progress: 60.0 s, 836.8 tps, lat 11.916 ms stddev 6.969
   progress: 120.0 s, 835.7 tps, lat 11.952 ms stddev 6.790
   progress: 180.0 s, 796.0 tps, lat 12.550 ms stddev 7.164
   progress: 240.0 s, 849.4 tps, lat 11.759 ms stddev 6.985
   progress: 300.0 s, 734.1 tps, lat 13.600 ms stddev 16.486
   progress: 360.0 s, 471.9 tps, lat 21.149 ms stddev 30.824
   progress: 420.0 s, 512.8 tps, lat 19.467 ms stddev 27.946
   progress: 480.0 s, 526.2 tps, lat 18.979 ms stddev 27.534
   progress: 540.0 s, 529.8 tps, lat 18.844 ms stddev 27.588
   progress: 600.0 s, 527.6 tps, lat 18.920 ms stddev 27.649
   progress: 660.0 s, 521.6 tps, lat 19.138 ms stddev 27.744
   progress: 720.0 s, 536.6 tps, lat 18.594 ms stddev 27.535
   progress: 780.0 s, 532.4 tps, lat 18.749 ms stddev 27.831
   progress: 840.0 s, 537.7 tps, lat 18.556 ms stddev 27.625
   progress: 900.0 s, 531.7 tps, lat 18.771 ms stddev 27.582
   transaction type: <builtin: TPC-B (sort of)>
   scaling factor: 1
   query mode: simple
   number of clients: 10
   number of threads: 1
   duration: 900 s
   number of transactions actually processed: 556819
   latency average = 16.135 ms
   latency stddev = 22.167 ms
   initial connection time = 89.717 ms
   tps = 618.664013 (without initial connection time)
   
   
   
   7. Изменим значение shared_buffers до 2 GB
   
      pgbench -c 10 -P 60 -T 900 -U postgres postgres
   
      progress: 60.0 s, 824.0 tps, lat 12.099 ms stddev 7.039
      progress: 120.0 s, 834.1 tps, lat 11.969 ms stddev 10.180
      progress: 180.0 s, 810.0 tps, lat 12.330 ms stddev 7.219
      progress: 240.0 s, 810.9 tps, lat 12.315 ms stddev 7.294
      progress: 300.0 s, 661.2 tps, lat 15.099 ms stddev 18.563
      progress: 360.0 s, 518.1 tps, lat 19.278 ms stddev 28.189
      progress: 420.0 s, 504.7 tps, lat 19.786 ms stddev 28.302
      progress: 480.0 s, 523.5 tps, lat 19.066 ms stddev 27.962
      progress: 540.0 s, 503.8 tps, lat 19.819 ms stddev 28.263
      progress: 600.0 s, 524.4 tps, lat 19.034 ms stddev 27.826
      progress: 660.1 s, 517.8 tps, lat 19.233 ms stddev 28.151
      progress: 720.0 s, 524.9 tps, lat 19.053 ms stddev 30.067
      progress: 780.0 s, 530.8 tps, lat 18.803 ms stddev 27.581
      progress: 840.0 s, 522.7 tps, lat 19.093 ms stddev 28.174
      progress: 900.0 s, 519.2 tps, lat 19.210 ms stddev 28.128
      transaction type: <builtin: TPC-B (sort of)>
      scaling factor: 1
      query mode: simple
      number of clients: 10
      number of threads: 1
      duration: 900 s
      number of transactions actually processed: 547820
      latency average = 16.400 ms
      latency stddev = 22.668 ms
      initial connection time = 97.640 ms
      tps = 608.669522 (without initial connection time)
   
   8. Изменим shared_buffers до 3GB
   
      pgbench -c 10 -P 60 -T 900 -U postgres postgres
   
      pgbench (14.2 (Debian 14.2-1.pgdg100+1))
      starting vacuum...end.
      progress: 60.0 s, 835.2 tps, lat 11.937 ms stddev 6.985
      progress: 120.0 s, 818.2 tps, lat 12.207 ms stddev 6.916
      progress: 180.0 s, 814.9 tps, lat 12.258 ms stddev 7.121
      progress: 240.0 s, 824.7 tps, lat 12.111 ms stddev 6.967
      progress: 300.0 s, 759.8 tps, lat 13.120 ms stddev 12.019
      progress: 360.0 s, 505.1 tps, lat 19.776 ms stddev 28.094
      progress: 420.0 s, 497.9 tps, lat 20.050 ms stddev 28.035
      progress: 480.0 s, 481.0 tps, lat 20.762 ms stddev 28.312
      progress: 540.0 s, 500.0 tps, lat 19.938 ms stddev 30.435
      progress: 600.0 s, 508.4 tps, lat 19.627 ms stddev 27.806
      progress: 660.0 s, 538.0 tps, lat 18.561 ms stddev 27.596
      progress: 720.0 s, 541.2 tps, lat 18.450 ms stddev 27.299
      progress: 780.0 s, 523.6 tps, lat 19.049 ms stddev 27.995
      progress: 840.0 s, 524.9 tps, lat 19.014 ms stddev 27.815
      progress: 900.0 s, 530.6 tps, lat 18.811 ms stddev 27.864
      transaction type: <builtin: TPC-B (sort of)>
      scaling factor: 1
      query mode: simple
      number of clients: 10
      number of threads: 1
      duration: 900 s
      number of transactions actually processed: 552227
      latency average = 16.269 ms
      latency stddev = 22.019 ms
      initial connection time = 95.340 ms
      tps = 613.628484 (without initial connection time)
   
      9. Выключим synchronous_commit
   
         pgbench -c 10 -P 60 -T 900 -U postgres postgres
   
         pgbench (14.2 (Debian 14.2-1.pgdg100+1))
         starting vacuum...end.
         progress: 60.0 s, 2110.2 tps, lat 4.698 ms stddev 2.948
         progress: 120.0 s, 2134.8 tps, lat 4.650 ms stddev 2.931
         progress: 180.0 s, 1039.0 tps, lat 9.530 ms stddev 24.759
         progress: 240.0 s, 1004.6 tps, lat 9.863 ms stddev 25.273
         progress: 300.0 s, 992.6 tps, lat 9.984 ms stddev 25.560
         progress: 360.0 s, 1032.6 tps, lat 9.615 ms stddev 24.918
         progress: 420.0 s, 1006.2 tps, lat 9.858 ms stddev 25.187
         progress: 480.0 s, 1026.1 tps, lat 9.677 ms stddev 24.958
         progress: 540.0 s, 1026.2 tps, lat 9.663 ms stddev 24.940
         progress: 600.0 s, 1022.9 tps, lat 9.709 ms stddev 25.023
         progress: 660.0 s, 1015.6 tps, lat 9.770 ms stddev 25.074
         progress: 720.0 s, 1003.5 tps, lat 9.880 ms stddev 25.188
         progress: 780.0 s, 1011.4 tps, lat 9.808 ms stddev 25.120
         progress: 840.0 s, 988.3 tps, lat 10.038 ms stddev 25.484
         progress: 900.0 s, 956.2 tps, lat 10.344 ms stddev 26.046
         transaction type: <builtin: TPC-B (sort of)>
         scaling factor: 1
         query mode: simple
         number of clients: 10
         number of threads: 1
         duration: 900 s
         number of transactions actually processed: 1042257
         latency average = 8.565 ms
         latency stddev = 22.060 ms
         initial connection time = 87.331 ms
         tps = 1158.122441 (without initial connection time)
   
         10. Выключим fsync
   
             pgbench (14.2 (Debian 14.2-1.pgdg100+1))
             starting vacuum...end.
             progress: 60.0 s, 2068.5 tps, lat 4.792 ms stddev 3.019
             progress: 120.0 s, 2100.2 tps, lat 4.728 ms stddev 2.951
             progress: 180.0 s, 1025.9 tps, lat 9.675 ms stddev 24.937
             progress: 240.0 s, 1020.0 tps, lat 9.709 ms stddev 24.961
             progress: 300.0 s, 1022.9 tps, lat 9.699 ms stddev 24.959
             progress: 360.0 s, 1027.2 tps, lat 9.654 ms stddev 24.861
             progress: 420.0 s, 979.5 tps, lat 10.124 ms stddev 25.699
             progress: 480.0 s, 1026.7 tps, lat 9.670 ms stddev 24.950
             progress: 540.0 s, 1017.3 tps, lat 9.748 ms stddev 25.027
             progress: 600.0 s, 1019.4 tps, lat 9.728 ms stddev 24.997
             progress: 660.0 s, 1033.6 tps, lat 9.596 ms stddev 24.864
             progress: 720.0 s, 1029.9 tps, lat 9.637 ms stddev 24.857
             progress: 780.0 s, 1029.3 tps, lat 9.637 ms stddev 24.860
             progress: 840.0 s, 1037.0 tps, lat 9.571 ms stddev 24.833
             progress: 900.0 s, 1022.6 tps, lat 9.708 ms stddev 24.983
             transaction type: <builtin: TPC-B (sort of)>
             scaling factor: 1
             query mode: simple
             number of clients: 10
             number of threads: 1
             duration: 900 s
             number of transactions actually processed: 1047607
             latency average = 8.522 ms
             latency stddev = 21.947 ms
             initial connection time = 89.646 ms
             tps = 1164.086012 (without initial connection time)
   
             
   
             
   
             Резюме: в основном менялось значение shared_buffers в сторону увеличения от рекомендованного RAM \ 4 и уменьшения количества воркеров autovacuum (для снижения нагрузки на ЦП)
   
             Увеличение значения TPS почти в 2 раза удалось достичь после отключения symchronous_commit: с 618 до 1158
   
             
   
             
   
         
   
      
   
   
   
   











