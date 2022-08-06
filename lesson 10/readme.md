# **Введение**

Домашнее задание на "Журналы".


# **Выполнение**

Настройте выполнение контрольной точки раз в 30 секунд.

```
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# show checkpoint_timeout ;
 checkpoint_timeout
--------------------
 5min
(1 строка)

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

postgres=# show checkpoint_timeout ;
 checkpoint_timeout
--------------------
 30s
(1 строка)

postgres=# CREATE DATABASE wal_tmp;
CREATE DATABASE
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 2/7230FB80
(1 строка)
```

10 минут c помощью утилиты pgbench подавайте нагрузку.

```
root@Saturn:/home/eplat4m# sudo -u postgres pgbench -i wal_tmp
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.31 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.14 s, vacuum 0.07 s, primary keys 0.09 s).
```
```
root@Saturn:/home/eplat4m# sudo -u postgres pgbench -P 60 -T 600 wal_tmp
pgbench (14.4 (Debian 14.4-1.pgdg90+1))
starting vacuum...end.
progress: 60.0 s, 457.0 tps, lat 2.188 ms stddev 1.228
progress: 120.0 s, 406.4 tps, lat 2.460 ms stddev 1.225
progress: 180.0 s, 464.3 tps, lat 2.153 ms stddev 1.160
progress: 240.0 s, 459.2 tps, lat 2.177 ms stddev 1.220
progress: 300.0 s, 459.6 tps, lat 2.175 ms stddev 1.186
progress: 360.0 s, 443.0 tps, lat 2.257 ms stddev 1.188
progress: 420.0 s, 422.3 tps, lat 2.368 ms stddev 1.239
progress: 480.0 s, 437.6 tps, lat 2.285 ms stddev 1.254
progress: 540.0 s, 476.8 tps, lat 2.097 ms stddev 1.168
progress: 600.0 s, 380.9 tps, lat 2.625 ms stddev 1.308
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 264430
latency average = 2.269 ms
latency stddev = 1.225 ms
initial connection time = 3.595 ms
tps = 440.718965 (without initial connection time)
```
Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

```
postgres=# select  '2/8AE55C40'::pg_lsn - '2/7230FB80'::pg_lsn;
 ?column?
-----------
 414474432

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 7232
checkpoints_req       | 598
checkpoint_write_time | 1414864
checkpoint_sync_time  | 56522
buffers_checkpoint    | 184552
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3365445
buffers_backend_fsync | 0
buffers_alloc         | 223558
stats_reset           | 2022-07-12 16:56:10.008303+05
```
В среднем 20 723 721 байт на контрольную точку, выполнились все контрльные точки 7212 - 7232 = 20.

Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

В синхронном режиме у нас было tps = 440.718965
Переключим в асинхронный

```
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 строка)
```
Запустим pgbench

```
root@Saturn:/home/eplat4m# sudo -u postgres pgbench -P 60 -T 600 wal_tmp
pgbench (14.4 (Debian 14.4-1.pgdg90+1))
starting vacuum...end.
progress: 60.0 s, 1936.4 tps, lat 0.516 ms stddev 0.083
progress: 120.0 s, 1906.7 tps, lat 0.524 ms stddev 0.079
progress: 180.0 s, 1929.0 tps, lat 0.518 ms stddev 0.077
progress: 240.0 s, 1919.5 tps, lat 0.521 ms stddev 0.087
progress: 300.0 s, 1938.0 tps, lat 0.516 ms stddev 0.075
progress: 360.0 s, 1937.5 tps, lat 0.516 ms stddev 0.077
progress: 420.0 s, 1871.4 tps, lat 0.534 ms stddev 0.098
progress: 480.0 s, 1903.7 tps, lat 0.525 ms stddev 0.087
progress: 540.0 s, 1921.3 tps, lat 0.520 ms stddev 0.075
progress: 600.0 s, 1912.4 tps, lat 0.523 ms stddev 0.086
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
duration: 600 s
number of transactions actually processed: 1150557
latency average = 0.521 ms
latency stddev = 0.083 ms
initial connection time = 3.352 ms
tps = 1917.605298 (without initial connection time)
```

Прирост более 4х раз.

Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

```
root@Saturn:/home/eplat4m# pg_createcluster 14 second -- --data-checksums                                               Creating new PostgreSQL cluster 14/second ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/second --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных включён.

исправление прав для существующего каталога /var/lib/postgresql/14/second... ок
создание подкаталогов... ок
выбирается реализация динамической разделяемой памяти... posix
выбирается значение max_connections по умолчанию... 100
выбирается значение shared_buffers по умолчанию... 128MB
выбирается часовой пояс по умолчанию... Asia/Yekaterinburg
создание конфигурационных файлов... ок
выполняется подготовительный скрипт... ок
выполняется заключительная инициализация... ок
сохранение данных на диске... ок
Ver Cluster Port Status Owner    Data directory                Log file
14  second  5433 down   postgres /var/lib/postgresql/14/second /var/log/postgresql/postgresql-14-second.log
root@Saturn:/home/eplat4m# pg_ctlcluster 14 second start                                                                root@Saturn:/home/eplat4m# pg_lsclusters                                                                                Ver Cluster Port Status Owner    Data directory                Log file
14  main    5432 online postgres /var/lib/postgresql/14/main   /var/log/postgresql/postgresql-14-main.log
14  second  5433 online postgres /var/lib/postgresql/14/second /var/log/postgresql/postgresql-14-second.log
```

Подключимся и проверим

```
root@Saturn:/home/eplat4m# sudo -u postgres psql -p 5433
psql (14.4 (Debian 14.4-1.pgdg90+1))
Введите "help", чтобы получить справку.

postgres=# show data_checksums ;
 data_checksums
----------------
 on
(1 строка)
```
Добавим данные
```
postgres=# CREATE TABLE test_text(t text);
CREATE TABLE
postgres=# INSERT INTO test_text SELECT 'строка '||s.id FROM generate_series(1,500) AS s(id);
INSERT 0 500
postgres=# select count(*) from test_text ;
 count
-------
   500
(1 строка)
```
Измените пару байт в таблице.

Расположение файла
```
postgres=# select pg_relation_filepath('test_text');
 pg_relation_filepath
----------------------
 base/13672/16384
(1 строка)
```
Остановим кластер и изменим байты.

```
root@Saturn:/home/eplat4m# pg_ctlcluster 14 second stop
root@Saturn:/home/eplat4m# dd if=/dev/zero of=/var/lib/postgresql/14/second/base/13672/1638 oflag=dsync conv=notrunc bs=1 count=8
16384      16384_fsm  16387      16388
root@Saturn:/home/eplat4m# dd if=/dev/zero of=/var/lib/postgresql/14/second/base/13672/16384 oflag=dsync conv=notrunc bs=1 count=8
8+0 записей получено
8+0 записей отправлено
8 байт скопировано, 0,0236234 s, 0,3 kB/s
```
Стартуем
```
root@Saturn:/home/eplat4m# pg_ctlcluster 14 second start
```
Проверим данные
```
postgres=# select * from test_text ;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 15585, а ожидалась - 16298
ОШИБКА:  неверная страница в блоке 0 отношения base/13672/16384
```
Ошибка, попробуем исправить.
```
postgres=# vacuum test_text ;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 15585, а ожидалась - 16298
ОШИБКА:  неверная страница в блоке 0 отношения base/13672/16384
КОНТЕКСТ:  при сканировании блока 0 отношения "public.test_text"
postgres=# reindex table test_text ;
REINDEX
postgres=# select * from test_text ;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 15585, а ожидалась - 16298
ОШИБКА:  неверная страница в блоке 0 отношения base/13672/16384
```
Отключим проверку контрольной суммы.
```
postgres=# alter system set ignore_checksum_failure = on;
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 строка)

postgres=# select count(*) from test_text;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 15585, а ожидалась - 16298
 count
-------
   500
(1 строка)
```
Работает хотя выданно предупреждение.

