# **Введение**

Домашнее задание на тему "Секционирование".

# **Выполнение**

Для выполнения будем использовать базу flights;

В начале попробуем сделать таблицу flights секционированной по статусу.
```
create table bookings.flights_part
(
    flight_id           integer default nextval('bookings.flights_flight_id_seq'::regclass) not null,
    flight_no           char(6)                                                             not null,
    scheduled_departure timestamp with time zone                                            not null,
    scheduled_arrival   timestamp with time zone                                            not null,
    departure_airport   char(3)                                                             not null
        references bookings.airports,
    arrival_airport     char(3)                                                             not null
        references bookings.airports,
    status              varchar(20)                                                         not null
        constraint flights_status_check
            check ((status)::text = ANY
                   (ARRAY [('On Time'::character varying)::text, ('Delayed'::character varying)::text, ('Departed'::character varying)::text, ('Arrived'::character varying)::text, ('Scheduled'::character varying)::text, ('Cancelled'::character varying)::text])),
    aircraft_code       char(3)                                                             not null
        references bookings.aircrafts,
    actual_departure    timestamp with time zone,
    actual_arrival      timestamp with time zone
) PARTITION BY LIST (status);

```
Создаем партиции
```
CREATE TABLE bookings.flights_part_departed PARTITION OF bookings.flights_part
    for values in ('Departed');
CREATE TABLE bookings.flights_part_arrived PARTITION OF bookings.flights_part
    for values in ('Arrived');
CREATE TABLE bookings.flights_on_time PARTITION OF bookings.flights_part
    for values in ('On Time');
CREATE TABLE bookings.flights_cancelled PARTITION OF bookings.flights_part
    for values in ('Cancelled');
CREATE TABLE bookings.flights_delayed PARTITION OF bookings.flights_part
    for values in ('Delayed');
CREATE TABLE bookings.flights_scheduled PARTITION OF bookings.flights_part
    for values in ('Scheduled');
```
Проверим состояние таблиц.
```
SELECT tableoid::regclass, count(*) FROM bookings.flights_part GROUP BY tableoid;
```
| tableoid | count |
| :--- | :--- |
| bookings.flights\_delayed | 41 |
| bookings.flights\_scheduled | 15383 |
| bookings.flights\_part\_arrived | 16707 |
| bookings.flights\_part\_departed | 58 |
| bookings.flights\_on\_time | 518 |
| bookings.flights\_cancelled | 414 |

Данные распределены.
```
explain select * from bookings.flights_part where status = 'Delayed'
```
| QUERY PLAN |
| :--- |
| Seq Scan on flights\_delayed flights\_part  \(cost=0.00..15.12 rows=2 width=170\) |
|   Filter: \(\(status\)::text = 'Delayed'::text\) |

Видно что планировщик проверяет только однц таблицу.

Попробуем использовать like
```
explain select * from bookings.flights_part where status like 'Delaye%'
```
| QUERY PLAN |
| :--- |
| Append  \(cost=0.00..822.66 rows=7 width=93\) |
|   -&gt;  Seq Scan on flights\_part\_arrived flights\_part\_1  \(cost=0.00..415.84 rows=1 width=63\) |
|         Filter: \(\(status\)::text \~\~ 'Delaye%'::text\) |
|   -&gt;  Seq Scan on flights\_cancelled flights\_part\_2  \(cost=0.00..10.18 rows=1 width=65\) |
|         Filter: \(\(status\)::text \~\~ 'Delaye%'::text\) |
|   -&gt;  Seq Scan on flights\_delayed flights\_part\_3  \(cost=0.00..15.12 rows=2 width=170\) |
|         Filter: \(\(status\)::text \~\~ 'Delaye%'::text\) |
|   -&gt;  Seq Scan on flights\_part\_departed flights\_part\_4  \(cost=0.00..1.73 rows=1 width=64\) |
|         Filter: \(\(status\)::text \~\~ 'Delaye%'::text\) |
|   -&gt;  Seq Scan on flights\_on\_time flights\_part\_5  \(cost=0.00..12.48 rows=1 width=63\) |
|         Filter: \(\(status\)::text \~\~ 'Delaye%'::text\) |
|   -&gt;  Seq Scan on flights\_scheduled flights\_part\_6  \(cost=0.00..367.29 rows=1 width=65\) |
|         Filter: \(\(status\)::text \~\~ 'Delaye%'::text\) |

К сожалению идет обход всех партиций.

Проверим сортировку.
```
explain select * from bookings.flights_part order by status;
```
| QUERY PLAN |
| :--- |
| Sort  \(cost=3423.37..3507.09 rows=33490 width=65\) |
|   Sort Key: flights\_part.status |
|   -&gt;  Append  \(cost=0.00..906.35 rows=33490 width=65\) |
|         -&gt;  Seq Scan on flights\_part\_arrived flights\_part\_1  \(cost=0.00..374.07 rows=16707 width=63\) |
|         -&gt;  Seq Scan on flights\_cancelled flights\_part\_2  \(cost=0.00..9.14 rows=414 width=65\) |
|         -&gt;  Seq Scan on flights\_delayed flights\_part\_3  \(cost=0.00..14.10 rows=410 width=170\) |
|         -&gt;  Seq Scan on flights\_part\_departed flights\_part\_4  \(cost=0.00..1.58 rows=58 width=64\) |
|         -&gt;  Seq Scan on flights\_on\_time flights\_part\_5  \(cost=0.00..11.18 rows=518 width=63\) |
|         -&gt;  Seq Scan on flights\_scheduled flights\_part\_6  \(cost=0.00..328.83 rows=15383 width=65\) |

Тоже самое.

Добавим индексы.

```
CREATE INDEX flights_part_idx ON ONLY bookings.flights_part(status);

CREATE INDEX CONCURRENTLY flights_part_departed_idx ON bookings.flights_part_departed (status);
CREATE INDEX CONCURRENTLY flights_part_arrived_idx ON bookings.flights_part_arrived (status);
CREATE INDEX CONCURRENTLY flights_on_time_idx ON bookings.flights_on_time  (status);
CREATE INDEX CONCURRENTLY flights_cancelled_idx ON bookings.flights_cancelled (status);
CREATE INDEX CONCURRENTLY flights_delayed_idx ON bookings.flights_delayed (status);
CREATE INDEX CONCURRENTLY flights_scheduled_idx ON bookings.flights_scheduled  (status);

ALTER INDEX bookings.flights_part_idx ATTACH PARTITION bookings.flights_part_departed_idx;
ALTER INDEX bookings.flights_part_idx ATTACH PARTITION bookings.flights_part_arrived_idx;
ALTER INDEX bookings.flights_part_idx ATTACH PARTITION bookings.flights_on_time_idx;
ALTER INDEX bookings.flights_part_idx ATTACH PARTITION bookings.flights_delayed_idx;
ALTER INDEX bookings.flights_part_idx ATTACH PARTITION bookings.flights_cancelled_idx;
ALTER INDEX bookings.flights_part_idx ATTACH PARTITION bookings.flights_scheduled_idx;
```
Проверим индекс.
```
SELECT indisvalid FROM pg_index WHERE indexrelid::regclass::text = 'bookings.flights_part_idx';

```
| indisvalid | indexrelid |
| :--- | :--- |
| true | bookings.flights\_part\_idx |

Индекс создан.

Проверим сортировку.
```
explain select * from bookings.flights_part order by status;
```
| QUERY PLAN |
| :--- |
| Append  \(cost=1.15..1103.17 rows=33121 width=64\) |
|   -&gt;  Index Scan using flights\_part\_arrived\_idx on flights\_part\_arrived flights\_part\_1  \(cost=0.29..475.59 rows=16707 width=63\) |
|   -&gt;  Index Scan using flights\_cancelled\_idx on flights\_cancelled flights\_part\_2  \(cost=0.15..13.66 rows=414 width=65\) |
|   -&gt;  Index Scan using flights\_delayed\_idx on flights\_delayed flights\_part\_3  \(cost=0.14..4.06 rows=41 width=170\) |
|   -&gt;  Index Scan using flights\_part\_departed\_idx on flights\_part\_departed flights\_part\_4  \(cost=0.14..4.31 rows=58 width=64\) |
|   -&gt;  Index Scan using flights\_on\_time\_idx on flights\_on\_time flights\_part\_5  \(cost=0.15..16.22 rows=518 width=63\) |
|   -&gt;  Index Scan using flights\_scheduled\_idx on flights\_scheduled flights\_part\_6  \(cost=0.29..423.73 rows=15383 width=65\) |

Индекс работает.
