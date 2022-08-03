# **Введение**

Домашнее задание на тему блокировки.



# **Подготовка**
Создадим таблицу с данными
```
CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
```

# **Выполнение**

### **Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.**


Установим параметры логирования.
```
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM set log_min_duration_statement = 200;
SELECT pg_reload_conf();
```
Запустим запрос в двух сессиях.
```
begin;
select * from accounts a for update;
```
В первой он выполнится а во второй будет ожидать.
Смотрим в журнал.
```
2022-08-02 17:12:44.193 +05 [17025] postgres@otus СООБЩЕНИЕ:  процесс 17025 продолжает ожидать в режиме ShareLock блокировку "транзакция 33246" в течение 1000.185 мс
2022-08-02 17:12:44.193 +05 [17025] postgres@otus ПОДРОБНОСТИ:  Process holding the lock: 17026. Wait queue: 17025.
2022-08-02 17:12:44.193 +05 [17025] postgres@otus КОНТЕКСТ:  при блокировке кортежа (0,1) в отношении "accounts"
2022-08-02 17:12:44.193 +05 [17025] postgres@otus ОПЕРАТОР:  select * from accounts a for update
2022-08-02 17:12:55.221 +05 [17025] postgres@otus СООБЩЕНИЕ:  процесс 17025 получил в режиме ShareLock блокировку "транзакция 33246" через 12027.667 мс
2022-08-02 17:12:55.221 +05 [17025] postgres@otus КОНТЕКСТ:  при блокировке кортежа (0,1) в отношении "accounts"
2022-08-02 17:12:55.221 +05 [17025] postgres@otus ОПЕРАТОР:  select * from accounts a for update
2022-08-02 17:12:55.221 +05 [17025] postgres@otus СООБЩЕНИЕ:  продолжительность: 12027.840 мс  выполнение <unnamed>: select * from accounts a for update
```
Видно что процесс с pid 17025 ожидал блокировки примерно 10с, а потом получил её.

### **Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.Пришлите список блокировок и объясните, что значит каждая.**

Выполним запрос в трех сессиях.

```
begin;
update accounts set amount = 10 where acc_no =1;
```
Pid: 18292, 15632, 17948

```
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid in (18292, 15632, 17948)
locktype     |relation     |virtxid|xid    |mode            |granted|
-------------+-------------+-------+-------+----------------+-------+
relation     |accounts_pkey|       |       |RowExclusiveLock|true   |
relation     |accounts     |       |       |RowExclusiveLock|true   |
virtualxid   |             |7/63   |       |ExclusiveLock   |true   |
relation     |accounts_pkey|       |       |RowExclusiveLock|true   |
relation     |accounts     |       |       |RowExclusiveLock|true   |
virtualxid   |             |8/56   |       |ExclusiveLock   |true   |
relation     |accounts_pkey|       |       |RowExclusiveLock|true   |
relation     |accounts     |       |       |RowExclusiveLock|true   |
virtualxid   |             |9/68   |       |ExclusiveLock   |true   |
tuple        |accounts     |       |       |ExclusiveLock   |true   |
transactionid|             |       |3282387|ExclusiveLock   |true   |
transactionid|             |       |3282388|ExclusiveLock   |true   |
transactionid|             |       |3282386|ShareLock       |false  |
tuple        |accounts     |       |       |ExclusiveLock   |false  |
transactionid|             |       |3282386|ExclusiveLock   |true   |
```

```
SELECT relation, locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for FROM pg_locks 

WHERE   relation = 'accounts'::regclass
relation|locktype|mode            |granted|pid  |wait_for|
--------+--------+----------------+-------+-----+--------+
 7652036|relation|RowExclusiveLock|true   |18292|{}      |
 7652036|relation|RowExclusiveLock|true   |15632|{18292} |
 7652036|relation|RowExclusiveLock|true   |17948|{15632} |
 7652036|tuple   |ExclusiveLock   |true   |15632|{18292} |
 7652036|tuple   |ExclusiveLock   |false  |17948|{15632} |
```
2 и 3 сессии зависают.

Видим эксклюзивные блокировки номера транзакций virtualxid и transactionid.
Построчная блокировка на первичный ключ accounts_pkey.
Блокировка версии строки tuple т.к. нужно установить очередность для более чем двух транзакций.

### **Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?**

Выполним следующее в трех разных сессиях.

№1
```
BEGIN;
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 1;
```
№2
```
BEGIN;
UPDATE accounts SET amount = amount + 20.00 WHERE acc_no = 2;
UPDATE accounts SET amount = amount + 20.00 WHERE acc_no = 1;
```
№3
```
BEGIN;
UPDATE accounts SET amount = amount - 30.00 WHERE acc_no = 3;
UPDATE accounts SET amount = amount - 30.00 WHERE acc_no = 2;
```
И В первой.
```
UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 3;
```
Получаем ошибку на клиенте.
```
SQL Error [40P01]: ERROR: deadlock detected
  Подробности: Process 4592 waits for ShareLock on transaction 3282399; blocked by process 7800.
Process 7800 waits for ShareLock on transaction 3282398; blocked by process 20168.
Process 20168 waits for ShareLock on transaction 3282397; blocked by process 4592.
```
В логах
```
2022-08-03 11:49:17.687 GMT [20168] LOG:  process 20168 still waiting for ShareLock on transaction 3282397 after 1006.658 ms
2022-08-03 11:49:17.687 GMT [20168] DETAIL:  Process holding the lock: 4592. Wait queue: 20168.
2022-08-03 11:49:17.687 GMT [20168] CONTEXT:  while updating tuple (0,13) in relation "accounts"
2022-08-03 11:49:17.687 GMT [20168] STATEMENT:  

	

	UPDATE accounts SET amount = amount + 20.00 WHERE acc_no = 1
2022-08-03 11:49:21.699 GMT [7800] LOG:  process 7800 still waiting for ShareLock on transaction 3282398 after 1004.629 ms
2022-08-03 11:49:21.699 GMT [7800] DETAIL:  Process holding the lock: 20168. Wait queue: 7800.
2022-08-03 11:49:21.699 GMT [7800] CONTEXT:  while updating tuple (0,2) in relation "accounts"
2022-08-03 11:49:21.699 GMT [7800] STATEMENT:  

	

	UPDATE accounts SET amount = amount - 30.00 WHERE acc_no = 2
2022-08-03 11:49:26.703 GMT [4592] LOG:  process 4592 detected deadlock while waiting for ShareLock on transaction 3282399 after 1008.324 ms
2022-08-03 11:49:26.703 GMT [4592] DETAIL:  Process holding the lock: 7800. Wait queue: .
2022-08-03 11:49:26.703 GMT [4592] CONTEXT:  while updating tuple (0,3) in relation "accounts"
2022-08-03 11:49:26.703 GMT [4592] STATEMENT:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 3
2022-08-03 11:49:26.703 GMT [4592] ERROR:  deadlock detected
2022-08-03 11:49:26.703 GMT [4592] DETAIL:  Process 4592 waits for ShareLock on transaction 3282399; blocked by process 7800.
	Process 7800 waits for ShareLock on transaction 3282398; blocked by process 20168.
	Process 20168 waits for ShareLock on transaction 3282397; blocked by process 4592.
	Process 4592: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 3
```
Вот что можно прочитать
pid 20168, пытается выполнить UPDATE accounts SET amount = amount + 20.00 WHERE acc_no = 1, но вынужден ждать
pid 7800, пытается выполнить UPDATE accounts SET amount = amount - 30.00 WHERE acc_no = 2, но вынужден ждать
pid 4592, замыкает цепь и вызывает deadlock, UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 3
Начало цепочки это pid 4592, а вот какой запрос был не понятно.

PS. На рабочем кластере полагаю пришлось бы покапатся в лога подольше.

### **Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга? Попробуйте воспроизвести такую ситуацию.**
Для этого нуж блокировать строки в разном порядке

№1
```
BEGIN;
declare c cursor for
select amount, acc_no from accounts a
order by acc_no asc for update;
```

№2
```
BEGIN;
declare c cursor for
select amount, acc_no from accounts a
order by acc_no desc for update;
```

№1
```
fetch;
```
№2
```
fetch;
```
№1
```
fetch;
```
№2
```
fetch;
```
№1
```
fetch;
ERROR: deadlock detected
  Подробности: Process 20168 waits for ShareLock on transaction 3282402; blocked by process 7800.
Process 7800 waits for ShareLock on transaction 3282401; blocked by process 20168.
```



