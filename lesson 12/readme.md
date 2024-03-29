# **Введение**

Домашнее задание на "Настройка PostgreSQL".

# **Выполнение**

Установим постгрес как в прошлых уроках.
Параметры виртуалки
RAM 2Gb
HDD 15Gb
2 CPU

Уcтановим sysbench
```
root@tuning:/home/snekrasov# curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
root@tuning:/home/snekrasov# sudo apt -y install sysbench
```

Добавим базу и пользователя для тестов

```
postgres=# CREATE DATABASE sbtest;
CREATE DATABASE
postgres=# CREATE USER sbtest WITH PASSWORD 'P@ssw0rd';
CREATE ROLE
postgres=# GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
GRANT
```
Подготовим данные
```
sysbench \
--db-driver=pgsql \
--oltp-table-size=100000 \
--oltp-tables-count=24 \
--threads=2 \
--pgsql-host=localhost \
--pgsql-port=5432 \
--pgsql-user=sbtest \
--pgsql-password=P@ssw0rd \
--pgsql-db=sbtest \
/usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua \
run
...

Creating secondary indexes on 'sbtest18'...
Creating table 'sbtest19'...
Inserting 100000 records into 'sbtest19'
Creating secondary indexes on 'sbtest19'...
Creating table 'sbtest20'...
Inserting 100000 records into 'sbtest20'
Creating secondary indexes on 'sbtest20'...
Creating table 'sbtest21'...
Inserting 100000 records into 'sbtest21'
Creating secondary indexes on 'sbtest21'...
Creating table 'sbtest22'...
Inserting 100000 records into 'sbtest22'
Creating secondary indexes on 'sbtest22'...
Creating table 'sbtest23'...
Inserting 100000 records into 'sbtest23'
Creating secondary indexes on 'sbtest23'...
Creating table 'sbtest24'...
Inserting 100000 records into 'sbtest24'
Creating secondary indexes on 'sbtest24'...
```
Проверим наличие данных для тестов

```
sbtest=# \dt+
                                   List of relations
 Schema |   Name   | Type  | Owner  | Persistence | Access method | Size  | Description
--------+----------+-------+--------+-------------+---------------+-------+-------------
 public | sbtest1  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest10 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest11 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest12 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest13 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest14 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest15 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest16 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest17 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest18 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest19 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest2  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest20 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest21 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest22 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest23 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest24 | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest3  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest4  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest5  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest6  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest7  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest8  | table | sbtest | permanent   | heap          | 21 MB |
 public | sbtest9  | table | sbtest | permanent   | heap          | 21 MB |
(24 rows)
```
Запуск без тюнинга.
```
sysbench \
--db-driver=pgsql \
--report-interval=2 \
--oltp-table-size=100000 \
--oltp-tables-count=24 \
--threads=2 \
--time=60 \
--pgsql-host=localhost \
--pgsql-user=sbtest  \
--pgsql-password=P@ssw0rd \
--pgsql-port=5432 \
--pgsql-db=sbtest \
/usr/share/sysbench/tests/include/oltp_legacy/oltp.lua \
run

...
SQL statistics:
    queries performed:
        read:                            178584
        write:                           51023
        other:                           25513
        total:                           255120
    transactions:                        12756  (212.52 per sec.)
    queries:                             255120 (4250.39 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0209s
    total number of events:              12756

Latency (ms):
         min:                                    3.18
         avg:                                    9.40
         max:                                  616.48
         95th percentile:                       30.81
         sum:                               119965.92

```
Воспользуюсь pgtune

```
# DB Version: 14
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 2 GB
# CPUs num: 2
# Data Storage: hdd

ALTER SYSTEM SET
 max_connections = '200';
ALTER SYSTEM SET
 shared_buffers = '512MB';
ALTER SYSTEM SET
 effective_cache_size = '1536MB';
ALTER SYSTEM SET
 maintenance_work_mem = '128MB';
ALTER SYSTEM SET
 checkpoint_completion_target = '0.9';
ALTER SYSTEM SET
 wal_buffers = '16MB';
ALTER SYSTEM SET
 default_statistics_target = '100';
ALTER SYSTEM SET
 random_page_cost = '4';
ALTER SYSTEM SET
 effective_io_concurrency = '2';
ALTER SYSTEM SET
 work_mem = '2621kB';
ALTER SYSTEM SET
 min_wal_size = '1GB';
ALTER SYSTEM SET
 max_wal_size = '4GB';
ALTER SYSTEM SET
 max_worker_processes = '2';
ALTER SYSTEM SET
 max_parallel_workers_per_gather = '1';
ALTER SYSTEM SET
 max_parallel_workers = '2';
ALTER SYSTEM SET
 max_parallel_maintenance_workers = '1';
```

```
SQL statistics:
    queries performed:
        read:                            287420
        write:                           82120
        other:                           41060
        total:                           410600
    transactions:                        20530  (342.06 per sec.)
    queries:                             410600 (6841.22 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0122s
    total number of events:              20530

Latency (ms):
         min:                                    3.35
         avg:                                    5.84
         max:                                  150.84
         95th percentile:                        9.39
         sum:                               119948.02
```
Прирост tps ~50%

Увеличиваю shared_buffers = '1GB';

```
SQL statistics:
    queries performed:
        read:                            263368
        write:                           75248
        other:                           37624
        total:                           376240
    transactions:                        18812  (313.45 per sec.)
    queries:                             376240 (6269.03 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0136s
    total number of events:              18812

Latency (ms):
         min:                                    3.42
         avg:                                    6.38
         max:                                   58.08
         95th percentile:                       12.75
         sum:                               119947.93
```
Ухудшение

Увеличиваю shared_buffers = '768MB';

```
SQL statistics:
    queries performed:
        read:                            267722
        write:                           76492
        other:                           38246
        total:                           382460
    transactions:                        19123  (318.62 per sec.)
    queries:                             382460 (6372.46 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0157s
    total number of events:              19123

Latency (ms):
         min:                                    3.43
         avg:                                    6.27
         max:                                   56.75
         95th percentile:                       12.08
         sum:                               119953.85
```
Незначительный прирост, однако хуже чем 512MB.
Установлю 512MB

Попробуем увеличить количество потов для теста

```
sysbench \
--db-driver=pgsql \
--report-interval=2 \
--oltp-table-size=100000 \
--oltp-tables-count=24 \
--threads=100 \
--time=60 \
--pgsql-host=localhost \
--pgsql-user=sbtest  \
--pgsql-password=P@ssw0rd \
--pgsql-port=5432 \
--pgsql-db=sbtest \
/usr/share/sysbench/tests/include/oltp_legacy/oltp.lua \
run
```

```
SQL statistics:
queries performed:
read:                            320600
write:                           91576
other:                           45812
total:                           457988
transactions:                        22894  (377.63 per sec.)
queries:                             457988 (7554.48 per sec.)
ignored errors:                      6      (0.10 per sec.)
reconnects:                          0      (0.00 per sec.)

General statistics:
total time:                          60.6226s
total number of events:              22894

Latency (ms):
min:                                   10.93
avg:                                  262.72
max:                                 2809.68
95th percentile:                      320.17
sum:                              6014754.86
```
Изменения не существенны.
Увеличим effective_io_concurrency = 4

```
SQL statistics:
    queries performed:
        read:                            318430
        write:                           90943
        other:                           45515
        total:                           454888
    transactions:                        22739  (375.04 per sec.)
    queries:                             454888 (7502.61 per sec.)
    ignored errors:                      6      (0.10 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.6287s
    total number of events:              22739

Latency (ms):
         min:                                   63.93
         avg:                                  264.58
         max:                                 2991.02
         95th percentile:                      314.45
         sum:                              6016338.54
```
Небольшой прирост.
Увеличим effective_io_concurrency = 100
```
SQL statistics:
    queries performed:
        read:                            313222
        write:                           89462
        other:                           44760
        total:                           447444
    transactions:                        22365  (368.14 per sec.)
    queries:                             447444 (7365.23 per sec.)
    ignored errors:                      8      (0.13 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.7489s
    total number of events:              22365

Latency (ms):
         min:                                  108.60
         avg:                                  269.26
         max:                                 1989.12
         95th percentile:                      325.98
         sum:                              6022091.86
```
Потеря tps

# **Вывод**
Текущие настройки на коротких нагрузках 60с.
max_connections = '200';
shared_buffers = '512MB';
effective_cache_size = '1536MB';
maintenance_work_mem = '128MB';
checkpoint_completion_target = '0.9';
wal_buffers = '16MB';
default_statistics_target = '100';
random_page_cost = '4';
effective_io_concurrency = '4';
work_mem = '2621kB';
min_wal_size = '1GB';
max_wal_size = '4GB';
max_worker_processes = '2';
max_parallel_workers_per_gather = '1';
max_parallel_workers = '2';
max_parallel_maintenance_workers = '1';

Если нагрузка более продолжительна 3600c тогда начинают влиять настройки autovacuum
autovacuum_naptime = 120
autovacuum_vacuum_scale_factor 0.1
autovacuum_max_workers 2
autovacuum_vacuum_threshold  50

Резюмируя можно использовать настройки Pgtune в большинстве случаев.