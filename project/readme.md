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