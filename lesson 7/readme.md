# **Введение**

Логический уровень PostgreSQL

# **Подготовка**

Создадим ВМ и установим postgres, как в предыдущих уроках.

# **Выполнение**

создайте новую базу данных testdb

```
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```

зайдите в созданную базу данных под пользователем postgres

```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=#
```

создайте новую схему testnm

```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```
создайте новую таблицу t1 с одной колонкой c1 типа integer

```
testdb=# CREATE TABLE t1(c1 int);
CREATE TABLE
```

вставьте строку со значением c1=1
```
testdb=# insert into t1(c1) values(1);
```
создайте новую роль readonly

```
testdb=# create role readonly;
CREATE ROLE
```
дайте новой роли право на подключение к базе данных testdb

```
testdb=# grant connect on DATABASE testdb TO readonly;
GRANT
```
дайте новой роли право на использование схемы testnm
```
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
```
дайте новой роли право на select для всех таблиц схемы testnm
```
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
создайте пользователя testread с паролем test123
```
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
```
дайте роль readonly пользователю testread
```
testdb=# GRANT readonly TO testread ;
GRANT ROLE
```
зайдите под пользователем testread в базу данных testdb
```
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
```
Вход через сокет закрыт, нет пользователя testread в ОС, зайдем через пароль.
```
root@lesson-7:/home/snekrasov# psql -U testread -W -d testdb -h 127.0.0.1
Password:
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=>
```
сделайте select * from t1;
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
Таблица в namespace public, а нас доступ только к testnm
```
testdb=> \dt+
                                     List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+------+-------+----------+-------------+---------------+------------+-------------
 public | t1   | table | postgres | permanent   | heap          | 8192 bytes |
(1 row)
```
Для роли нет соответсвия $user таблица попала в public

вернитесь в базу данных testdb под пользователем postgres
```
testdb=> exit
root@lesson-7:/home/snekrasov# sudo -u postgres psql
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
Type "help" for help.

postgres=#
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
удалите таблицу t1
```
testdb=# drop table t1;
DROP TABLE
```
создайте ее заново но уже с явным указанием имени схемы testnm
```
testdb=# CREATE TABLE testnm.t1(c1 int);
CREATE TABLE
```
вставьте строку со значением c1=1

```
CREATE TABLE
testdb=# insert into testnm.t1(c1) values(1);
INSERT 0 1
```
зайдите под пользователем testread в базу данных testdb
```
root@lesson-7:/home/snekrasov# psql -U testread -W -d testdb -h 127.0.0.1
Password:
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```
сделайте select * from testnm.t1;

```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Таблица была создана и по умолчанию не пременилиь права выданные ранее.
```
testdb=> \dp testnm.t1
                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 testnm | t1   | table |                   |                   |
(1 row)
```

Зайдем под пользователем postgres и установим привелегии по умолчанию.
```
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
```
тогда для новых таблиц они будут применятся сразу.
Для текущей таблицы добавим снова.

```
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

сделайте select * from testnm.t1;

```
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
Получилось!

теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

```
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```
Получилось.
```
testdb=> \dt t2
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t2   | table | testread
(1 row)
```
таблица находится в public
```
testdb=> show search_path;
   search_path
-----------------
 "$user", public
(1 row)

testdb=>
```
search_path указывает на public, эта роль есть у всех пользователей.
запретим создание для роли public в база testdb
```
testdb=# REVOKE CREATE ON SCHEMA public FROM public;
REVOKE
testdb=# REVOKE ALL ON DATABASE testdb FROM public;
REVOKE
```
теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);


```
testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
INSERT 0 1
```
Нехватает прав на схему public.