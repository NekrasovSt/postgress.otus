# **Введение**

Домашнее задание на тему "Виды и устройство репликации в PostgreSQL. Практика применения".

# **Выполнение**

Создадим таблицы на ВМ.

```
postgres=# create table test1(c text);
CREATE TABLE
postgres=# create table test2(c text);
CREATE TABLE
```
Добавим пользователей

```
postgres=# CREATE USER repl WITH REPLICATION LOGIN PASSWORD 'P@ssw0rd';
CREATE ROLE
postgres=# GRANT ALL PRIVILEGES ON DATABASE postgres TO repl;
GRANT
```

Включаем логическую репликацию
```
postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM
```
Тоже самое на второй ВМ.

Создаем публикацию.

```
postgres=# CREATE PUBLICATION test_pub FOR TABLE test1;
CREATE PUBLICATION
```
Подписываемся на второй ВМ
```
postgres=# CREATE SUBSCRIPTION test_sub
postgres-# CONNECTION 'host=10.129.0.16 port=5432 user=repl password=P@ssw0rd dbname=postgres'
postgres-# PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
Все тоже самое для первой ВМ

```
postgres=# CREATE SUBSCRIPTION test_sub
postgres-# CONNECTION 'host=10.129.0.23 port=5432 user=repl password=P@ssw0rd dbname=postgres'
postgres-# PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
postgres=# \dRs
            List of subscriptions
   Name   |  Owner   | Enabled | Publication
----------+----------+---------+-------------
 test_sub | postgres | t       | {test_pub}
```

проверим делаем инсерт в первой ВМ

```
postgres=# insert into test1 (c) values('hello');
```
И смотрим во второй
```
postgres=# select * from test1;
   c
-------
 hello
(1 row)
```
так же проверим и test2.

```
insert into test2 (c) values ('world');
```
```
postgres=# select * from test2;
   c
-------
 world
(1 row)
```

Между ВМ1 и ВМ2 настроена логическая репликация для таблиц test1 и test2


## **Настойка для ВМ3**

Нам нужно просто сделать подписки.
```
postgres=# CREATE SUBSCRIPTION test_sub1
postgres-# CONNECTION 'host=10.129.0.16 port=5432 user=repl password=P@ssw0rd dbname=postgres'
postgres-# PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub1" on publisher
CREATE SUBSCRIPTION
postgres=# CREATE SUBSCRIPTION test_sub2
postgres-# CONNECTION 'host=10.129.0.23 port=5432 user=repl password=P@ssw0rd dbname=postgres'
postgres-# PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub2" on publisher
CREATE SUBSCRIPTION
```

Проверим что дошли данные добавленные ранее
```
postgres=# select * from test1;
   c
-------
 hello
(1 row)

postgres=# select * from test2;
   c
-------
 world
(1 row)
```
Все работает.
## **Настойка для ВМ4**
Тут у нас будет физическая репликация

Останавливаем кластер, удаляем файлы
```
snekrasov@vm4:~$ sudo pg_ctlcluster 14 main stop
snekrasov@vm4:~$ sudo rm -rf /var/lib/postgresql/14/main
```
Запуск pg_basebackup

```
snekrasov@vm4:~$ sudo -u postgres pg_basebackup -p 5432 -h 10.129.0.29 -U repl -R -D /var/lib/postgresql/14/main
```
Запуск кластера
```
sudo pg_ctlcluster 14 main start
```
Проверим данные
```
postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test1 | table | postgres
 public | test2 | table | postgres
(2 rows)

postgres=# select * from test1;
   c
-------
 hello
(1 row)

postgres=# select * from test2;
   c
-------
 world
(1 row)
```
Все работает.
