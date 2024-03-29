# **Итоговая работа**

## **Проверка etcd кластера**

```
root@dbproxy:/home/snekrasov# etcdctl cluster-health
member 3f6ab05dcdcf44e7 is healthy: got healthy result from http://10.129.0.18:2379
member 8b1c8834246bf481 is healthy: got healthy result from http://10.129.0.25:2379
cluster is healthy
```

```
root@dbproxy:/home/snekrasov# etcdctl member list
3f6ab05dcdcf44e7: name=dbproxy peerURLs=http://10.129.0.18:2380 clientURLs=http://10.129.0.18:2379 isLeader=true
8b1c8834246bf481: name=db1 peerURLs=http://10.129.0.25:2380 clientURLs=http://10.129.0.25:2379 isLeader=false
```

```
root@dbproxy:/home/snekrasov# etcdctl set foo bar
bar
root@dbproxy:/home/snekrasov# etcdctl get foo bar
bar
root@dbproxy:/home/snekrasov# etcdctl ls
/foo
```

## **Проверка patroni кластера**

```
root@db2:/home/snekrasov# patronictl -c /etc/patroni/config.yml  list
+ Cluster: postgres (7151059579343117417) -+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| db1    | 10.129.0.25 | Leader  | running |  1 |           |
| db2    | 10.129.0.6  | Replica | running |  1 |         0 |
+--------+-------------+---------+---------+----+-----------+

root@db1:/home/snekrasov# patronictl -c /etc/patroni/config.yml reload postgres
+ Cluster: postgres (7151059579343117417) -+----+-----------+
| Member | Host        | Role    | State   | TL | Lag in MB |
+--------+-------------+---------+---------+----+-----------+
| db1    | 10.129.0.25 | Replica | running | 12 |         0 |
| db2    | 10.129.0.6  | Leader  | running | 12 |           |
+--------+-------------+---------+---------+----+-----------+
Are you sure you want to reload members db2, db1? [y/N]: y
Reload request received for member db2 and will be processed within 10 seconds

```
На случай ошибки
```
patronictl -c /etc/patroni/config.yml  reinit postgres
```

## **Проверка haproxy**

Веб доступен http://{dbproxy}:7000

## **Проверка PgBouncer**

```postgresql
psql -h localhost -p 6432 -U postgres -W -d pgbouncer
```

```postgresql
pgbouncer=#
SHOW CLIENTS;
type |   user   | database  | state  |   addr    | port  | local_addr | local_port |      connect_time       |      request_time       | wait | wait_us | close_needed |      ptr       | link | remote_pid | tls 
------+----------+-----------+--------+-----------+-------+------------+------------+-------------------------+-------------------------+------+---------+--------------+----------------+------+------------+-----
 C    | postgres | pgbouncer | active | 127.0.0.1 | 41664 | 127.0.0.1  |       6432 | 2022-10-08 14:00:37 UTC | 2022-10-08 14:01:30 UTC |   48 |  217284 |            0 | 0x556bba41f440 |      |          0 | 
(1 row)

pgbouncer=#
SHOW POOLS;
database  |   user    | cl_active | cl_waiting | cl_cancel_req | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us | pool_mode 
-----------+-----------+-----------+------------+---------------+-----------+---------+---------+-----------+----------+---------+------------+-----------
 pgbouncer | pgbouncer |         1 |          0 |             0 |         0 |       0 |       0 |         0 |        0 |       0 |          0 | statement
(1 row)

```
Посмотреть хэши паролей
```
SELECT pg_shadow.usename,  passwd FROM pg_shadow;
```

## **Api для тестов**

- /api/book/{id}
- /api/book/{id}/items
- /api/book/{id} POST
- /api/returnBook POST
- /api/book?size={s}&page={p}
- /api/author
- /api/author/{id}

## **Запуск танка**
```
docker run --rm -v /var/yandex.tank:/var/loadtest -it direvius/yandex-tank
```
## **Проверка postgres exporter**
http://{dbip}:9187

## **Проверка Prometheus**
http://{dbproxy}:9090/targets

## **Grafana**
http://{dbproxy}:3000

## **Презентация**
https://docs.google.com/presentation/d/11LtWWsWzv6LFDjcztFjEUOieDKW4OxSQ2W0sN9Ofp8g/edit#slide=id.g5dfdcf0df5_0_16
![](postgres.png)