# **Введение**

Домашняя работа "SQL и реляционные СУБД. Введение в PostgreSQL"

# **Подготовка**

Создадим базу данных, таблицу, заполним данными.

```
postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus
Пароль:
Вы подключены к базе данных "otus" как пользователь "postgres".
otus=# \set AUTOCOMMIT OFF
otus=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
otus=# insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
INSERT 0 1
INSERT 0 1
COMMIT
```
# **Выполнение**

Проверим наличие данных

```
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строки)
```

Посмотреть текущий уровень изоляции

```
otus=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 строка)
```
Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

```
otus=# begin;
BEGIN
```
В первой сессии добавить новую запись
```
otus=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```

Завершить первую транзакцию - commit;
```
otus=# commit;
COMMIT
```

Сделать select * from persons во второй сессии
```
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
```
Есть новая строка, т.к. проявляется аномалия non-repeatable read
Завершите транзакцию во второй сессии
```
otus=# commit;
COMMIT
```
Начать новые но уже repeatable read транзакции - set transaction isolation level repeatable read;

```
otus=# set transaction isolation level repeatable read;
SET
otus=# begin;
ПРЕДУПРЕЖДЕНИЕ:  транзакция уже выполняется
BEGIN
```
В первой сессии добавить новую запись
```
otus=# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
Сделать select * from persons во второй сессии

```
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
```
Новой записи нет - первая транзакция не закомичена. 

Завершить первую транзакцию - commit;

```
otus=# commit;
COMMIT
```
Сделать select * from persons во второй сессии
```
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строки)
```
Новой записи нет - хотя первая транзакция закомичена, уровень изоляции видит слепок на начало второй транзакции.


Завершить вторую транзакцию
```
otus=# commit;
COMMIT
```
Сделать select * from persons во второй сессии

```
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 строки)
```
Все транзакции заверщены мы видим актуальное состояние БД.







