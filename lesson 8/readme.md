# **Введение**

Домашняя работа "MVCC, vacuum и autovacuum"

# **Подготовка**

Создадим ВМ и установим postgres, как в предыдущих уроках.

# **Выполнение**

Установим параметры /etc/postgresql/14/main/postgresql.conf

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB

Инициализация

```
root@benchmark:/home/snekrasov# sudo -u postgres pgbench -i
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.50 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.32 s, vacuum 0.06 s, primary keys 0.11 s).
```

Настройки автовакума по умолчанию
```
                 name                  |  setting  |  context   |                                        short_desc
---------------------------------------+-----------+------------+-------------------------------------------------------------------------------------------
 autovacuum                            | on        | sighup     | Starts the autovacuum subprocess.
 autovacuum_analyze_scale_factor       | 0.1       | sighup     | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples.
 autovacuum_analyze_threshold          | 50        | sighup     | Minimum number of tuple inserts, updates, or deletes prior to analyze.
 autovacuum_freeze_max_age             | 200000000 | postmaster | Age at which to autovacuum a table to prevent transaction ID wraparound.
 autovacuum_max_workers                | 3         | postmaster | Sets the maximum number of simultaneously running autovacuum worker processes.
 autovacuum_multixact_freeze_max_age   | 400000000 | postmaster | Multixact age at which to autovacuum a table to prevent multixact wraparound.
 autovacuum_naptime                    | 60        | sighup     | Time to sleep between autovacuum runs.
 autovacuum_vacuum_cost_delay          | 2         | sighup     | Vacuum cost delay in milliseconds, for autovacuum.
 autovacuum_vacuum_cost_limit          | -1        | sighup     | Vacuum cost amount available before napping, for autovacuum.
 autovacuum_vacuum_insert_scale_factor | 0.2       | sighup     | Number of tuple inserts prior to vacuum as a fraction of reltuples.
 autovacuum_vacuum_insert_threshold    | 1000      | sighup     | Minimum number of tuple inserts prior to vacuum, or -1 to disable insert vacuums.
 autovacuum_vacuum_scale_factor        | 0.2       | sighup     | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.
 autovacuum_vacuum_threshold           | 50        | sighup     | Minimum number of tuple updates or deletes prior to vacuum.
 autovacuum_work_mem                   | -1        | sighup     | Sets the maximum memory to be used by each autovacuum worker process.
```

Первый прогон
```
progress: 3000.0 s, 411.1 tps, lat 19.441 ms stddev 14.103
progress: 3060.0 s, 616.1 tps, lat 12.961 ms stddev 9.305
progress: 3120.0 s, 558.3 tps, lat 14.307 ms stddev 10.497
progress: 3180.0 s, 629.8 tps, lat 12.676 ms stddev 9.173
progress: 3240.0 s, 601.4 tps, lat 13.282 ms stddev 9.670
progress: 3300.0 s, 595.2 tps, lat 13.416 ms stddev 10.404
progress: 3360.0 s, 601.4 tps, lat 13.278 ms stddev 9.960
progress: 3420.0 s, 602.6 tps, lat 13.253 ms stddev 9.812
progress: 3480.0 s, 623.0 tps, lat 12.816 ms stddev 9.456
progress: 3540.0 s, 571.7 tps, lat 13.970 ms stddev 10.106
progress: 3600.0 s, 578.0 tps, lat 13.818 ms stddev 10.616
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 2074952
latency average = 13.856 ms
latency stddev = 10.881 ms
initial connection time = 15.308 ms
tps = 576.373507 (without initial connection time)
```
Установим autovacuum_naptime = 120 и autovacuum_vacuum_scale_factor 0.1

```
progress: 3000.0 s, 318.4 tps, lat 25.093 ms stddev 11.913
progress: 3060.0 s, 319.4 tps, lat 25.016 ms stddev 11.611
progress: 3120.0 s, 317.2 tps, lat 25.201 ms stddev 12.319
progress: 3180.0 s, 320.8 tps, lat 24.918 ms stddev 11.709
progress: 3240.0 s, 320.2 tps, lat 24.955 ms stddev 11.527
progress: 3300.0 s, 319.5 tps, lat 25.011 ms stddev 11.825
progress: 3360.0 s, 320.9 tps, lat 24.901 ms stddev 11.641
progress: 3420.0 s, 317.2 tps, lat 25.189 ms stddev 11.446
progress: 3480.0 s, 321.0 tps, lat 24.890 ms stddev 10.762
progress: 3540.0 s, 320.3 tps, lat 24.951 ms stddev 12.556
progress: 3600.0 s, 320.5 tps, lat 24.936 ms stddev 11.917
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1798251
latency average = 15.991 ms
latency stddev = 11.488 ms
initial connection time = 15.834 ms
tps = 499.512448 (without initial connection time)
```

Установим autovacuum_naptime = 240 и autovacuum_vacuum_scale_factor 0.5

```
progress: 3000.0 s, 317.3 tps, lat 25.193 ms stddev 13.101
progress: 3060.0 s, 316.1 tps, lat 25.291 ms stddev 14.336
progress: 3120.0 s, 317.1 tps, lat 25.203 ms stddev 11.396
progress: 3180.0 s, 320.8 tps, lat 24.913 ms stddev 10.541
progress: 3240.0 s, 320.9 tps, lat 24.907 ms stddev 11.189
progress: 3300.0 s, 319.3 tps, lat 25.031 ms stddev 12.950
progress: 3360.0 s, 319.3 tps, lat 25.030 ms stddev 11.849
progress: 3420.0 s, 317.2 tps, lat 25.189 ms stddev 11.606
progress: 3480.0 s, 320.9 tps, lat 24.911 ms stddev 11.820
progress: 3540.0 s, 320.0 tps, lat 24.975 ms stddev 10.869
progress: 3600.0 s, 318.0 tps, lat 25.131 ms stddev 11.462
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1297874
latency average = 22.167 ms
latency stddev = 12.475 ms
initial connection time = 18.602 ms
tps = 360.519970 (without initial connection time)
```
Установим autovacuum_vacuum_scale_factor 0 и  autovacuum_vacuum_threshold 10000

```
progress: 3000.0 s, 319.7 tps, lat 25.001 ms stddev 12.152
progress: 3060.0 s, 318.4 tps, lat 25.100 ms stddev 12.374
progress: 3120.0 s, 317.8 tps, lat 25.156 ms stddev 12.345
progress: 3180.0 s, 320.7 tps, lat 24.928 ms stddev 12.543
progress: 3240.0 s, 319.5 tps, lat 25.024 ms stddev 12.507
progress: 3300.0 s, 319.9 tps, lat 24.984 ms stddev 11.348
progress: 3360.0 s, 320.0 tps, lat 24.974 ms stddev 10.787
progress: 3420.0 s, 316.6 tps, lat 25.242 ms stddev 11.642
progress: 3480.0 s, 316.9 tps, lat 25.216 ms stddev 11.474
progress: 3540.0 s, 313.4 tps, lat 25.495 ms stddev 12.696
progress: 3600.0 s, 317.8 tps, lat 25.164 ms stddev 16.640
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1157851
latency average = 24.850 ms
latency stddev = 12.079 ms
initial connection time = 13.908 ms
tps = 321.624385 (without initial connection time)
```
Установим autovacuum_vacuum_scale_factor 0.01, autovacuum_vacuum_threshold 0б autovacuum_max_workers 2 

```
progress: 3000.0 s, 320.4 tps, lat 24.949 ms stddev 11.029
progress: 3060.0 s, 320.3 tps, lat 24.951 ms stddev 11.097
progress: 3120.0 s, 317.6 tps, lat 25.163 ms stddev 12.818
progress: 3180.0 s, 315.3 tps, lat 25.348 ms stddev 12.177
progress: 3240.0 s, 320.8 tps, lat 24.918 ms stddev 11.889
progress: 3300.0 s, 319.0 tps, lat 25.058 ms stddev 12.482
progress: 3360.0 s, 319.2 tps, lat 25.046 ms stddev 12.212
progress: 3420.0 s, 318.5 tps, lat 25.090 ms stddev 10.921
progress: 3480.0 s, 319.3 tps, lat 25.031 ms stddev 11.481
progress: 3540.0 s, 319.5 tps, lat 25.021 ms stddev 13.345
progress: 3600.0 s, 319.2 tps, lat 25.047 ms stddev 13.266
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1158250
latency average = 24.841 ms
latency stddev = 11.892 ms
initial connection time = 14.396 ms
tps = 321.735435 (without initial connection time)
```

# **Вывод**

Наименьшее колебания tps достигается при

autovacuum_naptime = 120
autovacuum_vacuum_scale_factor 0.1
autovacuum_max_workers 3
autovacuum_vacuum_threshold  50
Остальные настройки по умолчанию 