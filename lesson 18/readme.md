# **Введение**

Домашнее задание на тему "Сбор и использование статистики".

# **Выполнение**

Для выполнения будем использовать базу flights;

## **Создать индексы на БД, которые ускорят доступ к данным.**
### **Создать индекс к какой-либо из таблиц вашей БД. Прислать текстом результат команды explain.**

Попробуем найти все рейсы в статусе Arrived.
```
explain
select * from bookings.flights f where status = 'Arrived';
```
```
Seq Scan on flights f  (cost=0.00..806.01 rows=16681 width=63)
  Filter: ((status)::text = 'Arrived'::text)                  
```
добавим индекс

```
create index idx_flights_status on bookings.flights(status);
```
```
Index Scan using idx_flights_status on flights f  (cost=0.29..689.53 rows=16681 width=63)|
  Index Cond: ((status)::text = 'Arrived'::text)                                         |
```
Существенное снижение стоимости

### **Реализовать индекс для полнотекстового поиска**

Найдем все ивановых.

```
explain
select * from bookings.tickets t where passenger_name ilike '%ivanov%'

Seq Scan on tickets t  (cost=0.00..10728.16 rows=22855 width=104)
  Filter: (passenger_name ~~* '%ivanov%'::text)                  
```

Добавим поле

```
alter table bookings.tickets add column fio tsvector;
update bookings.tickets
set fio = to_tsvector(passenger_name);
```

```
explain
select passenger_name
from bookings.tickets
where fio @@ to_tsquery('ivanov');

Gather  (cost=1000.00..55768.52 rows=7041 width=16)                         
  Workers Planned: 2                                                        
  ->  Parallel Seq Scan on tickets  (cost=0.00..54064.42 rows=2934 width=16)
        Filter: (fio @@ to_tsquery('ivanov'::text))                         
```
Добавим индекс и повторим
```
CREATE INDEX search_index_ord ON bookings.tickets USING GIN (fio);

explain
select passenger_name
from bookings.tickets
where fio @@ to_tsquery('ivanov');

Bitmap Heap Scan on tickets  (cost=60.32..7736.92 rows=7041 width=16)            
  Recheck Cond: (fio @@ to_tsquery('ivanov'::text))                              
  ->  Bitmap Index Scan on search_index_ord  (cost=0.00..58.56 rows=7041 width=0)
        Index Cond: (fio @@ to_tsquery('ivanov'::text))                          
```
Существенное снижение стоимости

### **Реализовать индекс на поле с функцией**

Найдем все аэропорты прибытия pkc.
```
explain
select * from bookings.flights f where lower(arrival_airport) = 'pkc';

Seq Scan on flights f  (cost=0.00..971.62 rows=166 width=63)
  Filter: (lower((arrival_airport)::text) = 'pkc'::text)    
```
Создадим индекс с функцией
```
create index idx_flights_arrival_airport on bookings.flights(lower(arrival_airport));

explain
select * from bookings.flights f where lower(arrival_airport) = 'pkc';
Bitmap Heap Scan on flights f  (cost=2.68..148.18 rows=166 width=63)                      
  Recheck Cond: (lower((arrival_airport)::text) = 'pkc'::text)                            
  ->  Bitmap Index Scan on idx_flights_arrival_airport  (cost=0.00..2.63 rows=166 width=0)
        Index Cond: (lower((arrival_airport)::text) = 'pkc'::text)                        
```
Существенное снижение стоимости

### **Создать индекс на несколько полей**

Найдем рейсы с отправлением из DME и посадкой в LED.

```
explain
select * from bookings.flights f where departure_airport ='DME' and arrival_airport  = 'LED';

Seq Scan on flights f  (cost=0.00..888.82 rows=184 width=63)                         
  Filter: ((departure_airport = 'DME'::bpchar) AND (arrival_airport = 'LED'::bpchar))
```

Создадим состовной индекс.

```
create index idx_flights_departure_airport_arrival_airport on bookings.flights(departure_airport, arrival_airport);

explain
select * from bookings.flights f where departure_airport ='DME' and arrival_airport  = 'LED';

Bitmap Heap Scan on flights f  (cost=3.28..161.76 rows=184 width=63)                                        
  Recheck Cond: ((departure_airport = 'DME'::bpchar) AND (arrival_airport = 'LED'::bpchar))                 
  ->  Bitmap Index Scan on idx_flights_departure_airport_arrival_airport  (cost=0.00..3.23 rows=184 width=0)
        Index Cond: ((departure_airport = 'DME'::bpchar) AND (arrival_airport = 'LED'::bpchar))             
```
### **Вывод**

Без индекса запрос выполняется с использованием Sec scan, а с индексом разновидности Index Scan что существенно быстрей.

## **Написания запросов с различными типами соединений.**

### **Реализовать прямое соединение двух или более таблиц**

Выберем все места для всех самолетов.
```
select a.aircraft_code, model, seat_no from bookings.aircrafts a
inner join bookings.seats s on a.aircraft_code = s.aircraft_code;
```

```
aircraft_code|model          |seat_no|
-------------+---------------+-------+
319          |Airbus A319-100|2A     |
319          |Airbus A319-100|2C     |
319          |Airbus A319-100|2D     |
319          |Airbus A319-100|2F     |
319          |Airbus A319-100|3A     |
319          |Airbus A319-100|3C     |
319          |Airbus A319-100|3D     |
319          |Airbus A319-100|3F     |
319          |Airbus A319-100|4A     |
319          |Airbus A319-100|4C     |
319          |Airbus A319-100|4D     |
319          |Airbus A319-100|4F     |
319          |Airbus A319-100|5A     |
319          |Airbus A319-100|5C     |
319          |Airbus A319-100|5D     |
319          |Airbus A319-100|5F     |
319          |Airbus A319-100|6A     |
319          |Airbus A319-100|6B     |
319          |Airbus A319-100|6C     |
...
```
### **Реализовать левостороннее (или правостороннее) соединение двух или более таблиц**

Выберем все рейсы которые отправляются из аэропортов.

```
select airport_code, airport_name, flight_no  from bookings.airports a 
left join bookings.flights f on a.airport_code = f.departure_airport  
```
```
airport_code|airport_name|flight_no|
------------+------------+---------+
DME         |Домодедово  |PG0405   |
DME         |Домодедово  |PG0404   |
DME         |Домодедово  |PG0405   |
DME         |Домодедово  |PG0402   |
DME         |Домодедово  |PG0405   |
DME         |Домодедово  |PG0404   |
DME         |Домодедово  |PG0403   |
DME         |Домодедово  |PG0402   |
DME         |Домодедово  |PG0405   |
DME         |Домодедово  |PG0402   |
DME         |Домодедово  |PG0403   |
DME         |Домодедово  |PG0404   |
DME         |Домодедово  |PG0405   |
DME         |Домодедово  |PG0402   |
DME         |Домодедово  |PG0402   |
DME         |Домодедово  |PG0403   |
DME         |Домодедово  |PG0404   |
DME         |Домодедово  |PG0405   |
DME         |Домодедово  |PG0403   |
...
```
### **Реализовать кросс соединение двух или более таблиц**

Найдем все комбинации аэропортов вылет/посадка.

```
select a1.airport_code, a2.airport_code from bookings.airports a1
cross join bookings.airports a2 
where a1 <> a2
```
```
airport_code|airport_code|
------------+------------+
MJZ         |NBC         |
MJZ         |NOZ         |
MJZ         |NAL         |
MJZ         |OGZ         |
MJZ         |CSY         |
MJZ         |NYM         |
MJZ         |NYA         |
MJZ         |URS         |
MJZ         |SKX         |
MJZ         |TBW         |
MJZ         |OVS         |
MJZ         |IJK         |
MJZ         |SLY         |
MJZ         |HMA         |
MJZ         |RGK         |
MJZ         |UKX         |
MJZ         |GDZ         |
MJZ         |NFG         |
MJZ         |VKT         |
MJZ         |KJA         |
...
```

### **Реализовать полное соединение двух или более таблиц**

Найдем самолеты без рейсов или рейсы без самолетов.
```
select a.aircraft_code, model, flight_no from bookings.aircrafts a
full join bookings.flights f on a.aircraft_code = f.aircraft_code
where flight_no is null or a.aircraft_code is null
```
```
aircraft_code|model          |flight_no|
-------------+---------------+---------+
320          |Airbus A320-200|         |
```






