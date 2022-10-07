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
```
## **Проверка haproxy**

Веб доступен http://{dbproxy}:7000